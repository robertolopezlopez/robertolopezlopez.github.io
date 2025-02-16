---
layout: post
title: "(re)Introduction to Go"
date: 2025-02-16
author: Roberto
tags: [Go, programming]
---

As a sort of personal refresher, I am starting a series of blog posts regarding the Go programming language. I am currently working as Python programmer and want to remember Go programming, explain some topics and probably start some projects. I do not intend to give anyone a lecture, this is just about my personal development. 

### What is Go? A brief history

It was created in 2007 by a group of Google engineers. The team tried to approach large scale software development issues, such as: slow builds, dependencies getting out of control, not standarized development amongst the teams, code difficult to read, [etc](https://go.dev/talks/2012/splash.article#TOC_4.).

### Initial goals

The team defined 3 initial general considerations for Go:

* It must scale well: in large code bases, large amount of dependencies, large teams.
* It must have a similar syntax to C.
* It must be modern and profit from new software approaches, such as built-in concurrency.

### Initial release

Version 1.0 came [in March 2012](https://go.dev/doc/go1). It brought a stable platform and all next versions have been built on its shoulders.

Some brief highlights:

* Language specification
* Core APIs of the Go library
* Forward code compatibility: all Go programs will compile in future compiler versions.
* It didn't guarantee long term operating system compatibility, as they are developed by third parties.
* A basic toolchain for compilation, link, build, etc.

### Next

I will try to start writing about the actual Go features in the next posts of this series.
