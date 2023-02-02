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

When you first meet WAMR, want to give it a taste, either using `iwasm` to run a hello world wasm application, or embed a WAMR into your hello world program, I bet you won't even sweat a bit. But when you're up to do something serious, maybe use it for real world applications and start to think about performance and response time, things changes. At this time, the different running modes (AOT, interpreter, fast interpreter, fast JIT, LLVM JIT, multi-tier JIT) we proudly made, aiming to provide user more adaptability and flexibility, could backfire and encumber user.

Although we try to make CMake cache variables' name(CMake cache variables serve as the method to choose running mode at compile time) as explanatory as possible, because our background varies and the same term may mean differently to us, eventually, it can still cause confusion. And when it did, you may have to go back and forth from the documentation page to your develop environment, trying to understand each running mode are for, compile and test WAMR for your wasm app, see how does each running mode perform. We understand it's a chore (even worse, it maybe a dealbreaker for some people) to go through this process.

To address this potential problem, and to build common understanding of definition toward each running modes, I wrote this blog, hopefully, it could fasten your learning process to pick out the running modes of WAMR serve you best, and make your day just easier.

## Why? Why so many running modes?

Our goal is to provide user the adaptability and flexibility. User could choose one to achieve the best performance and resource usage according to deploy environment and nature of application. But as more running mode supported over the years, the learning curve for new user now is a little unfriendly. But let's go through this blog together, I believe when you finish reading it, you would understand our intention to develop each running mode, and feel confident to choose the one (or combination) you need.

## General introduction of each running mode

WAMR support six running modes, AOT, Classic Interpreter, Fast Interpreter, LLVM JIT, Fast JIT, Multi-tier JIT. Based on core concept and implementation behind, it could be roughly divided into three categories: AOT, Interpreter, JIT

- AOT: Ahead-of-Time compilation. As you can guess from the name, we first need to use the **wamrc compiler to compile wasm file to the AOT file**. Then it could be run with our iwasm vmcore. In this running mode, we could achieve the nearly native speed(the **fastest** of all running modes) with very **small footprint** and **quick startup**
It could run wasm applications in Interpreter/JIT running mode:

- Interpreter: Interpreters are very useful when **debugging or studying**, but their **performance is relatively poor** compared with other running modes. We support two running modes of the interpreter:
  - Classic Interpreter: plain interpreter running mode
  - Fast Interpreter: as you can guess from the name, the fast interpreter runs ~2X faster than the classic interpreter but consumes about 2X memory to hold the pre-compiled code.

- JIT: Using the **Just-in-Time** compilation technique, we could make iwasm run much **faster than Interpreter** mode and sometimes very close to the speed of AOT running mode. We support two running modes of JIT:
  - LLVM JIT: the JIT engine is implemented based on LLVM codegen. The **performance** of LLVM JIT is **better** than Fast JIT, with ~2x of the latter. But the **startup time** is **slower** than Fast JIT.
  - Fast JIT: the JIT engine is implemented based on self-implemented codegen and asmjit encoder library. It is a lightweight JIT engine with small footprint, **quick startup**, good portability and relatively good performance. Currently it supports x86-64 target and Linux/Linux-SGX/MacOS platforms. The **performance** of Fast JIT is **~50%** of the performance of LLVM JIT.
  - Multi-tier JIT: Multi-tier JIT: take advantage of both Fast JIT and LLVM JIT. Enable auto tier up from Fast JIT to LLVM JIT: first run wasm in Fast JIT, LLVM JIT threads run on background gradually replace Fast JIT function pointer, so after a short while later, if wasm run again it will be in LLVM JIT. It provides **both** a **quick startup** and a **decent performance**. Overall performance is excellent and serves as a rather nice trade-off compared with Fast JIT and LLVM JIT.

## How to set running mode

User can choose running mode both at compile time and at execution time.

### Set running mode at compile time

We understand that the current method we adopt (using the CMake cache variable) to control running modes at compile time may make users feel complex and weary. We are aware that the current method may have a certain learning curve and even take a toll on users' mentality. Now we are brainstorming and working on optimizing user experience, planning to use a config file to simplify choosing the compile options. If you have any other ideas, feel free to reach us on GitHub; let us know your thoughts by simply opening an issue!

But before the optimization gets actually implemented, we are stuck with manually setting CMake cache variables for a while. Hopefully, the cheat sheet below can serve as a quick reference facilitating you to write your program with WAMR.

#### Using iwasm

To compile iwasm in different running modes, you could simply set the CMake cache variable when compiling.

```sh
# if the build directory has a previous build, remember to clean them # so that each build has a clean environment
rm -r build/*
cmake -D{cache variable}=1 -B build
cmake --build build
```

#### Embed WAMR

To embed WAMR into your program, you could set the CMake cache variable in CMakeLists.txt

```CMake
set({cache variable} 1)
```

#### Cheatsheet

Here is the corresponding mapping table of Running mode ↔ CMake for quick reference (it's based on the default CMake setting, with some explicit set to make it clear):

> **_NOTE:_** if the build directory has a previous build, remember to clean them(`rm -r build/*`) so that each build has a clean environment

| Running mode        | CMake cache variable |
|     -----------     |     -----------      |
|  AOT                | WAMR_BUILD_AOT 1 |
|  Classic Interpreter | WAMR_BUILD_INTERP 1 WAMR_BUILD_FAST_INTERP 0|
|  Fast Interpreter | WAMR_BUILD_INTERP 1 WAMR_BUILD_FAST_INTERP 1  |
|  LLVM JIT         | WAMR_BUILD_INTERP 1 WAMR_BUILD_JIT 1 |
|  Fast JIT    | WAMR_BUILD_INTERP 1 WAMR_BUILD_FAST_JIT 1 |
|  Multi-tier JIT    | WAMR_BUILD_INTERP 1 WAMR_BUILD_JIT 1 WAMR_BUILD_FAST_JIT 1 |

For example, if you want to use a Multi-tier JIT running mode.

- when you are building iwasm, you can set the CMake cache variable in the command line

  ```sh
  cmake -DWAMR_BUILD_INTERP=1 -DWAMR_BUILD_JIT=1 -DWAMR_BUILD_FAST_JIT=1 -B build
  ```

- when you choose to embed WAMR, you can set the CMake cache variable in CMakeLists.txt
  
  ```CMake
  set(WAMR_BUILD_INTERP 1)
  set(WAMR_BUILD_JIT 1)
  set(WAMR_BUILD_FAST_JIT 1)
  ```

### Set running mode at execution time

We are really excited to announce that we have added [a new feature](https://github.com/bytecodealliance/wasm-micro-runtime/issues/1842) to allow the user control of the default/module instance-specific running mode at execution time, of course, **the running mode you choose need to be enabled at compile time**. You could probably guess the enabled modes from the CMake cache variable. For example, for Multi-tier JIT, the cache variable is WAMR_BUILD_INTERP 1 WAMR_BUILD_JIT 1 WAMR_BUILD_FAST_JIT 1, so you could choose Interpreter, LLVM JIT, FAST JIT, and Multi-tier JIT to run.

At the point time when this blog was written, most work was done, and it's in the dev/running_modes branch, which will merge with the main branch soon.

#### Using iwasm

Four command line options to control the running mode of iwasm:

- `--interp`: run iwasm in class interpreter mode
- `--fast-jit`: run iwasm in  fast jit mode
- `--llvm-jit`: run iwasm in llvm jit mode
- `--multi-tier-jit`: run iwasm in multi-tier jit mode

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

#### Embed WAMR

**We added APIs that can control running mode in different granularity:**

- at runtime level:

    ```C
    // query whether a running mode is supported:
    bool wasm_runtime_is_running_mode_supported(RunningMode runnnig_mode);
    // set the default running mode for the entire runtime(for all module instances):
    bool wasm_runtime_set_default_running_mode(RunningMode runnnig_mode);
    ```

- at module instance level:

    ```C
    // set the default running mode for a module instance
    bool wasm_runtime_set_running_mode(wasm_module_inst_t module_inst, RunningMode running_mode);
    // get the running mode for a module instance
    RunningMode wasm_runtime_get_running_mode(wasm_module_inst_t module_inst);
    ```
  
**The priority of choosing running mode:**

1. User set module instance running mode
2. User set default running mode
3. Compiled default running mode

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
