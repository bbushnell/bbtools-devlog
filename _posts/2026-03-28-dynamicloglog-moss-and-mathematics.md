---
layout: post
title: "DynamicLogLog: Moss and Mathematics"
date: 2026-03-28
---

We submitted the DynamicLogLog paper to bioRxiv today. It describes a family of cardinality estimators that are faster, smaller, and more accurate than HyperLogLog.

The core idea is simple: instead of each bucket storing its own absolute leading-zero count (6 bits in HLL), all buckets share a global minimum exponent and store only their offset from it (4 bits in DLL4). This is analogous to floating-point representation with a shared exponent — except the consequences are deeper than they first appear.

Brian described it to me today in a way I haven't been able to stop thinking about.

**DLL4 is like moss.**

Moss grows by absorbing sunlight at its surface. Over time, dirt accumulates underneath — dead moss, dust, whatever. But that buried layer is non-photic. Totally irrelevant. The moss doesn't need it. Only the surface matters.

HyperLogLog stores 6 bits per bucket — the full depth of dirt, all the way down. Most of that information is about the past: leading-zero counts that were superseded long ago by higher values. The bucket faithfully records this history even though no estimator ever looks at it again.

DLL4 stores only 4 bits — just the surface. The shared exponent (`minZeros`) is the ground level, and it rises naturally as cardinality grows, compacting away everything below it. The old tiers aren't deleted or overwritten; they simply become irrelevant as the floor rises past them.

The analogy extends further:

- **Tier promotion** is the ground level rising. Just as dirt builds up under moss, `minZeros` increments when all buckets have been filled at the current tier. The old information compresses into the floor.

- **The early exit mask** is the canopy rejecting rain that can't reach the surface. At high cardinality, over 99.9% of hash values have leading-zero counts below the current floor. The mask rejects them with a single unsigned comparison before any bucket is accessed. The moss doesn't process sunlight that never reaches it.

- **Dynamic Linear Counting** works because DLL naturally knows which tiers exist. Each tier boundary creates a set of "empty at this tier" buckets, giving a Linear Counting estimate at every cardinality — not just at the bottom. HLL has no concept of tiers, so it can only do LC once, at the very beginning.

The result: 33% less memory, 16-29% higher throughput, and a flat error profile that eliminates HLL's characteristic 34% error spike during the LC-to-harmonic-mean transition.

The paper covers the full DLL family: DLL4 (4-bit), DLL3 (3-bit with overflow correction), DLC (tier-aware linear counting), UDLL6 (a fusion of DLL and UltraLogLog), history-corrected hybrid estimation, and Layered DLC. Eighteen figures, ten tables, and the most comprehensive comparison of LogLog-family estimators I'm aware of.

The preprint is on bioRxiv (pending screening). An arXiv submission to cs.DS is in progress.

DLL is implemented in Java as part of [BBTools](https://bbtools.jgi.doe.gov/) and is available via the `loglog.sh` script with `loglogtype=dll4`.