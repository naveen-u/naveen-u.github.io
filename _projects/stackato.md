---
title: stackato
description: An LLVM pass for preventing data-oriented programming (DOP) attacks with runtime-randing stack padding.
language: C++
gh: https://github.com/naveen-u/stackato
order: 2
layout: project
preamble: Project for CSE 583 - Advanced Compilers (Fall 2024) at the University of Michigan under <a href="https://lingjia.engin.umich.edu/">Prof. Lingjia Tang</a>. Done in collaboration with <a href="https://serradane.github.io/serradane/">Serra Dane</a>, <a href="https://archit-bhatnagar.github.io/">Archit Bhatnagar</a>, <a href="https://public.websites.umich.edu/~akashpt/">Akash Poptani</a>, and <a href="https://erikchi.com/">Erik Chi</a>.
---

Applications in memory unsafe languages like C
and C++ are susceptible to non-control data attacks, where
an adversary exploits memory corruption vulnerabilities to
manipulate program execution without leaving the programdefined
control flow. Hu et al. (2016) demonstrated that
such Data-Oriented Programming (DOP) attacks can be
constructed to achieve Turing-completeness.
Prior works like *Smokestack* (Aga & Austin, 2019) have
demonstrated that defenses which randomize relative distances
between stack variables can thwart DOP attacks by
preventing attackers from being able to manipulate variables
of choice through memory corruption vulnerabilities and
detecting attacks or demoting them to denial-of-service.
In this project, we present `Stackato`, a novel defense
mechanism against DOP attacks. `Stackato` builds upon the
principles of Smokestack, implementing runtime random
padding and canary insertion to thwart and detect sophisticated
non-control data attacks.

We successfully demonstrate Stackato’s effectiveness
against real-world DOP attack scenarios using the `Min-DOP`
framework. Performance evaluations on SPEC CPU2017
and the LLVM Test Suite show minimal overhead, with
an average compile-time increase of 6.10% and a runtime
overhead of just 0.23% across LLVM Test Suite benchmarks.

The code is available on [github](https://github.com/naveen-u/stackato) and the full report is available below.

<div style="position: relative; width: 100%; padding-top: 129%; overflow: hidden;">
<iframe src="https://drive.google.com/file/d/1B1gN5R8mlZSb4GmDuGxbrp--sOO4QisJ/preview" style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border: 0;" allow="autoplay" frameborder="0" allowfullscreen></iframe>