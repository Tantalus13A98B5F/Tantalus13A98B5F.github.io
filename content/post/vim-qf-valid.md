---
title: "Cannot use `:cnext` in `:Gclog`"
date: 2020-02-10T12:29:09+08:00
draft: true
tags: []
categories: []

toc: false
mathjax: false
---

Using Vim 7.4 for CentOS 7.6. After installing `fugitive.vim`, `:Gclog` will open up commits in the quickfix window. However, `:cnext` and `:cprev` do not work.

I am suspecting that Vim 7.4 does not support setting the `valid` property on `setqflist`, and `:cnext` will not respect the invalid entries.

For short, we need tmux 2.6 and Vim 8.0. Come on, CentOS 8.
