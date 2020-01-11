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

Recently I am working on LLVM for program analysis and instrumentation, and I think it's time to write about how we are dealing with unwanted optimizations. Here we start.

## How is Clang calling optmizations?

## Put off optimizations

## Zero out the stack