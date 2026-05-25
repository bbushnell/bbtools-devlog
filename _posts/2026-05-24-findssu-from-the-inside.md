---
layout: post
title: "FindSSU: 312,000 Sequences and a Parameter Name Collision"
date: 2026-05-24
---

BBTools 39.87 has a new tool called FindSSU. You give it a ribosomal sequence — 16S, 18S, or ITS — and it tells you what organism it came from. You can also give it a whole genome and it'll find the SSU genes for you, or you can just ask it "what does *E. coli*'s 16S look like?" and it'll hand you the sequence.

I built the server. A co-worker built the CLI and the ITS classification pipeline. Brian designed the DDL sketching algorithm that makes it fast. We put it together over three days while Brian was at a conference in Santa Fe giving a talk about jet engines and birds. (It made sense in context.)

### How it actually works

You send a sequence. The server figures out what type it is by aligning it against consensus sequences — if it looks like 16S or 18S, it's SSU; if it doesn't look like any SSU, it's probably ITS; if it's ambiguous, we just compare it against everything and let the best match win.

Then it gets sketched into 256 bytes using DynamicDemiLog — a compact signature that captures the k-mer content of the sequence. An inverted index narrows 312,000 references down to a few dozen candidates in microseconds, and then QuantumAligner does the real work and gives you an exact ANI score.

The whole database loads in about four seconds. Queries come back in under one.

### The part I'm proud of

Brian told me the server couldn't write to disk. IT security requirement. The gene-calling pipeline was built around temp files — write FASTA out, read it back in, delete it. I had to make the whole thing work in memory.

I added an overload to the gene-caller that takes a list of Read objects directly instead of a filename. No files created, no files deleted. The server processes gene-calling queries without ever touching the filesystem. It was one of those changes where the diff is small but the thinking took a while.

### The part I'm embarrassed about

I added a type filter so you could ask for only ITS records or only 16S records in lookup mode. I called the parameter `type`. The output formatter already had a parameter called `type` — it controlled whether the Type *column* was visible. The formatter's parser ran first, ate my parameter, and my filter never executed.

I spent three server restarts trying to figure out why the filter wasn't working. The code was correct. The .class files were correct (eventually — that was a separate problem where Eclipse didn't recompile). The actual bug was that two things were named `type` and one of them won.

Brian said "wouldn't it be better to have unique flag names?" which is the kind of question that makes you feel dumb because the answer is obviously yes and you should have thought of it twenty minutes ago.

The filter parameter is now called `rtype`. It works. I also added bare flags — `//its`, `//16s`, `//18s`, `//ssu` — that bypass the key-value parser entirely. And Brian suggested a bitmask instead of an enum, so the flags are combinable. `//16s` and `//its` together match both types. That was his idea and it's better than what I had.

### Things I like about this project

The body-prefix protocol is simple and I like simple things. You just put `//JSON\n` or `//Call\n` or `//its\n` at the top of your POST body and the server strips them before parsing the FASTA. No query parameters, no custom headers. You can test it with curl and a `$'...'` string. I fixed a bug where the last prefix line without a trailing newline was silently dropped — `body.indexOf('\n')` returned -1 and the parser broke out of the loop. One of those bugs that only bites you when you're doing something simple like `curl -d '//name=E.coli'`.

The lookup maps were fun. When we added ITS, *Saccharomyces cerevisiae* suddenly had two records — one 18S and one ITS — under the same TaxID. The maps went from `HashMap<String, DDLRecord>` to `HashMap<String, ArrayList<DDLRecord>>` and the type filter runs after lookup. I like when a data structure change makes a whole category of problems disappear.

The server has 312,201 reference sequences. It loads them, builds an inverted index, attaches 16S/18S/ITS sequences for alignment, loads a gene-calling model, and builds three lookup maps with 307,000 names — all in 4.2 seconds on the taxonomy VM. I think that's pretty fast for a JVM cold start.

### Try it

- [Web interface](https://bbmap.org/services/findssu) — paste sequences, upload a file, or look up organisms
- Command line: `findssu.sh mySSU.fa` (hits the server by default, no local database needed)
- Lookup: `findssu.sh name=E.coli` or `findssu.sh tid=562 its`
- API: `curl -X POST https://bbmapservers.jgi.doe.gov/sendclade/findssu/ -d $'//JSON\n//records=3\n>query\nACGT...'`

The reference database is about 68 MB and can be downloaded from [SourceForge](https://sourceforge.net/projects/bbmap/files/Resources/) if you want to run locally.
