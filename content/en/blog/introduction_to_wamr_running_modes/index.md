---
title: "Introduction to WAMR running modes"
description: ""
excerpt: ""
date: 2023-01-19T17:20:07+08:00
lastmod: 2023-01-28T17:20:07+08:00
draft: false
weight: 50
images: []
categories: ['introduction']
tags: []
contributors: ['Tianlong Liang']
pinned: false
homepage: false
---


## The options of running a Wasm module
Usually, a WebAssembly module can be executed in either interpreter, Just-In-Time (JIT), or Ahead-Of-Time (AOT) compilation mode, and the choice can be based on the preference for execute performance, resource, etc.   

WAMR supports all three modes and even more:
- **AOT**:  WAMR AOT helps to achieve nearly native speed, very **small footprint**, and **quick startup**. Use the wamrc compiler to compile wasm file to the AOT file, and then run it on iwasm vmcore.
- **Interpreter**: Small footprint, small memory consumption, and relatively slow. WAMR offers two interpreters:
  - **Classic Interpreter (CI)**: A textbook implementation of Wasm interpreter. It is currently needed for supporting source debugging.
  - **Fast Interpreter (FI)**: Precompile the Wasm opcode to internal opcode and runs ~2X faster than the classic interpreter, but it consumes a bit more memory than CI.
- **JIT**: Run Wasm at nearly native speed yet keeps Wasm as distribution media which is platform-agnostic. The cost is compilation during execution. WAMR supports two JIT layers:
  - **LLVM JIT**: Based on LLVM framework and offer the best execution **performance**. Its cost is the longer compilation time.
  - **Fast JIT**: A lightweight JIT engine with a small footprint, quick **startup**, yet good performance. Currently, it supports x86-64 arch and Linux/Linux-SGX/MacOS platforms. 

**JIT layers tier-up on the fly**: WAMR supports switching from Fast JIT to LLVM JIT during Wasm execution, which provides both a quick cold start with Fast JIT and excellent performance with LLVM JIT.  
   ![](wamr_jit_tier_up.png)


## The considerations of Wasm execution mode 

When you use WAMR in your products, there are a few things that should be considered about the Wasm execution mode: 
1. At compilation time, what execution engines should be compiled in?
2. At running time, how to determine the execution mode for a loaded Wasm module?

### Compile the engines

Except that the Fast Interpreter can't co-exist with other execution engines in a software binary, all other execution engines can be built into single software at any combination.

WAMR offers several CMake cache variables for the compilation control of the execution engines:

| Execution Engine        | CMake cache variable |
|     -----------     |     -----------      |
|  AOT                | WAMR_BUILD_AOT=1 |
|  Classic Interpreter | WAMR_BUILD_INTERP=1, WAMR_BUILD_FAST_INTERP=0|
|  Fast Interpreter | WAMR_BUILD_INTERP=1, WAMR_BUILD_FAST_INTERP=1  |
|  LLVM JIT         | WAMR_BUILD_JIT=1 |
|  Fast JIT    | WAMR_BUILD_FAST_JIT=1 |

> Note: Building LLVM JIT and Fast JIT will automatically include the Classic Interpreter since they depend on CI. It means when WAMR_BUILD_JIT or WAMR_BUILD_FAST_JIT is enabled for CMAKE, WAMR_BUILD_INTERP will be turned on automatically.

The example of CMake build command below compiles the Classic Interpreter, LLVM JIT, and Fast JIT Execution engine into the runtime binary.
  ```sh
  cmake -DWAMR_BUILD_INTERP=1 -DWAMR_BUILD_JIT=1 -DWAMR_BUILD_FAST_JIT=1 -B build
  ```


### Control the execution mode 

The software that embeds the WAMR can fully control what execution mode for a loaded Wasm module, by calling the runtime APIs. 
- set runtime default mode:
    ```C
    // query whether a running mode is supported:
    bool wasm_runtime_is_running_mode_supported(RunningMode runnnig_mode);
    // set the default running mode for the entire runtime(for all module instances):
    bool wasm_runtime_set_default_running_mode(RunningMode runnnig_mode);
    ```
- set module instance:
    ```C
    // set the default running mode for a module instance
    bool wasm_runtime_set_running_mode(wasm_module_inst_t module_inst, RunningMode running_mode);
    // get the running mode for a module instance
    RunningMode wasm_runtime_get_running_mode(wasm_module_inst_t module_inst);
    ```

- definition of running mode:
    ```C
    typedef enum RunningMode{
        Mode_Interp = 1,     // interpreter
        Mode_Fast_JIT,       // fast jit
        Mode_LLVM_JIT,       // llvm jit
        Mode_Multi_Tier_JIT, // multi-tier jit
    } RunningMode;
    ```  
Notes: 
1. the running mode is supported only when the related execution engine is built into the binary. 
2. The `Mode_Multi_Tier_JIT` enables tier-up from Fast JIT to LLVM JIT. It is supported only when both Fast JIT and LLVM JIT are available in the runtime software.
3. The selection of fast interpreter and classic interpreter is determined in compilation time, since only one interpreter can be built into the binary.

**The priority of choosing running mode:**  
1. User set module instance running mode
2. User set default running mode of the whole runtime
3. If the user didn't set it at runtime or instance levels, the following order will be used for the selection:  
    (1) The JIT layers tier-up on the fly if both JIT layers are available  
    (2) LLVM JIT execution, if available  
    (3) Fast JIT execution, if available  
    (4) Interpreter  
    
**A simple usage of APIs:**

```C
...
init_args.running_mode = Mode_Fast_JIT;
...
// initialize the default running mode for entire runtime(all module instance)
wasm_runtime_full_init(&init_args);
...
// alternatively, you could set the default running mode the later
wasm_runtime_set_default_running_mode(Mode_Interp);
...
buffer = bh_read_file_to_buffer("a.wasm", &size);
module = wasm_runtime_load(buffer, size, error_buf, sizeof(error_buf));
module_inst = wasm_runtime_instantiate(module, stack_size, heap_size,error_buf, sizeof(error_buf));
...
// will run in default running mode
wasm_runtime_call_wasm(exec_env, a_func, 1, wasm_argv);
...
buffer_b = bh_read_file_to_buffer("b.wasm", &size);
module_b = wasm_runtime_load(buffer_b, size, error_buf, sizeof(error_buf));
module_inst_b = wasm_runtime_instantiate(module_b, stack_size, heap_size,error_buf, sizeof(error_buf));
wasm_runtime_set_running_mode(module_inst_b, Mode_Multi_Tier_JIT);
...
// will run in user set module instance running mode
wasm_runtime_call_wasm(exec_env, b_func, 1, wasm_argv);
```

## Try out iwasm from WAMR binary release

If you want to have a quick try, it would be a good option to download the WAMR from [binary release](https://github.com/bytecodealliance/wasm-micro-runtime/releases) and start the iwasm command.

There are four command line options to control the running modes of iwasm:
- `--interp`: run iwasm in classic interpreter mode
- `--fast-jit`: run iwasm in fast jit mode
- `--llvm-jit`: run iwasm in llvm jit mode
- `--multi-tier-jit`: run iwasm in multi-tier jit mode

Command examples:  
```sh
# default mode: multi-tier jit will be run
iwasm test.wasm
# or could explicitly run multi-tier jit
iwasm --multi-tier-jit test.wasm
# choose to run llvm jit mode
iwasm --llvm-jit --llvm-jit-opt-level=3 test.wasm
# choose to run fast jit mode
iwasm --fast-jit test.wasm
# choose to run interpreter mode
iwasm --interp test.wasm
```
