# [RFC][clang] Modules Build Daemon: Build System Agnostic Support for Explicitly Built Modules

## Abstract

Modules have the potential to significantly improve compile-time performance, as they eliminate the repetitive processing of header files that occur during textual inclusion builds. By processing a module once and reusing it across all associated translation units, modules offer a more efficient approach to managing dependencies. Clang currently supports two methods for building modules: implicit and explicit. With the expicit method the build system has complete knowladge of the dependency graph before building begins while with the implicit method the build system discovers dependencies as modules are built. While the explicit method boosts speed, they demand considerable development effort from build systems. The implicit method, on the other hand, integrate seemlesly with existing workflows but is inefficiency. "Modules Build Daemon: Build System Agnostic Support for Explicitly Built Modules" aims to balance these two approaches, enabling developers to reap the benefits of explicit modules irrespective of their build system. This project aims to implement a daemon that serves as a module manager. By simply incorporating a single command line flag, each Clang invocation registers its translation unit with the daemon, which then scans the unit's dependencies. As translation units are registered and analyzed, the daemon constructs a dependency graph for the entire project. Concurrently, it uses the emerging graph to schedule and compile each module. This approach allows for a single entity to effectively coordinate the build of modules.

## Scope

For the purpose of Google Summer of Code, development will be focused on providing support for Unix-like systems. I would like to keep Windows and other operating systems in mind so that nothing needs to be rearchitected when support for other operating systems are implemented down the road.

## Clang Driver

### Option Parsing

The clang driver will recognize `-fmodule-build-daemon` as a valid command line option

```bash
# example
$ clang++ -fmodule-build-daemon foo.cpp -o foo
```

```cpp
// llvm-project/clang/include/clang/Driver/Options.td
def fmodule-build-daemon : Flag<["-"], "fmodule-build-daemon">, 
	Group<f_Group>, Flags<[NoXarchOption]>, 
	HelpText<"Enable module build daemon functionality">;
```

### Tool Specific Argument Translation

When `-fmodule-build-daemon` is passed to the clang driver, the driver will include `-cc1modbuildd` (cc1 module build daemon) as the first flag with each clang invocation.

```console
$ clang++ -### -fmodule-build-daemon foo.cpp bar.cpp -o test

"clang-17" "-cc1modbuildd" "-o" "/tmp/foo-66a77d.o" "-x" "c++" "foo.cpp"
"clang-17" "-cc1modbuildd" "-o" "/tmp/bar-73584c.o" "-x" "c++" "bar.cpp"
"lld" ... "-o" "test" ... "/tmp/foo-66a77d.o" "/tmp/bar-73584c.o" ...
```

```cpp
// llvm-project/clang/lib/Driver/ToolChains/Clang.cpp

void Clang::ConstructJob(Compilation &C, const JobAction &Job, ...) {
    
    // handle -fmodule-build-daemon
    if (Job.getKind() == Action::ModBuildJobClass) {
        CmdArgs.push_back("-cc1modbuildd");

        // pass -fmodule-build-daemon related options to cc1modbuildd.
        Args.AddAllArgs(CmdArgs, ModBuildOpts);
    }
}
```

### Integration

If the first flag is `-cc1modbuildd` then `cc1modbuildd_main` will be called instead of `cc1_main`. Creating a separate entry point for the module build daemon mode will make it easier to encapsulate the behavior of the build daemon and prevent the core of the compiler from turning into a full-fledged build system.

```cpp
// llvm-project/clang/tools/driver/driver.cpp

int clang_main(int Argc, char **Argv, const llvm::ToolContext &ToolContext) {
    if (Args.size() >= 2 && StringRef(Args[1]).startswith("-cc1"))
        return ExecuteCC1Tool();
}

static int ExecuteCC1Tool(SmallVectorImpl<const char *> &ArgV, const llvm::ToolContext &ToolContext) {
	
    StringRef Tool = ArgV[1];

    if (Tool == "-cc1")
        return cc1_main(ArrayRef(ArgV).slice(1), ArgV[0], GetExecutablePathVP);

    if (Tool == "-cc1modbuildd")
        return cc1modbuildd_main(ArrayRef(ArgV).slice(1), ArgV[0], GetExecutablePathVP);
}
```

## Daemon

The daemon will use Unix sockets as its form of IPC. The Windows 10 April 2018 update included support for Unix sockets, making this IPC portable.

### Overview

Requirements
- The number of active threads managed by the daemon should be equal to the number of registered clang invocations to comply with the `-j` limit
- An activated thread will first complete a dependency scan of the registered clang invocation and then begin building dependencies
- The dependencies built by an active thread do not necessarily have to be required by the translation unit that activated the thread

```cpp
// pseudo-code of cc1modbuildd_main.cpp

void scanDependencies(Client client, ThreadSafeGraph<Dependency>& depsGraph) {
    // code for scanning dependencies
    // dependencies are put into depsGraph.
}

void buildDependencies(Client client, ThreadSafeGraph<Dependency>& depsGraph) {
    // code for building dependencies
    // dependencies are fetched from depsGraph
    // runs until a client disconnects
    //     - when a thread builds the last dependency for a TU, the clang invocation will disconnect,
    //       and the daemon will tell whichever thread completed the build to shutdown 
}

void handleConnection(Client client, ThreadSafeGraph<Dependency>& depsGraph, llvm::ThreadPool& Pool) {
    scanDependencies(client, depsGraph);
    Pool.async(buildDependencies, depsGraph);
}

void BuildServer::listen() {

    // shared dependency graph
    ThreadSafeGraph<Dependency> depsGraph;
    llvm::ThreadPool Pool;
    
    while (true) {
        Client client = listenForClangInvocation();
    
        if (client != NULL) {
            // If a new client has connected, allocate a thread for handling the client.
            Pool.async(handleConnection, client, depsGraph, Pool);
        }
    }
}

void initializeDaemon()
{
    BuildServer.start();
    BuildServer.listen();
}

int cc1modbuildd_main() {

    if(daemon != exists) {
        initializeDaemon(); 
    } else {
        connectToDaemon();
    }
    compileTranslationUnit();
    return 0;
}
```

### Build Session

By managing build sessions, cache validation becomes much more reliable and efficient. The daemon will validate the cache at the beginning of a build session as clang requires sources. Once the daemon validates a source's cache, the daemon only needs to check that any subsequent clang invocations come from the same build session to reuse the cache.

How to define build sessions?

There are two main ways to compile a software project. A build system like Ninja or Make is the most popular way. The secondary method is to write commands manually and run them one by one via the terminal or with a shell script. For projects that use build systems, the daemon will define a build session by the start time of each clang invocation's parent process. In the `Ninja build system` example below if the parent process (PID 7482) had been running for `4:58 (298 seconds)` and the current time is `1685204282`, then the process would have a start time and build session ID of `1685203984`. It would be nice to define a build session by the parent process' PID, but there is a slight chance that two consecutive build system invocations use the same PID.

```
# Ninja build system

PPID    PID  COMMAND
2191   2191  /lib/systemd/systemd --user
2191   2871  \_ /usr/libexec/gnome-terminal-server
2871   2879   |   \_ bash
2879   7483   |   |   \_ ninja -j2
7483   7818   |   |       \_ /bin/sh -c /opt/llvm-project/bin/clang++  -o AsmMatcherEmitter.cpp.o -c AsmMatcherEmitter.cpp
7818   7819   |   |       |   \_ /opt/llvm-project/bin/clang++ -o AsmMatcherEmitter.cpp.o -c AsmMatcherEmitter.cpp
7483   7820   |   |       \_ /bin/sh -c /opt/llvm-project/bin/clang++  -o GIMatchTree.cpp.o -c GIMatchTree.cpp
7820   7821   |   |       |   \_ /opt/llvm-project/bin/clang++ -o GIMatchTree.cpp.o -c GIMatchTree.cpp
```

There is no great way to define a build session for projects that use shell scripts or manually run commands. The first iteration of the build daemon will not support build sessions for shell scripts and manually run commands. The daemon will treat each clang invocation as its own build session.

```
# Shell script

PPID    PID  COMMAND
2191   2191  /lib/systemd/systemd --user
2191  35721  \_ /opt/llvm-project/bin/clang++ -o AsmMatcherEmitter.cpp.o -c AsmMatcherEmitter.cpp
2191  35722  \_ /opt/llvm-project/bin/clang++ bar.cpp -o GIMatchTree.cpp.o -c GIMatchTree.cpp
```


### Cache Validation

The daemon needs a way to check if the cache saved from the last build session is still valid for the current build session. There are two common approaches: compare timestamps or compare hashes. Timestamps can be unreliable, especially with remote or virtual filesystems, so the build system will use hashes to check for changes. When saving the built dependency, the daemon will include the hash as part of its file name so that it does not waste disk space mapping file names to hashes.


### Cache Management

The cache will consist of compiled dependencies and the dependency graph from previous builds.

The build daemon will cache precompiled modules as Clang AST files. These Clang AST files encode the AST and associated data structures in a compressed bitstream format. Initially, the daemon will store cached dependencies exclusively on disk. Most modern systems can hold all of the built dependencies on disk. However, the build module will include a cache management mechanism to ensure support for systems with limited resources. The cache management mechanism will determine which built dependencies to retain based on the frequency of use, time-to-live, and the build sessions' dependency graph. Once a module is built or accessed, the cache management mechanism recalculates its TTL.

```cpp
// scenario one: module is compiled and added to the cache
initial_ttl = depth_in_DAG * SCALING_FACTOR_1;

// scenario two: precompiled module is used
new_ttl = curent_ttl + (depth_in_DAG * SCALING_FACTOR_2);
```
Once a module's ttl has expired, it will be flagged as a candidate for deletion.


### Scanning

The daemon must validate a translation unit's cached dependency graph with every build session. To validate the dependency graph, the daemon can either rescan the translation unit or check the consistency of each file and only rescan those files that have changed. For the first iteration of the build daemon, the plan is to rescan each translation unit. Still, down the road, I will implement an optimized approach of only rescanning modified. The daemon will use the `DependencyScanning` utilities provided under `llvm-project/clang/lib/Tooling` originally developed for `clang-scan-deps` to complete the scan.

```cpp
// llvm-project/clang/tools/driver/cc1modbuildd_main.cpp

scanDependencies(Client client, ThreadSafeGraph<Dependency>& depsGraph() {

    TranslationUnitDeps TUDeps;
    TUDeps = DependencyScanningTool::getTranslationUnitDependencies();    
    handleTranslationUnitResults();
}
```
`handleTranslationUnitResults()` will merge `TranslationUnitDeps` into `ThreadSafeGraph<Dependency>`.


### Scheduling

The daemon will schedule dependencies based on a topological sort, prioritizing modules that are required by more translation units. Meaning, if two dependencies are of the same priority in the topological sort, but one dependency is required by two translation units while the other is required by five translation units, the daemon will first schedule the dependency required by five translation units. The topological order of the dependency graph, `ThreadSafeGraph<Dependency>`, will be stored in a linked list. A linked list allows the daemon to insert new dependencies at arbitrary points throughout. Plus, selecting the next dependency to build is as simple as running a pop on the linked list. A linked list allows the topological order to be efficiently updated as `ThreadSafeGraph<Dependency>` is updated.


### Termination

The build daemon will automatically terminate after "sitting empty" for a specified time. For example, if a clang invocation de-registers with the daemon, leaving it with zero registered clang invocations. The daemon will wait `n` seconds before terminating itself.