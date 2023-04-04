# Modules Build Daemon: Build System Agnostic Support for Explicitly Built Modules [GSoC Proposal]

## Motivation

Modules have the potential to significantly enhance compile-time performance, as they eliminate the need for repetitive processing of header files during textual inclusion builds. By processing a module once and reusing it across all associated translation units, modules offer a more efficient approach to managing build dependencies. Clang currently supports two methods for building modules: implicit and explicit. While explicit modules boast speed, they demand considerable development effort from build systems. Implicit modules, on the other hand, integrate smoothly with existing workflows but lack efficiency. "Modules Build Daemon: Build System Agnostic Support for Explicitly Built Modules" aims to balance these two approaches, enabling developers to reap the benefits of explicit modules irrespective of their build system.

## Overview

This project aims to implement a daemon that serves as a build system manager for modules, providing support for explicitly built modules irrespective of the build system. By simply incorporating a single command line flag, each Clang invocation registers its translation unit with the daemon, which then scans the unit's dependencies. As translation units are registered and analyzed, the daemon constructs a dependency graph for the entire project. Concurrently, it utilizes the emerging graph to schedule and build each module's AST. This approach allows for a single, comprehensive entity to effectively coordinate and manage the build of modules throughout the entire build process.

---
## Project Priorities

During the initial phase of development, the primary focus will be on parallel builds for Unix systems using both traditional Clang modules and C++ standard modules. In the `Future Work` section, I outline future development plans that extend beyond the scope of GSoC.

It is important to note that when it comes to C++ standard modules the build daemon does not attempt to replace any existing build systems. When possible it will be perferable to use a tool like `CMake` which supports explicit modules. (https://www.kitware.com/import-cmake-c20-modules/) The build daemon's purpose is to cover cases when a build system is not available. This is most common with large projects using traditional Clang modules and with tools like debugger and indexer.

---
## Project Details

Development will be split into three phases
1. Integrate build daemon flag into Clang driver
2. Setup build daemon infrastructure
3. Implement core build daemon functionality

---
**Phase 1: integrate build daemon flag into the Clang driver.**

The Clang driver consists of five stages: Parse, Pipeline, Bind, Translate, and Execute. Phase 1 focuses on ensuring that the build daemon flag is properly handled throughout all five stages.

> 1. Parse: Option Parsing

The Clang driver will recognize `-fmodule-build-daemon` as a valid command line option

```console
$ Clang++ -fmodule-build-daemon foo.cpp bar.cpp -o test
```
```cpp
// Options.td

def fmodule-build-daemon : Flag<["-"], "fmodule-build-daemon">, 
	Group<f_Group>, Flags<[NoXarchOption]>, 
	HelpText<"Enable module build daemon functionality">;
```

> 2. Pipeline: Compilation Action

There will be no change to the pipeline stage. The module build deamon fits in well with `2: compiler` as it contributes to the IR that will be handed to the backend.

``` console
$ Clang++ -ccc-print-phases -fmodule-build-daemon foo.cpp

            +- 0: input, "foo.cpp", c++
         +- 1: preprocessor, {0}, c++-cpp-output
      +- 2: compiler, {1}, ir
   +- 3: backend, {2}, assembler
+- 4: assembler, {3}, object
5: linker, {4}, image
```

> 3. Bind: Tool & Filename Selection

The `ToolChain` will still select `Clang` as the appropriate tool.

``` console
$ Clang++ -ccc-print-bindings -fmodule-build-daemon foo.cpp -o test

"x86_64-unknown-linux-gnu" - "Clang", inputs: ["foo.cpp"], output: "/tmp/foo-f45458.o"
"x86_64-unknown-linux-gnu" - "GNU::Linker", inputs: ["/tmp/foo-f45458.o"], output: "test"
```

> 4. Translate: Tool Specific Argument Translation

When `-fmodule-build-daemon` is passed to the Clang driver Translate must include `-cc1modbuildd` as the first flag with each Clang invocation. See more discussion on `-cc1modbuildd` in `Phase 2: Initialization`.

```console
$ Clang++ -### -fmodule-build-daemon foo.cpp bar.cpp -o test

"Clang-17" "-cc1modbuildd" "-o" "/tmp/foo-66a77d.o" "-x" "c++" "foo.cpp"
"Clang-17" "-cc1modbuildd" "-o" "/tmp/bar-73584c.o" "-x" "c++" "bar.cpp"
"ld" ... "-o" "test" ... "/tmp/foo-66a77d.o" "/tmp/bar-73584c.o" ...
```
```cpp
// Clang/lib/Driver/ToolChains/Clang.cpp

void Clang::ConstructJob(Compilation &C, const JobAction &Job, ...) {
    if (Job.getKind() == Action::ModBuildJobClass) {
        CmdArgs.push_back("-cc1modbuildd");
    }
}
```

> 5. Execute

Compilation is executed.

---
**Phase 2: Setup build daemon infrastructure**

The goal of phase 2 is to implement the boiler plate and infrastructure required to develop the build daemon's core functionality. This includes the ability for Clang invocations to spawn the build deamon, the ability for Clang invocations to register with the build deamon, and a mechanism to terminate the build deamon. There is an existing daemon implementation that can scan file dependencies in a downstream fork (https://github.com/apple/llvm-project/blob/next/Clang/tools/driver/cc1depscan_main.cpp) that will be used as the basis for phase 2 development.

To prevent duplicate code between `cc1depscand` and `cc1modbuildd` it would be a good idea to move common daemon functionality out of `cc1depscand` files to another location. Perhaps a `llvm-project/Clang/tools/daemon` directory could be created.

> Daemon Details

The build daemon will use Unix sockets for inter-process communication. Windows is in the process of supporting Unix sockets (https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/), which will allow the build daemon to be portable.

> Initialization

When `-fmodule-build-daemon` is passed to the Clang driver `-cc1modbuildd` will be included as the first command line option for each Clang invocation. Once instantiated the Clang invocation will look for a running daemon. If the daemon exists the Clang invocation will register with it. If the daemon does not exist the Clang invocation will first initialize the daemon then register with it.

```cpp
// Clang/tools/driver/driver.cpp

int Clang_main() {
    if (Args.size() >= 2 && StringRef(Args[1]).startswith("-cc1"))
        return ExecuteCC1Tool();
}

static int ExecuteCC1Tool(SmallVectorImpl<const char *> &ArgV, 
                          const llvm::ToolContext &ToolContext) {
	
    StringRef Tool = ArgV[1];
    if (Tool == "-cc1")
        return cc1_main(ArrayRef(ArgV).slice(1), ArgV[0], GetExecutablePathVP);

    #if LLVM_ON_UNIX
    if (Tool == "-cc1modbuildd")
        return cc1modbuildd_main(ArrayRef(ArgV).slice(1), ArgV[0], GetExecutablePathVP);
    #endif /*LLVM_ON_UNIX*/
}
```

```cpp
// cc1modbuildd_main.cpp

int cc1modbuildd_main() {

	bool NoSpawnDaemon = (bool)Sharing.Path;
	auto Daemon = NoSpawnDaemon
                    ? Daemon::connectToDaemonAndShakeHands(Path)
                    : Daemon::constructAndShakeHands(Path, Exec, Sharing);

	// register with daemon
	CC1DaemonProtocol Comms(*Daemon);
	Comms.putCommand(WorkingDirectory, OldArgs, Mapping);
}
```

> Termination

The build daemon will automatically terminate after "sitting empty" for a specified amount of time. For example, if a Clang invocation de-registers with the daemon, leaving it with zero registered Clang invocations. The daemon will wait `n` seconds before terminating itself. By using a time limit, the daemon may persist across a large project. 

---
**Phase 3: Implement core build daemon functionality**

The goal of phase 3 is to implement scanning, scheduling, cache management, and module building. Phase 3 is the largest phase by far. 

> Scanning

While `cc1depscan_main.cpp` implements a scanning daemon it is limited to file dependencies. So, the build daemon will use the scanning tool `Clang-scan-deps`. `Clang-scan-deps` can be integrated into the build daemon by relying on `class FullDeps`.

`FullDeps` or more specifically the `FullDeps` attribute `Modules` will represet all dependencies for the daemon. The scans for each translation unit will be merged into `Modules` using the public method `mergeDeps`. 

```cpp
// Clang/tools/Clang-scan-deps/ClangScanDeps.cpp

class FullDeps
	public:
		void mergeDeps(StringRef Input, TranslationUnitDeps TUDeps, size_t InputIndex);
	private:
		std::unordered_map<IndexedModuleID, ModuleDeps, IndexedModuleIDHasher> Modules;

struct ModuleDeps
{
	ModuleID ID; // module ID
	llvm::StringSet<> FileDeps; // collection of paths to direct dependencies
}
```

> Scheduling & Building

Scheduling will be done in accordance with a deterministic topological sort. The order of dependencies will be based first on the number of translation units that require a dependency then second on the alphabetical ordering of the dependencies. 

For example, the build daemon receives its first registration, `lib/Parse/ParseAST.cpp`, and creates a graph of its dependencies. At first, since `lib/Parse/ParseASTcpp` is the only translation unit registered with the build daemon, the dependencies all have the same weight of `1` and are ordered in alphabetical order.

<img src="parseast_schedule.PNG" width="100%" height="100%">

Now, a second translation unit, `lib/Sema/SemaConcept.cpp`, is registered with the build daemon. The build daemon scans its dependencies and incorporates the new translation unit's dependency graph into the project's dependency graph. 

<img src="semaconcept_graph.PNG" width="25%" height="25%">

The weights and build schedule are updated to reflect the new information.

<img src="project_schedule.PNG" width="100%" height="100%">

> Cache Management

The build module will cache precompiled modules, as Clang AST files. These Clang AST files encode the AST as well as its associated data structures in a compressed bitstream format. One of the advantages of using Clang AST files is that they can be linked together to depict the dependency graph of a project, making them an ideal choice for the build daemon.

Initially, the precompiled modules will be exclusively stored on disk. This approach allows most modern computers to simultaneously hold all of the precompiled modules in the cache. However, to ensure compatibility with systems that have limited resources, the build module will also include a cache invalidation mechanism.

The cache invalidation mechanism will determine which modules to retain in the cache based on three factors: frequency of use, time-to-live (TTL), and the project's dependency graph. Each time a module is built or accessed, its TTL will be recalculated using specific equations that take these factors into account.

```cpp
// scenario one: module is compiled and added to the cache
initial_ttl = depth_in_DAG * SCALING_FACTOR_1;

// scenario two: precompiled module is used
new_ttl = curent_ttl + (depth_in_DAG * SCALING_FACTOR_2);
```
Once a module's ttl has expired, it will be flagged as a candidate for deletion. If multiple modules have expired, the cache invalidator will start with the module that is least frequently used.

> Rebuilds

The proposal thus far has primarily addressed building software from scratch. However, in most real-world scenarios, build systems such as `Ninja` and `Make` leverage a cache of object files to optimize the build process. Specifically, these systems recompile only the translation units that have undergone changes since the last build. To identify such units, build systems compare the timestamps of source and object files. The same strategy will be adopted by the build daemon for handling precompiled modules.

When a Clang invocation communicates with the build daemon, the daemon generates a dependency graph and consults the cache to determine whether the modules have already been built. If the modules exist, the daemon compares the timestamps of the source file and the precompiled module. When the source file has a more recent timestamp, indicating an update since the last build, the module is rebuilt. Additionally, any dependent modules that rely on the updated module are also rebuilt as part of the process.

Currently, the build daemon can only be spawned by a Clang invocation. Consequently, the daemon's ability to check whether a precompiled module  should be rebuilt is contingent upon the build system's decision to rebuild the translation unit. This limitation presents a challenge that, while acknowledged, falls outside the scope of the GSoC project. It will be addressed during future development.

---
## Timeline

Review of official timeline
- March 20 to April 4: GSoC application period
- May 4: Accepted projects announced
- May 29 to August 28 (13 weeks): Work!

Project Timeline

- Phase 0: May 4 to May 28 (3.5 weeks)

    - Task 1: finalize implementation details
        - Verification Approach: complete a detailed algorithm design for each core feature of the build daemon (scanning, caching, scheduling) and create a list of subtasks for each task

- Phase 1: May 29 - June 11 (2 weeks)

    - Task 1: add `-fmodule-build-daemon` flag to Clang so that it is recognized as a valid flag
        - Verification Approach: a developer will be able to build any valid Clang project with `-fmodule-build-daemon` included in the list of flags.
        - Notes: At this point the flag `-fmodule-build-daemon` will not have any functionality but will simply be a valid option and serve as a starting point for further development

    - Task 2: create a new Job so that `-cc1modbuildd` can be passed to the Clang front end
        - Verification Approach: when a developer uses `-fmodule-build-daemon` then `-cc1modbuildd` will be passed to the Clang front end
        - Notes: at this point the flag `-cc1modbuildd` will not have any functionality

- Phase 2: June 12 - July 2 (3 weeks)

    - Task 1: create the skeleton of the build daemon
        - Verification Approach: When `-fmodule-build-daemon` is passed to the Clang driver a build daemon is initialized and terminates after a specified amount of time
        - Notes: will be based on `cc1depscan`

    - Task 2: implement ability for Clang invokations to register and unregister with the daemon
        - Verification Approach: When `-fmodule-build-daemon` is passed to the Clang driver the build daemon will maintain a list of active Clang invocations


- Phase 3: July 3 - August 28 (8 weeks)

    - Task 1: Implement ability for build daemon to scan dependencies of registered Clang invocations
        - Verification Approach: As Clang invocations are registered with the build daemon the daemon will generate a dependency graph for the registered translation unit.
        - Note: to verify, the dependency graph for each translation unit will be written to a log file

    - Task 2: Implement ability for build daemon to construct a project wide dependency graph
        - Verification Approach: The build daemon should maintian a dependency graph representative of all translation units scanned
        - Note: In addition to the translation unit dependency graph logs the build daemon will write the project wide dependency graph to a log file before terminating

    - Task 3: Implement ability for build daemon to schedule dependency builds
        - Verification Approach: based on the project wide dependency graph the build daemon should mainitain a build list that specifies the order modules are to be built
        - Note: after each scan is completed the build list will be updated

    - Task 4: Implement ability for build daemon to spawn Clang invocations
        - Verification Approach: build daemon will use the build list to spawn Clang invocations, build modules, ans store AST files in memory
        - Note: all AST files will simply be stored on disk. There will be no cache invalidation

    - Task 5: Implement ability for build daemon to return module's AST files to the Clang invocation
        - Verification Approach: Origional Clang invocation can used modules build by daemon to compile a translation unit
        - Note: Up until this point Clang invocations will register with the build daemon but still use the implicit system. After this task is complete the Clang invocation will use the modules built by the daemon.

    - Task 6: Implement cache management
        - Verification approach: When the system runs out of space the cache invalidator will be smart about how it makes room


---
## Future Work

I am excited about the project and will not be able to accomplish everything I would like to in the alloted time period. I have listed a few enhancements I look foward to tackling after GSoC.

- Implement support for Windows
- Implement support for distributed builds
- Add flags to customize deamon behavior
    - Time before termination
	- Cache size
- Schedule and cache optimization
	- Add a "hot" cache that stores precompiled modules in memory