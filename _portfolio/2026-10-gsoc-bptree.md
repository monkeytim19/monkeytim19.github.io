---
title: "Google Summer of Code 2026: Succinct Balanced-Parentheses Tree for scikit-bio"
excerpt: "Designed and merged BPTree, a succinct tree structure, into scikit-bio (NumFOCUS): half the memory of TreeNode, 5.6–12× faster Newick parsing, and CPU/GPU engines for batched queries."
collection: portfolio
date: 2026-10-01
---

**Duration:** May 2026 – Oct 2026  
**Final report:** [gist.github.com/monkeytim19/22475ea04a9b67b53b0cdb4a19537473](https://gist.github.com/monkeytim19/22475ea04a9b67b53b0cdb4a19537473)  
**Mentors:** Dr. Qiyun Zhu and James Morton  
**Blog series:** [GSoC posts](/tags/#gsoc)

Google Summer of Code 2026 project with NumFOCUS, contributing to the [scikit-bio](https://scikit.bio/) library.

- Designed and merged BPTree, a succinct balanced-parentheses tree, into scikit-bio over 13 reviewed PRs (+10.5k lines)
- Halved memory relative to the pointer-based TreeNode (3.1 GB vs 6.1 GB at 10M tips), read Newick 5.6–12× faster, and pruned 10M-tip trees in seconds where TreeNode ran out of memory
- Implemented O(log n) range queries with a range min-max tree, making LCA queries 7–8× faster than TreeNode on random 1M-tip trees
- Built interchangeable Cython (OpenMP) and Numba engines plus CUDA/ROCm GPU kernels for batched queries, cutting LCA cost from 10 µs per query (TreeNode) to 2.3 ns on GPU
- Built a benchmark suite across four tree topologies and 100 to 10M tips, and fixed latent bugs including a 32-bit overflow and a Newick reader buffer overflow

**Tools:** Python, Cython, OpenMP, Numba, CUDA, NumPy, CuPy, PyTorch
