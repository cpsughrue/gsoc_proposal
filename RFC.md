# [RFC] Modules Build Daemon: Build System Agnostic Support for Explicitly Built Modules

## Abstract

Modules have the potential to significantly improve compile-time performance, as they eliminate the repetitive processing of header files that occur during textual inclusion builds. By processing a module once and reusing it across all associated translation units, modules offer a more efficient approach to managing dependencies. Clang currently supports two methods for building modules: implicit and explicit. With the explicit method the build system has complete knowledge of a translation unit's dependency graph before building begins while with the implicit method the build system discovers dependencies as modules are built. While the explicit method boosts speed, they demand considerable development effort from build systems. The implicit method, on the other hand, integrate seamlessly with existing workflows but is inefficient. "Modules Build Daemon: Build System Agnostic Support for Explicitly Built Modules" aims to balance these two approaches, enabling developers to reap the benefits of explicit modules irrespective of their build system. This project aims to implement a daemon that serves as a module manager. By simply incorporating a single command line flag, each Clang invocation registers its translation unit with the daemon, which then scans the unit's dependencies. As translation units are registered and analyzed, the daemon constructs a dependency graph for the entire project. Concurrently, it uses the emerging graph to schedule and compile each module. This approach allows for a single entity to effectively coordinate the build of modules.

## Scope

For the purpose of Google Summer of Code, development will be focused on providing support for Unix-like systems. I would like to keep Windows and other operating systems in mind so that nothing needs to be reiarchitected when support for other operating systems are implemented down the road.

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

When `-fmodule-build-daemon` is passed to the clang driver, the driver will check to see if a module build daemon is already running. If so, the driver will only launch clang with `-cc1`.

```console
# module build daemon already running
$ clang++ -### -fmodule-build-daemon foo.cpp -o foo

"clang-17" "-cc1" "-fmodule-build-daemon" "-o" "/tmp/foo-73584c.o" "-x" "c++" "foo.cpp"
```

If the daemon is not running, the driver will launch clang with `-cc1modbuildd` to spawn the module build daemon then launch clang with `-cc1`.

```console
# module build daemon not already running
$ clang++ -### -fmodule-build-daemon foo.cpp -o foo

"clang-17" "-cc1modbuildd"
"clang-17" "-cc1" "-fmodule-build-daemon" "-o" "/tmp/foo-73584c.o" "-x" "c++" "foo.cpp"
```

### Integration

If the clang binary is run with the flag `-cc1modbuildd` then `cc1modbuildd_main()` will be called instead of `cc1_main()`. By creating a separate entry point for the module build daemon, the daemon specific behavior can be encapsulated preventing the the compiler from turning into a build system.

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
    //     - when a thread builds the last dependency for a TU, the clang 
    //       invocation will disconnect, and the daemon will tell whichever 
    //       thread completed the build to shutdown 
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

int cc1modbuildd_main() {

    BuildServer.start();
    BuildServer.listen();
}
```

### Cache Validation

The daemon needs a way to check if source files have changed since the last time they were built. There are two common approaches: compare timestamps or compare hashes. Timestamps can be unreliable, especially with remote or virtual file systems, so the build system will use the hash of each file to check for changes. Once a module is built a new entry will be added to the cache_map file. If the daemon detects that the hash of the source file has changed then it knows to rebuild the module.

```
# cache_map

hash                                      file

7dd3a27d375652b36ef2a9e2d92a4c6f2e8845ec  DependencyScanningFilesystem.cppm
b1a43877e38980b3b73b1e39e2badf81a8157c72  DependencyScanningService.cppm
```

### Build Session

By managing build sessions, cache validation becomes more reliable and efficient. At the beginning of a build session, as clang invocations request compiled modules, the daemon will validate the cache by comparing hashes. Once the daemon validates a source's cache, the daemon only needs to check that any subsequent clang invocations come from the same build session to reuse the cache. Luckly clang provides support for defining a build session.

```
-fbuild-session-file=<file>
Use the last modification time of <file> as the build session timestamp

-fbuild-session-timestamp=<time since Epoch in seconds>
Time when the current build session started
```

The build session ID will be stored alongside hash information and file names in the `cache_map` file.

```
# cache_map

build_ID   hash                                      file

1685846174 7dd3a27d375652b36ef2a9e2d92a4c6f2e8845ec  DependencyScanningFilesystem.cppm
1685846174 b1a43877e38980b3b73b1e39e2badf81a8157c72  DependencyScanningService.cppm
```

### Cache Management

The cache will consist of compiled dependencies and the dependency graph from previous builds.

The build daemon will cache precompiled modules as Clang AST files. These Clang AST files encode the AST and associated data structures in a compressed bitstream format. Initially, the daemon will store cached dependencies exclusively on disk. Most modern systems can hold all of the built dependencies on disk. However, the build module will incorporate a cache management mechanism to ensure support for systems with limited resources. The daemon will support the same two flags provided by clang to clean the cache.

```bash
# Specify the interval (in seconds) after which a module file will be considered unused
-fmodules-prune-after=<seconds>

# Specify the interval (in seconds) between attempts to prune the module cache
-fmodules-prune-interval=<seconds>
```

### Scanning

If no cached dependency graph exists, the daemon will construct one for each translation unit as they register with the daemon. If a cached dependency graph exits, the daemon must validate it with every new build session. To validate the dependency graph, the daemon will check the consistency of each file in the translation unit's dependency graph, including the translation unit itself. If no files have changed and the context hash matches the previous build session, the daemon does not have to re-scan the translation unit. If either the translation unit, any of its dependencies, or the context hash has changed since the last build session, the daemon will re-scan the translation unit.

The daemon will use the `DependencyScanning` utilities provided under `llvm-project/clang/lib/Tooling` originally developed for `clang-scan-deps` to complete the scan.

```cpp
// llvm-project/clang/tools/driver/cc1modbuildd_main.cpp

scanDependencies(Client client, ThreadSafeGraph<Dependency>& depsGraph()) {

    TranslationUnitDeps TUDeps;
    TUDeps = DependencyScanningTool::getTranslationUnitDependencies();    
    handleTranslationUnitResults();
}
```

`handleTranslationUnitResults()` will merge `TranslationUnitDeps` into `ThreadSafeGraph<Dependency>`.


### Scheduling

There are two potential scheduling strategies:

1. The daemon will schedule dependencies based on a topological sort, prioritizing modules required by more translation units. If two dependencies are of the same priority in the topological sort, but one dependency is required by two translation units while the other is required by five translation units, the daemon will schedule the dependency required by five translation units to be built first.

2. The daemon will schedule dependencies based on a topological sort, prioritizing one translation unit at a time. The daemon will schedule all the dependencies for translation unit A to be built before it schedules any dependencies for translation unit B to be built.

I will conduct an analysis to determine which strategy results in faster build times.

### Termination

The build daemon will automatically terminate after "sitting empty" for a specified amount of time. For example, if a clang invocation de-registers with the daemon, leaving it with zero registered clang invocations. The daemon will wait the specified amount of time for a new clang invocation to register with itself before terminating.