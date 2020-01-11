---
title: "Clang: Don't Touch My Code!"
date: 2020-01-11T16:31:37+08:00
draft: true
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

Before we get into the detail, let's look at what `clang` is doing internally. The `-###` argument may be helpful.

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
 -lazy-block-freq -opt-remark-emitter -tailcallelim -simplifycfg -reassociate -domtree
 -loops -loop-simplify -lcssa-verification -lcssa -basicaa -aa -scalar-evolution
 -loop-rotate -licm -loop-unswitch -simplifycfg -domtree -basicaa -aa -loops
 -lazy-branch-prob -lazy-block-freq -opt-remark-emitter -instcombine -loop-simplify
 -lcssa-verification -lcssa -scalar-evolution -indvars -loop-idiom -loop-deletion
 -loop-unroll -mldst-motion -phi-values -basicaa -aa -memdep -lazy-branch-prob
 -lazy-block-freq -opt-remark-emitter -gvn -phi-values -basicaa -aa -memdep -memcpyopt
 -sccp -demanded-bits -bdce -basicaa -aa -loops -lazy-branch-prob -lazy-block-freq
 -opt-remark-emitter -instcombine -lazy-value-info -jump-threading -correlated-propagation
 -basicaa -aa -phi-values -memdep -dse -loops -loop-simplify -lcssa-verification -lcssa
 -basicaa -aa -scalar-evolution -licm -postdomtree -adce -simplifycfg -domtree -basicaa
 -aa -loops -lazy-branch-prob -lazy-block-freq -opt-remark-emitter -instcombine -barrier
 -elim-avail-extern -basiccg -rpo-functionattrs -globalopt -globaldce -basiccg -globals-aa
 -float2int -domtree -loops -loop-simplify -lcssa-verification -lcssa -basicaa -aa
 -scalar-evolution -loop-rotate -loop-accesses -lazy-branch-prob -lazy-block-freq
 -opt-remark-emitter -loop-distribute -branch-prob -block-freq -scalar-evolution
 -basicaa -aa -loop-accesses -demanded-bits -lazy-branch-prob -lazy-block-freq
 -opt-remark-emitter -loop-vectorize -loop-simplify -scalar-evolution -aa -loop-accesses
 -loop-load-elim -basicaa -aa -lazy-branch-prob -lazy-block-freq -opt-remark-emitter
 -instcombine -simplifycfg -domtree -loops -scalar-evolution -basicaa -aa -demanded-bits
 -lazy-branch-prob -lazy-block-freq -opt-remark-emitter -slp-vectorizer -opt-remark-emitter
 -instcombine -loop-simplify -lcssa-verification -lcssa -scalar-evolution -loop-unroll
 -lazy-branch-prob -lazy-block-freq -opt-remark-emitter -instcombine -loop-simplify
 -lcssa-verification -lcssa -scalar-evolution -licm -alignment-from-assumptions
 -strip-dead-prototypes -globaldce -constmerge -domtree -loops -branch-prob -block-freq
 -loop-simplify -lcssa-verification -lcssa -basicaa -aa -scalar-evolution -branch-prob
 -block-freq -loop-sink -lazy-branch-prob -lazy-block-freq -opt-remark-emitter -instsimplify
 -div-rem-pairs -simplifycfg -write-bitcode
Pass Arguments:  -targetlibinfo -domtree -loops -branch-prob -block-freq
Pass Arguments:  -targetlibinfo -domtree -loops -branch-prob -block-freq
Pass Arguments:  -tti
```

So you see... there are so many passes, and there are so many rounds of passes!

## Put off optimizations

## Zero out the stack