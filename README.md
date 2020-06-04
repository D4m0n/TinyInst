# LiteInst

```
Copyright 2020 Google LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

## What is LiteInst?

Liteinst is a lightweight dynamic instrumentation library that can be used to instrument only selected module(s) in the process, while leaving the rest of the process to run natively. It is meant to be easy to understand, easy to hack on and easy to hack with. It is not designed to be compatible with all targets (more on that later).

### How does it compare to [DynamoRIO](https://dynamorio.org/) and [PIN](https://software.intel.com/en-us/articles/pintool)?

LiteInst is not meant as a replacement for complex instrumentation frameworks such as DynamoRIO and PIN, but rather an alternative for scenarios where a more lightweight solution would do. LiteInst assumes that the target is well-behaved (in the sense explained below) which is not the case for more complex frameworks. Thus, you probably won’t be able to successfully run LiteInst against malware as [was done with DynamoRIO previously](https://www.slideshare.net/MaximShudrak/fuzzing-malware-for-fun-profit-applying-coverageguided-fuzzing-to-find-bugs-in-modern-malware). On the other hand, if a target does not work with other frameworks due to the module that does not need to be instrumented, and the instrumented module is well-behaved, it might work with LiteInst. Because with LiteInst, most of the process will run natively, it will have shorter process startup time, and might outperform other solutions in cases where the target process spends a lot of time in the modules where instrumentation is not needed.

### How does it compare to [Mesos](https://github.com/gamozolabs/mesos) and [TrapFuzz](https://github.com/googleprojectzero/p0tools/tree/master/TrapFuzz)?

LiteInst is a full binary rewriting solution, so arbitrary behavior can be changed in the target module. This allows it, for example, to be able to extract edge coverage instead of only basic blocks. Additionally, LiteInst does not depend on other software, such as IDA Pro, to identify basic blocks.

### Which operating system does LiteInst support?

Currently Windows only (32- and 64-bit). Mac OS support might be considered in the future.

### Which targets are compatible with LiteInst?

LiteInst assumes all instrumented modules are well-behaved in the sense that

- There is no self-modifying code
- Return address on the stack is never accessed by the program directly
OR/AND (depending on the settings)
- No data is ever stored before the top of the stack (on addresses lower than pointed to by ESP/RSP). This condition can be relaxed into "no data before (ESP/RSP - arbitrary_offset)" using the `-sp_offset` flag.

### What is the performance overhead?

According to early measurements on image decoding, on a well-behaving 64-bit target with default LiteInst settings, the performance overhead was around 15% without a client and about 20% with the example coverage-collecting client. Note that this does not include the timeout introduced by initially instrumented the modules. See performance tips below for more details.

## Building LiteInst

1. Open a command prompt and set up your build environment, e.g. run vcvars64.bat / vcvars32.bat

2. Navigate to the directory containing the source

3. Run the following commands (change the generator according to the version of Visual Studio and platform you want to build for):

```
mkdir build
cd build
cmake -G"Visual Studio 15 2017 Win64" ..
cmake --build . --config Release
```

Note #1: 64-bit build will also run against 32-bit targets

Note #2: Encountering problems creating a 32-bit build on 64-bit windows due to the environment not being properly set up and libraries missing? Open the generated .sln file in Visual Studio and build from there instead of running cmake --build. Also note that 64-bit build is going to work on 32-bit targets, so creating a 32-bit build might not be necessary.

## Using LiteInst

LiteInst is primarily meant to be used as a library inside other programs.

A LiteInst client is written as a subclass of the LiteInst class. The client can then override the API methods it needs. The API methods are defined below.

After the client is created, it must be initialized with command line options by calling

`void init(int argc, char **argv);`

The command line options are defined below and a client can also define their own. After that, to run and control an instrumented program, the following functions can be used.

`DebuggerStatus Run(char *cmd, uint32_t timeout);`
`DebuggerStatus Attach(unsigned int pid, uint32_t timeout);`

These functions either run a program (using the specified command line) or attach to an already running program. If no target method is specified, the target will continue running until either the program exits, the program crashes, or the timeout (given in milliseconds) expires. If a target method is defined, LiteInst is going to return whenever the target method is entered and whenever target method returns, allowing the caller to perform additional tasks.

When `Run` and `Attach` return while the target process is still alive, the following functions can be used to either terminate the process or continue execution.

`DebuggerStatus Kill();`

`DebuggerStatus Continue(uint32_t timeout);`

LiteInst comes with an example coverage binary, which can be invoked using

`litecov.exe <options> -- <target command line>`

## Instrumentation API

### Debugger event callbacks 

These callbacks are for information only and the client should not emit any instrumented code during them. Clients must call the same handler defined in the superclass before handling these events themselves.

`OnProcessCreated`
Called when the target process is created or attached.

`OnProcessCreated`
Called when the target process exits.

`OnProcessEntrypoint`
Called when the process (main binary) entrypoint gets reached

`OnTargetMethodReached`
If the target method is defined, called when the target method is reached for the first time.

`OnModuleLoaded`
Called when a module is loaded. Called for each module, not just instrumented ones.

`OnModuleUnloaded`
Called when a module is unloaded. Called for each module, not just instrumented ones.

`OnException`
Called when an exception is encountered. The client must either return true (if the exception was handled) or the result of the same method on the parent class.

### Instrumentation callbacks

During these callbacks, the client can add code to the target by calling `WriteCode()`. 

`OnBasicBlock`
Can be used to insert code that's going to run on a particular basic block

`OnEdge`
Can be used to insert code that's going to run on a particular edge. Note: For performance reasons, this callback is only emitted on non-deterministic edges (i.e. conditional jumps) and indirect jumps/calls (e.g. `call rax`). For edges where the next basic block is always known given the previous basic block (e.g. `jmp offset`, `call offset`), no callback will be emitted.

`OnInstruction`
Can be used to modify the instruction or insert code before it. Depending on the return code the original instruction is either going to be emitted or not after the callback.

### Other callbacks

`OnModuleEntered`
Called when a control flow is transferred into an instrumented module from another module

`OnModuleInstrumented`
Called when a module gets instrumented. This happens generally when the process entrypoint is reached (if the target method is not defined) or when the target method is reached (if it is defined). The client can initialize its instrumentation-related data here

`OnModuleUninstrumented`
Called when instrumentation data is no longer valid and needs to be cleared. Note that this is not the same as module being unloaded as, by default, instrumentation persists across module unloads / reloads. This callback can be used to clear any instrumentation-related data in the client.

## Command Line Options

### Instrumentation related

`-instrument_module [module name]` specifies which module to instrument, multiple `-instrument_module` options can be specified to instrument multiple modules.

`-indirect_instrumentation [none|local|global|auto]` which instrumentation to use for indirect jump/calls

`-patch_return_addresses` - replaces return address with the original value, causes returns to be instrumented using whatever `-indirect_instrumentation` method is specified

`-persist_instrumentation_data` (default = true) Does not reinstrument module on module unloads / reloads. Only works if the module is loaded on the same address it was loaded before.

`-instrument_cross_module_calls` (default=true) If multiple `-instrument_module` modules are specified and one calls into another, jump to instrumented code of the other module without causing an exception (which would cause slowdowns).

`-stack_offset` (default=0) When saving context on the stack, leave this many bytes on top of the stack (before stack pointer) unchanged.

### Debugging related

`-trace_debug_events` - prints debugger events (modules loaded, exceptions, etc.) 

`-trace_basic_blocks` - prints basic blocks as they get executed

`-trace_module_entries` - prints all entries into instrumented code

### Target method and persistence

LiteInst allows user to define a target method. If a target method is defined, no code will be instrumented (everything will run natively) until the target method is reached for the first time. Additionally, LiteInst will break execution on the target method entry and exit.

`-target_module` - module containing the target method

`-target_method` - name of the target method. This only works if the target method is exported or you have symbols for the target module.

`-target_offset` - use when target method can't be specified by name. Relative address of the target method from the module base

`-loop` - if this flag is specified, LiteInst will run the target method in an infinite loop (or until Kill() is called or process terminates for another reason). Function arguments will be saved and restored between iterations. This is mainly used to force persistence for fuzzing.

`-nargs` - number of target method arguments to save between iterations. To be used together with `-loop`

`-callcon [ms64|stdcall|fastcall|thiscall]` - calling convention target method uses. To be used together with `-loop`


## Coverage module

Liteinst comes with an (example) coverage module, `LiteInstCoverage`. The coverage module can collect basic block or edge coverage (controlled using `-covtype` flag).

Special feature of the coverage module is that the coverage buffer in the target process is initially allocated as read-only, causing an exception the first time new coverage is encountered. Combined with an option to ignore a certain subset of coverage, this enables quickly querying if running the target with a given input resulted in new coverage or not.

## How LiteInst works?

LiteInst is built on top of a custom debugger. The debugger watches the target process for events such as modules being loaded, breakpoint being hit, exceptions being fired etc. The debugger also implements breakpoints and persistence if the target method is specified.

When a module to be instrumented is loaded, it is initially "instrumented" in the following way

- All executable regions in the module are marked as non-executable, while retaining other permissions (read/write) as they were originally. This causes an exception whenever control flow reaches an instrumented module, which is caught and handled by the debugger.

- An executable region of memory is allocated within 2GB of the original module address range. This is where the instrumented/rewritten code of the module will be placed. 2GB is important as it allows all instructions that use addressing in the form of [rip+offset] to be replaced with [rip+fixed_offset].

Whenever an instrumented module is entered (whether for the first time or any other time), the basic block that was hit is instrumented, together with all basic blocks that can be reliably discovered by recursively following conditional branches as well as direct calls and jumps (e.g. jmp offset, call offset).

This is sufficient to run the instrumented code because

- all direct jumps/calls will land in the instrumented code at the correct location

- all indirect jump/calls (e.g. call rax) will land in their original code location, which causes an exception, which the debugger resolves by replacing the instruction pointer with the corresponding location in the instrumented code.

However, while this works, note that it will cause an exception on every indirect call/jump whose target is in an instrumented module. Since exception handling is slow, instrumenting targets with a lot of indirection (e.g. virtual methods in C++, function pointers) will be slow without additional instrumentation.

### Instrumenting indirect calls and jumps

LiteInst can instrument indirect calls and jumps to avoid exceptions on (already-seen) indirect targets. An instrumented call/jump, instead of jumping to the original target, will instead jump to the head of the linked list of stubs. Each stub contains a pair of (original_target, translated_target). It tests if the jump/call target matches original_target, and if so, control flow is directed to translated_target. Otherwise, it jumps to the next stub. If the end of the list is reached, that means the jump/call target hasn’t been seen before. This will cause a breakpoint that is caught by the debugger, which will be resolved by creating another stub and inserting it into the list.

This mechanism can be implemented in 2 ways
- per-callsite (local) list
- global hashtable used by all indirect jumps/calls

Global hashtable results in better performance. Local (per-callsite list) allows getting correct edges (with correct source address) on indirect calls/jumps.

Note that on modern Windows, due to CFG, all indirect jump/calls happen from the same location, therefore with CFG-compiled binaries, it is impossible (without some kind of special handling) to get accurate edges anyway. This, along with the performance benefit, is the reason why global hashlist is the default method for handling indirect calls/jumps in LiteInst.

### Return address patching

By default, when a call happens in instrumented code, the return address being written is going to be the next instruction in the *instrumented code*. This works correctly in most cases, however it will cause problems if the target process ever accesses return addresses for purposes other than return. A notable example of this is stack unwinding in the exception handling (SEH) on 64-bit Windows. Therefore, targets that need to catch exceptions won’t work correctly with LiteInst by default.

At this time, LiteInst also has an option (exposed through `-patch_return_addresses` flag) to rewrite return addresses into their corresponding values in the non-instrumented code whenever a call occurs. Note that, without additional instrumentation, this causes an exception on every return (causing a massive slowdown). However, with `-patch_return_addresses`, return instructions also get instrumented similarly to indirect jumps/calls. While this resolves returns happening within a module, note that all returns from a non-instrumented into an instrumented module will still cause exceptions.

Since this is a fairly common case, more performant options for supporting exception handling in Windows will be investigated in the future. This can be accomplished using RtlAddFunctionTable / RtlAddGrowableFunctionTable / RtlInstallFunctionTableCallback APIs.

## Performance tips

The biggest overhead in LiteInst comes from an exception being thrown whenever an instrumented module is entered from a non-instrumented module. You can see these exceptions being triggered using the `-trace_module_entries` flag. Indirect jump/call instrumentation should be used whenever possible and return instrumentation should not be used whenever possible. LiteInst performs best on modules (or module groups) that are reasonably self-contained. For example if you have two modules, A and B, where A calls B often but only B is instrumented, this will cause a lot of slowdown. Better performance could be achieved by instrumenting both A and B.

## Debugging tips

Use `-trace_basic_blocks` to see basic blocks as they are being executed. You’ll see both the addresses in the instrumented code and the corresponding addresses in the non-instrumented code.

Use the OnException() callback to examine program state when the crash occurs.

## Disclaimer

This is not an official Google product.

