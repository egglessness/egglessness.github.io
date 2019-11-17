---
layout: post
title: "Testing a MachineFunctionPass on LLVM without having to recompile every time"
description: "Recompiling the whole LLVM infrastructure is a very tedious and long process. I'll show you how to get away with it in just a few seconds."
date: 2018-10-17
tags: [llvm, tutorial]
comments: true
share: true
keywords:
    - llvm
    - machine function pass
    - machinefunctionpass
    - llc
    - compile
    - cmake
---

LLVM is an extremely beautiful project: its modularity enables anyone to extend the compiler infrastructure by writing modules to be used either on the backend or the frontend.

When it comes to extend the backend, however, LLVM gives us the possibility to choose the abstraction layer we want to work on: on one hand we can write passes that operate at the IR (Intermediate Representation) level, on the other we can go even deeper, writing a `MachineFunctionPass` to directly work at the assembly level of our target architecture.

While it is possible to develop normal passes outside the LLVM source tree, and dynamically load them using the `opt` tool, if we have to work on `MachineFunctionPasses` we have to recompile the whole backend each time we made a tiny modification to the pass code.   
This is due to the fact that our `MachineFunctionPass` is used only in the late stages of machine code generation, on MIR code (Machine IR, much more lower level); for this reason those passes are tightly coupled with the LLVM backend.

Since recompiling everything each time is unfeasible (it would take hours), I'll show you a way to selectively recompile only parts of the backend, getting things done in seconds!

----------

### Compiling LLVM
We will start from scratch: we will setup a faster and smarter build system (`ninja-build`), compile everything and once we have done with this, we won't have to wait hours anymore.

1. **Install** all the **prerequisites**:

    ```
    $ sudo apt install xz-utils cmake ninja-build clang
    ```
    
2. **Download** LLVM 7.0 **sources** from [here](http://releases.llvm.org/7.0.0/llvm-7.0.0.src.tar.xz)
3. **Unpack** them in a directory of your choice which will refer to as `[LLVM_SRC]`. I personally created a new directory outside the source folder.

4. Create a **build directory**:

    ```
    $ mkdir llvm-build
    $ cd llvm-build
    ```
4. Let's **configure** the build environment, instructing `cmake` as follows:

   ```
    $ cmake -DCMAKE_BUILD_TYPE=Debug -DLLVM_TARGETS_TO_BUILD=X86 -DBUILD_SHARED_LIBS=ON \
      ../llvm-7.0.0.src -GNinja
    ```
    As you can see, there are a couple of flags that are worth to be mentioned:
    - `-DCMAKE_BUILD_TYPE=Debug`: just to obtain a debug build (more flexible)
    - `-DLLVM_TARGETS_TO_BUILD=X86`: since we're interested only in the X86 platform, we don't want to lose time compiling the backend also for all the other platforms, such as ARM, MIPS, SPARC, etc. This speeds up the compilation process, and make us save up to 4 GB of disk space.
    - `-DBUILD_SHARED_LIBS=ON`: shared code is moved in `.so` libraries, that can be linked at runtime, thus speeding up the compilation process even more.
    - `-GNinja`: specifies to use `ninja` as build generator. By using `ninja` the overall compile time can decrease by more than 50% (it seems that it has better support to multithreading), but most importantly we can invoke a specific command to compile only `llc`.
    
5. Now start the actual **compilation** within your build directory

    ```
    $ cmake --build .
    ```

    Building takes some time to finish. 

6. Finally, we can create a symbolic link to our custom version of `llc`, in order to call it in a simpler way:

    ```
    $ sudo ln -s [BUILD-PATH]/bin/llc /usr/local/bin/llc
    ```
        
#### Recompiling LLC 
At this stage, every time we modify our `MachineFunctionPass`, we just have to tell `ninja-build` to recompile only `llc` (LLVM system compiler), without having to recompile the whole backend. 

This is just a matter of seconds by running:

```
$ ninja llc
```



### Running experiments
`llc` works at the IR level, so we have to generate the `.ll` file out of our C program:

```
clang -O0 -S -emit-llvm hello.c -o hello.ll
```

then we have to run only the code generation use `llc`:

```
llc hello.ll
```

The output is an `asm` file, that can be compiled simply with `gcc`:

```
gcc hello.s -o hello
```


----------
That's it!


I hope this will speed up your development as it did with me! ;)