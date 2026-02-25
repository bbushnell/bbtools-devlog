---
layout: post
title: "Knowing Your Genome Size Before You Trust Your Assembly"
date: 2026-02-25
---

*(The k-mer hashing post is coming - this one jumped the queue because it came up in real work today.)*

Long-read assemblers like Flye are good at producing contigs from PacBio HiFi reads, but they often produce some junk alongside the real sequence. The junk typically shows up as short, low-depth contigs - assembly artifacts that don't represent actual genomic sequence.

The obvious filter is depth: discard any contig below some fraction of the average depth. This works in the simplest case. But it can breaks in some cases you actually care about.

**Why depth thresholds are fragile**

A collapsed repeat region produces a contig at 2× or 3× the main depth - it passes confidently, but represents multiple genomic copies mushed into one. A genuine low-copy plasmid sits near the chromosomal depth, or below it, and gets discarded as junk when it shouldn't be. A contaminating viral sequence, say from a phage infection of the sample, appears at extremely high depth and sails right through. The threshold's numerics depend on what's in the assembly, which is circular: you're using the assembly to evaluate the assembly.

There's a better reference point: the reads.

**K-mer frequency analysis**

When you sequence a genome at sufficient depth, most 31-mers will appear roughly as many times as your coverage depth. Sequence a haploid genome at 100×, and most genomic k-mers appear ~100 times in the read set. Sequencing errors make k-mers that appear once or twice. Repetitive sequences make k-mers that appear at integer multiples of the main depth.

A histogram of k-mer frequencies has a recognizable shape: a spike near zero from error k-mers, a main peak at the coverage depth, and a tail of repeat k-mers at higher frequencies. From the position and volume of that main peak, you can estimate genome size directly from the reads, before looking at the assembly at all.

BBTools does this with `kmercountexact` and the `peaks` flag.

**The workflow**

First, filter low-quality reads. For PacBio HiFi, reads with more than 0.5% error rate are degraded and should go.  Entropy-masking, in prokaryotes, will eliminate low-complexity sequencing artifacts that could otherwise show up as high-order peaks and bloat the estimate:

```
bbduk.sh in=reads.fq.gz out=filtered.fq.gz maq=23 entropymask minentropy=0.82
```

Then run k-mer counting with peak analysis:

```
kmercountexact.sh in=filtered.fq.gz peaks=peaks.txt -Xmx16g minprob=0.75
```

The output file contains, among other things:

```
#error_kmers        8588841
#genomic_kmers      5666151
#main_peak          139
#genome_size_in_peaks   5790811
#genome_size            5837986
#fold_coverage      139
```

**What these numbers mean**

`error_kmers` are the k-mers appearing at very low frequency, before the first real peak - sequencing errors producing unique k-mers that don't represent real genomic sequence.

`genomic_kmers` is everything else: k-mers that appear at depths consistent with real genome coverage.

`main_peak` is the coverage depth at the tallest peak - in this case, 139×.

`genome_size_in_peaks` and `genome_size` are two estimates that bracket the true genome size. The first sums peak volumes times their inferred copy numbers, only counting k-mers at the peaks themselves. The second integrates the entire histogram from the first peak onward, assigning copy numbers to every frequency bin including the valleys between peaks and beyond the highest peak. The first slightly underestimates; the second slightly overestimates. The best estimate of genome size falls between them.

**Using this to evaluate an assembly**

Once you have the bounds, compare them to your assembly. Sum the lengths of your high-depth contigs. If that sum falls inside `[genome_size_in_peaks, genome_size]`, those contigs explain the complete genome. Any remaining low-depth contigs aren't needed to account for the observed sequence.

In a case from today: two high-depth contigs summed to 5,796,338 bp. The bounds were 5,790,811 and 5,837,986. The sum sits inside the interval. The genome is those two contigs. All remaining low-depth contigs are likely artifacts or contamination.

This is more principled than a depth threshold because it's derived from the reads independently of the assembly. A depth threshold's definition of "suspicious" shifts based on what's in the assembly; these bounds don't.

**Practical notes**

`bbnorm.sh` and `khist.sh` can also generate k-mer histograms using Bloom filters instead of exact counting. Bloom filters use less memory and never run out, at the cost of some imprecision - useful when the dataset is too large to count exactly. For most bacterial genomes at typical HiFi coverage, `kmercountexact` fits comfortably in 16GB.

The `minprob=0.75` flag in the second command filters k-mers below a probability-of-correctness threshold, reducing the influence of low-quality k-mers on the histogram shape. This matters when reads aren't uniformly high quality, and complements read quality-filtering.

For diploid or polyploid genomes, the histogram shows multiple peaks at integer ratios, and the tool will attempt to estimate ploidy automatically. The `ploidy` parameter lets you specify it directly if you know it.
