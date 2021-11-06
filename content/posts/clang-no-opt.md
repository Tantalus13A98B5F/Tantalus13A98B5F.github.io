---
title: "Clang: Don't Touch My Code!"
date: 2020-01-24T16:31:37+08:00
draft: false
tags: ["clang"]
categories: ["System"]

toc: true
mathjax: false
---

Clang/LLVM is a toolchain that is widely adopted by both researchers and the industry. Based on LLVM, compilers for new targets are built, and program analysis solutions are developed, thanks to the well-defined LLVM IR. Nevertheless, Clang/LLVM is also famous for some of its strange (often just "different" from gcc) behaviors and very aggressive optimizations.

Recently I am working on LLVM (7.0) for program analysis and instrumentation, and I think it's time to write about how we are dealing with unwanted optimizations. Here we start.

## Our Problem

We are doing a whole program analysis, thus we need to compile every source file to bitcode respectively, and then link them together to perform the analysis and instrumentation. Since there can be information loss during optimization, it's best to analyze an untouched bitcode file, and then perform the optimization after instrumentation. Something like,

```sh
$ clang -emit-llvm hello.c -c -o hello.bc
$ opt -load mypass.so -mypassname hello.bc >output.bc
$ clang -emit-llvm -O2 output.bc -o optimized.bc
```

However, the latter optimization is not happening, i.e. `output.bc` and `optmized.bc` are nearly the same.

## How is Clang calling optmizations?

Suspecting some action might be done in the frontend, we looked into what `clang` is doing internally. The `-###` argument may be helpful.

```sh
$ clang -\#\#\# hello.c -O2 -c -o hello.bc -emit-llvm
clang version 7.0.0
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /llvmroot/bin
 "/llvmroot/bin/clang-7" "-cc1" "-triple" "x86_64-unknown-linux-gnu" "-emit-llvm-bc"
 "-emit-llvm-uselists" "-disable-free" "-main-file-name" "hello.c" "-mrelocation-model"
 "static" "-mthread-model" "posix" "-fmath-errno" "-masm-verbose" "-mconstructor-aliases"
 "-munwind-tables" "-fuse-init-array" "-target-cpu" "x86-64" "-dwarf-column-info"
 "-debugger-tuning=gdb" "-momit-leaf-frame-pointer" "-coverage-notes-file" "/home/test/playground/hello.gcno"
 "-resource-dir" "/llvmroot/lib/clang/7.0.0" "-internal-isystem" "/usr/local/include"
 "-internal-isystem" "/llvmroot/lib/clang/7.0.0/include" "-internal-externc-isystem" "/usr/include/x86_64-linux-gnu"
 "-internal-externc-isystem" "/include" "-internal-externc-isystem" "/usr/include"
 "-O2" "-fdebug-compilation-dir" "/home/test/playground" "-ferror-limit" "19"
 "-fmessage-length" "189" "-fobjc-runtime=gcc" "-fdiagnostics-show-option" "-fcolor-diagnostics"
 "-vectorize-loops" "-vectorize-slp" "-o" "hello.bc" "-x" "c" "hello.c" "-faddrsig"
```

You can see that `clang` is calling the internal `-cc1` interface. Though said to be modular, `-cc1` is not further calling the `opt` optimizer, but consuming the `-O2` option itself. In fact the LLVM IR codegen is built into the `-cc1` as a library. You can pass arguments to the codegen after the switch `-mllvm`, and using `-debug-pass` you can see the information of passes executed:

```sh
$ clang -mllvm -debug-pass=Arguments hello.c -O2 -c -o hello.bc -emit-llvm
Pass Arguments:  -tti -targetlibinfo -tbaa -scoped-noalias -assumption-cache-tracker
 -verify -ee-instrument -simplifycfg -domtree -sroa -early-cse -lower-expect
Pass Arguments:  -tti -targetlibinfo -tbaa -scoped-noalias -assumption-cache-tracker
 -profile-summary-info -forceattrs -inferattrs -ipsccp -called-value-propagation
 -globalopt -domtree -mem2reg -deadargelim -domtree -basicaa -aa -loops -lazy-branch-prob
 -lazy-block-freq -opt-remark-emitter -instcombine -simplifycfg -basiccg -globals-aa
 -prune-eh -inline -functionattrs -domtree -sroa -basicaa -aa -memoryssa -early-cse-memssa
 -basicaa -aa -lazy-value-info -jump-threading -correlated-propagation -simplifycfg
 -domtree -basicaa -aa -loops -lazy-branch-prob -lazy-block-freq -opt-remark-emitter
 -instcombine -libcalls-shrinkwrap -loops -branch-prob -block-freq -lazy-branch-prob
 -lazy-block-freq -opt-remark-emitter -pgo-memop-opt -basicaa -aa -loops -lazy-branch-prob
 <...>
Pass Arguments:  -targetlibinfo -domtree -loops -branch-prob -block-freq
Pass Arguments:  -targetlibinfo -domtree -loops -branch-prob -block-freq
Pass Arguments:  -tti
```

So you see... there are so many passes, and there are so many rounds of passes!

If you look into the source of `clang`, you'll see that all it is doing upon a `*.c` file is handled inside the `clang::ParseAST` function, defined at `lib/Parse/ParseAST.cpp:114`. Through `BackendConsumer::HandleTopLevelDecl`, each item defined in the source file is translated into IR, if `-emit-llvm` is used, and through `BackendConsumer::HandleTranslationUnit`, the IR is optimized and printed. Passes are directly called in `EmitAssemblyHelper::EmitAssembly`, and here we can see that different rounds of passes stand for per-function round, per-module round, and code-gen round, respectively. Thus you cannot append all the output by `-debug-pass` to an `opt` run -- it will not work.

We compared the IR generated under different optimization level before going through all those passes, and we found that they were the same -- no optimizations done in the frontend.

## Put off optimizations

Back to our topic. Regretfully we found that making clear about the pass running is not directly helpful, though interesting. Then what is that preventing the optimization? Maybe some metadata?

We examined the IR code again, and found out that there's a function attribute called `optnone`, like this:

```ll
define dso_local i32 @main() #0 {
    ...
}

attributes #0 = { noinline nounwind optnone uwtable ...
```

After manually removing this attribute, the optimization now works. If manual modification is not acceptable, `-Xclang -disable-O0-optnone` is to the rescue.

One more thing. If we are using a higher optimization level, like `-O2`, can we simply put off the execution of all the passes? Yeah, use `-Xclang -disable-llvm-passes`.

## Zero out the stack

Another interesting task is that, how to safely zero out the stack in clang. In C11, there is `memset_s`, a safe memory-set operation that will not get optimized. However, it is optional, and not included in clang. But there are other chances.

As `memset` is in fact an internal function in LLVM, it could be created using a dedicated interface, i.e. [`IRBuilder::CreateMemSet`](http://llvm.org/doxygen/classllvm_1_1IRBuilderBase.html#a7a87ff785ecd8547f1fb561b6016827e). There is a `volatile` argument, with no explanation here. Well, all memory-write operations in LLVM could be marked as `volatile` so that it will not get optimized, and `memset` is just another case.

Well, LLVM is great, offering so many possibilities. I just hope there could be better documentation :)
