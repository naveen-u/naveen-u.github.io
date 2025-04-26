---
title: delta-rs-mt
description: A fork of delta-rs that supports multi-table transactions in Delta Lake without an external commit coordinator.
language: Rust
gh: https://github.com/naveen-u/delta-rs-mt
order: 1
layout: project
preamble: Project for CSE 584 - Advanced Database Systems (Winter 2025) at the University of Michigan under <a href="https://web.eecs.umich.edu/~linmacse/">Prof. Lin Ma</a>. Done in collaboration with <a href="https://in.linkedin.com/in/anirudh-bhaskar-086867245">Anirudh Bhaskar</a>, <a href="https://www.linkedin.com/in/divyamthinks">Divyam Sharma</a>, and <a href="https://www.linkedin.com/in/nilaygautam">Nilay Gautam</a>.
---

Delta Lake provides ACID guarantees and efficient
metadata management for single-table workloads on
cloud object stores, but lacks native support for atomic
operations spanning multiple tables. In this work, we
introduce *transaction groups* (or T-Groups), a protocol
extension that enables multi-table transactions using
only object-store primitives — no external coordinators
required. We implement our design in `delta-rs-mt`, a fork
of the `delta-rs` library, and evaluate on
TPC-DS benchmarks. We find that single-table operations incur
<0.25s of write overhead and reads show <3%
latency increase; multi-table writes exhibit predictable
log-contention. Our approach seamlessly integrates with
existing Delta Lake features, making robust multi-table
ACID transactions practical for large-scale analytics
without relying on external coordination services.

The code is available on [github](https://github.com/naveen-u/delta-rs-mt) and the full report is available below.

<div style="position: relative; width: 100%; padding-top: 129.4%; overflow: hidden;">
<iframe src="https://drive.google.com/file/d/1JVveXWdmlz2AyBtxI7DJRhxeRS0dIwzI/preview" style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border: 0;" allow="autoplay" frameborder="0" allowfullscreen></iframe>