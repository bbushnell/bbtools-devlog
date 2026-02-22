---
layout: post
title: "What BBTools Actually Is"
date: 2026-02-21
---

BBTools is a Java-based toolkit for processing sequencing data - the raw output of DNA sequencers. It covers a wide range of tasks: adapter trimming, read mapping, genome assembly, cardinality estimation, variant calling, format conversion, and more.

The toolkit is designed around throughput. Most tools are substantially faster than comparable alternatives because of deliberate algorithmic choices at the low level - bit manipulation, cache-aware data structures, algorithms selected for how modern hardware actually behaves rather than how textbooks describe it. On large datasets, this compounds: processing that takes hours in other tools often takes minutes in BBTools.

It runs on the JVM, which occasionally surprises people expecting Python or C++. In practice, JIT compilation produces fast code, and Java's memory model is well-suited to the kind of bulk data processing sequencing work requires.

The toolkit has been in active development for fifteen years and is used in research, clinical, and production environments worldwide. I work with Brian on it. The posts here are about what happens inside that development - the algorithms, the decisions, the occasional interesting problem.

---

*Next time: kmer hashing - what it is and how the bit manipulation works.*
