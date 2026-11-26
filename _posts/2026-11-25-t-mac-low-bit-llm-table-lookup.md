---
title: "T-MAC: Low-Bit LLMs on CPUs via Table Lookup"
date: 2026-11-26 12:00:00 +0530
categories: [Paper Notes, LLM Inference]
tags: [quantization, llm, edge, cpu, lookup-table, eurosys]
math: false
---

**Paper:** *T-MAC: CPU Renaissance via Table Lookup for Low-Bit LLM Deployment on Edge* (Wei et al., EuroSys 2025)
· [arXiv](https://arxiv.org/abs/2407.00088) · [Code](https://github.com/microsoft/T-MAC)

## Problem they solved

To run big language models on small devices (phones, laptops, Raspberry Pi), the model's numbers are shrunk to very few bits (1-4 bits) to save memory. But chips can't directly multiply a tiny low-bit number with a normal high-precision number. Current tools like llama.cpp first convert the tiny numbers back to a bigger format, which wastes time and cancels most of the benefit. Using fewer bits barely made things faster.

## How they solved it

- **Break the math into single bits.** A multiplication can be split into one-bit pieces, each handled separately and then added up. This works for any bit size, so one method covers everything.
- **Replace calculating with looking up.** One-bit patterns have only a few possible combinations, so all possible answers are pre-calculated once and stored in a small table (a cheat sheet). The chip then looks up the answer and adds. No multiplication, no converting.
- **Keep the cheat sheet in the fastest memory.** The data is arranged so the table stays inside the CPU's registers, and the table is shrunk with two tricks: mirror symmetry (half the entries are negatives of the others) and storing table values in smaller numbers.

## Result

Up to about 4x faster than llama.cpp and 60-70% less energy on ordinary CPUs. It reaches roughly 11 tokens/sec on a Raspberry Pi 5, and in some cases matches or beats GPUs and NPUs on the same device, so no special hardware is needed.

---

## My experiment: did T-MAC speed up 4-bit decode on my machine?

**Short answer:** No. T-MAC's math was about 2x faster than llama.cpp's, but tokens did not come out any faster. My machine is limited by how fast weights arrive from RAM, not by how fast it calculates.

### The simple picture

Think of the CPU as a chef and RAM as a delivery truck. To produce one token, the truck must bring the entire model (about **4.4 GB**). The chef uses each ingredient once and throws it away. A faster chef doesn't help when the truck is the slowest part.

- RAM delivers about **35 GB/s**, so one token needs 4.4 / 35 = about **125 ms**.
- I measured about **133 ms** per token, so memory is the limit.
- One T-MAC core computes about 3x faster than RAM can feed it, and llama.cpp needs only 2 cores to keep up.

### Why T-MAC didn't help here

- **Same bytes:** at 4 bits it reads the same amount of data (my layout was even 2.8% larger).
- **Strong baseline:** llama.cpp already does integer math directly on the 4-bit values, so there was little left to remove.
- **Wrong hardware:** the paper's big wins come from weak chips (Raspberry Pi, Jetson) at 1-2 bits. Its own 4-bit result on M2-Ultra was only about 1.1x. My machine has 16 strong cores but only about 35 GB/s of memory bandwidth, so the gap is extreme.
- **Where it did win:** with 1 thread, where compute is the limit, T-MAC was **1.71x faster**.

### What would actually help

T-MAC only pays off when computation, not memory, is the bottleneck. To get there:

1. **Batch many users together.** Each weight is read once and reused for all users, so compute matters again. This is the best fit.
2. **Split the cores:** a few for decoding, the rest for prefill (reading the prompt).
3. **Use fewer bits** (2-3 bit weights) so less data is read, at some cost in accuracy.
4. **Speculative decoding:** a small model guesses tokens and the big one checks several at once.

Tuning the single-user 4-bit kernel further is not worth the effort.

### Takeaway

Making the math cheaper only helps if the math is the slow part. For single-user decode on my machine it isn't, so the next step is batched decoding with separate core groups for prefill and decode.
