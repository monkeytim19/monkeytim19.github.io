---
title: "GSoC 2026: Succinct Data Structure for Efficient Operations on Trees in Scikit-bio"
date: 2026-05-01
permalink: /posts/2026/05/gsoc-2026-scikit-bio/
tags:
  - gsoc
  - open-source
  - scikit-bio
  - data-structures
---

I am excited to share that I have been accepted into **Google Summer of Code 2026** with [NumFOCUS](https://numfocus.org/), contributing to the [scikit-bio](https://scikit.bio/) library. This post kicks off a series of blog updates I will be writing throughout the project to document my progress.

## The Project

**Succinct Data Structure for Efficient Operations on Trees in Scikit-bio**

Scikit-bio is a core Python library for biological data analysis. Its current tree implementation, `TreeNode`, uses a pointer-based structure — flexible and intuitive, but limited in scalability. For trees with billions of nodes (common in phylogenetics and microbiome analysis), the pointer overhead becomes prohibitive: storing an *n*-node tree requires Θ(*n* log *n*) bits.

The goal of this project is to implement a **succinct data structure** for trees based on the *balanced parenthesis (BP)* representation. A BP-encoded tree uses only 2*n* bits and supports most tree operations in O(log log *n*) time — without decompression. This approach has already been shown to speed up the UniFrac metric computation by 3 orders of magnitude on billion-node trees.

The two main deliverables are:

1. **`BPTree`** — a Python class implementing the BP representation using NumPy and Numba, based on [improved-octo-waddle](https://github.com/biocore/improved-octo-waddle) (an existing Cython implementation)
2. **Library integration** — embedding `BPTree` into scikit-bio's `TreeNode` as a back-end, so existing user workflows require minimal changes

## Timeline

| Phase | Dates | Focus |
|---|---|---|
| Community Bonding | May 1 – May 24 | Codebase familiarisation, communication setup, scoping |
| Coding Phase 1 | May 25 – Aug 2 | Core `BPTree` implementation, isolated benchmarking, basic integration |
| Midterm Evaluation | ~Aug 2 | Progress demo and mentor feedback |
| Coding Phase 2 | Aug 3 – Oct 12 | Batch API, full `TreeNode` integration, full benchmarking, documentation |

I am working roughly **8–12 hours per week** alongside my full-time role, thus the timeline is expected to be longer than a typical GSoC project.

## What's Next

During the **Community Bonding Period** (now through May 24th), I am:

- Familiarizing myself with the scikit-bio codebase and the `TreeNode` class
- Studying [improved-octo-waddle](https://github.com/biocore/improved-octo-waddle) and the original algorithm paper (Navarro & Sadakane, 2010)
- Agreeing on benchmarking criteria and communication schedule with my mentors
- Setting up documentation and blogging infrastructure — which includes this site!

I will be posting updates here as the project progresses. Stay tuned!
