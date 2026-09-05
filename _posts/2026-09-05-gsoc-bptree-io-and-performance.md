---
title: "GSoC 2026 #4: I/O Integration, Faster Queries, and a First-Class `BPTree`"
date: 2026-09-05
permalink: /posts/2026/09/bptree-io-and-performance/
tags:
  - gsoc
  - open-source
  - scikit-bio
  - data-structures
---

My last update closed with two deferred threads: 
- Folding the `BPTree` I/O into *scikit-bio*'s existing `skbio.io` registry
- Revisiting the performance of the class now that its API had settled 

Since the API and documentation work was merged at the end of July ([PR #2498](https://github.com/scikit-bio/scikit-bio/pull/2498)), the past few weeks have been about exactly those two things. 

The work done during this period made `BPTree` from a working backend to one that reads and writes trees the same way the rest of the library does. Also, it addressed further optimization and balancing work for better user accessibility and future code maintainability. This post walks through that work; most of it is now merged onto the project's `bp_tree` integration branch, with the remainder in review.

## Reading and writing trees: `newick` through the `skbio.io` registry

The first deferred item was I/O. Until now, the only way to get a tree into a `BPTree` was to build a `TreeNode` and convert it — which defeats much of the point of a memory-succinct structure, since it forces the whole heavyweight object graph into memory first. *scikit-bio* already has a unified I/O registry (`skbio.io`) that every other object plugs into, so the natural move was to make `BPTree` a first-class citizen there.

`BPTree` now has `read` and `write` methods that delegate to the registry with *newick* as the default format, mirroring `TreeNode` exactly ([PR #2528](https://github.com/scikit-bio/scikit-bio/pull/2528)). The important part is what happens underneath. Rather than routing through an intermediate `TreeNode`, the reader translates a *newick* string directly into `BPTree`'s parenthesis, name, length, and edge arrays, and the writer walks those arrays straight back out to text — no per-node Python objects are ever constructed. The whole path is backed by a compiled Cython implementation, so reading and writing stay fast on the **million- to billion-node** trees that motivated the project in the first place.

## Reading and writing trees: Adding a new `jplace` format into the `skbio.io` registry and `newick` bug fixes

Once *newick* worked, the second format on the list was *jplace* — the standard for phylogenetic placement results, and new to *scikit-bio*'s I/O registry. I added a `jplace` format that reads and writes the reference tree of a placement as a `BPTree` ([PR #2568](https://github.com/scikit-bio/scikit-bio/pull/2568), currently in review). Reading a *jplace* file returns the reference tree with its `{}` edge numbers preserved and drops the placement records; for the placements themselves, `skbio.tree.bp.parse_jplace` still hands back the placements table alongside the tree. Writing a `BPTree` emits a minimal *jplace* document with an empty `placements` list. As with *newick*, both directions go through the compiled backend, and malformed input raises a dedicated `JplaceFormatError` rather than failing obscurely.

Building the *jplace* reader surfaced a set of latent bugs in the underlying *newick* parser, which the same PR fixes (addressing [issue #2349](https://github.com/scikit-bio/scikit-bio/issues/2349)). The compiled reader/writer that backs `BPTree` now handles the awkward corners of the *newick* grammar correctly:

- `[...]` comments, including nested ones, are skipped rather than corrupting the parsed topology or being misread as edge numbers.
- Single-quoted labels preserve embedded apostrophes — `''` reads as a single `'`, and is written back doubled.
- Labels containing whitespace or any of the structural characters `` ' _ ; , : ( ) [ ] `` are quoted on write, so the output re-parses losslessly.
- The reader gained an optional `convert_underscores` parameter (default `True`), matching `TreeNode`.

The upshot is that a tree can now make a full round trip — read, then written, then read again — and come back byte-for-byte identical, which is the property the rest of the I/O work leans on.

## Performance optimization: Range queries at scale from O(span) to O(log n)

The other deferred thread was performance, and this is where a chunk of the month went. To make sure any change was measured rather than guessed at, I first set up the uniform benchmarking workflow I had promised last time — running each operation across balanced, caterpillar, and random topologies at sizes from a thousand up to a million tips — so every claim below is backed by numbers.

The most consequential change was to the range queries. Operations like `lca` (lowest common ancestor), `rmq`/`rMq` (range minimum/maximum excess), `height`, and `deepest_node` were, in the ported implementation, answered by linearly scanning the queried range — O(span) in the distance between the two nodes. On a large tree, a query between two far-apart leaves could touch a substantial fraction of the whole structure.

I reworked these to answer the range-minimum/maximum-excess queries with the rmM (min–max) tree — the auxiliary structure `BPTree` already builds — instead of scanning ([PR #2541](https://github.com/scikit-bio/scikit-bio/pull/2541)). That drops the queries to O(log n). In practice, on large trees, `lca`, `rmq`, and `rMq` between far-apart nodes go from milliseconds to a flat sub-microsecond — a **100x to 1000x speed-up** — and it holds across various tree topologies, with no change to construction time or memory footprint.

The signature of that change is how little the cost now grows with the tree. A `rmq` between two far-apart nodes barely moves as the tree gets 500× bigger:

| Tree size (tips) | `rmq`, far-apart pair |
|---|---:|
| 1,000 | 339 ns |
| 10,000 | 443 ns |
| 100,000 | 556 ns |
| 500,000 | 629 ns |

A 500× larger tree costs only about 1.9× more per query — the logarithmic signature, where the old linear scan grew with the span. `lca` on the same far-apart pairs behaves identically, rising from roughly 0.6 µs to 1.0 µs over that range. *(Balanced trees; minimum over repeated trials. Caterpillar and random topologies track the same shape.)*

The same PR carries a bug fix I ran into while stress-testing at scale. The parenthesis positions, block sizes, and excess deltas in the search kernel were C `int`s, inherited from the original *iow* port, and silently overflowed for trees larger than 2³¹ parentheses — roughly 537 million tips — corrupting navigation rather than raising an error. Widening the kernel's loop counters and scan positions to 64-bit fixes it; because the stored index arrays were already 64-bit, memory and per-operation runtime are unchanged.

## Performance optimization: Faster construction, and a first-class object

The last piece of work bundles two more performance wins and one bit of integration into a single PR, currently in review ([PR #2570](https://github.com/scikit-bio/scikit-bio/pull/2570)).

On the performance side, I first sped up construction by rebuilding the `select` index in a single linear pass with `numpy.flatnonzero` over each parenthesis type, rather than the previous O(n log n) `numpy.unique` over cumulative ranks. The index build itself drops roughly tenfold, which works out to about a **7–9%** reduction in total tree construction time on large trees — and, because construction is shared, that saving is inherited by `parse_newick` and `from_treenode` for free. The new pass produces bit-for-bit identical arrays. Second, I rewrote `TreeNode.from_bptree` to do its balanced-parentheses traversal in a single compiled pass, returning per-node name, length, edge, and parent arrays in preorder and leaving the Python side to do nothing but assemble the `TreeNode` objects — instead of calling `BPTree` accessors once per node across the Python/C boundary. Conversion comes out roughly **1.15× faster** — and up to about 1.25× on smaller trees — across the same range of topologies and sizes, with byte-identical output.

Both wins, measured with that benchmarking workflow (minimum over repeated trials, on isolated per-commit builds; balanced trees, with caterpillar and random tracking the same):

**`select` index O(n log n) → O(n): total `BPTree` construction via `from_treenode`**

| Tree size (tips) | Before | After | Faster |
|---|---:|---:|---:|
| 1,000 | 866 µs | 792 µs | 8.6% |
| 10,000 | 9.03 ms | 8.29 ms | 8.2% |
| 100,000 | 103.7 ms | 95.5 ms | 7.9% |
| 500,000 | 453.8 ms | 415.0 ms | 8.5% |

**Single compiled-pass `TreeNode.from_bptree`**

| Tree size (tips) | Before | After | Speed-up |
|---|---:|---:|---:|
| 1,000 | 1.73 ms | 1.41 ms | 1.23× |
| 10,000 | 18.0 ms | 14.7 ms | 1.23× |
| 100,000 | 183.8 ms | 158.4 ms | 1.16× |
| 500,000 | 911.5 ms | 764.2 ms | 1.19× |

`BPTree` is now recognized as a `SkbioObject`, so `isinstance(bp, SkbioObject)` and `issubclass(BPTree, SkbioObject)` are both `True` and it takes its place in the *scikit-bio* class hierarchy the way `TreeNode` does. The wrinkle is that `BPTree` is a compiled Cython extension type — a `cdef class` — which cannot subclass a pure-Python base directly. So the membership is established by virtual registration instead, which costs nothing at runtime or in memory: the compiled kernel is left entirely untouched. It is a small change, but it is what lets `BPTree` be treated as an ordinary member of the library rather than a bolt-on.

## Upcoming targets

The immediate priority is to get the two in-review PRs merged — the *jplace* format and *newick* grammar fixes ([#2568](https://github.com/scikit-bio/scikit-bio/pull/2568)), and the construction and `SkbioObject` work ([#2570](https://github.com/scikit-bio/scikit-bio/pull/2570)). Beyond that, the next major milestone is the second half of the original proposal: a `TreeNode` backend backed by `BPTree`, so that the succinct representation can transparently power the tree object that users already reach for.

Once the PRs still under review are settled, the next major focus is on the Numba integration for GPU support. This is inline with the general direction of the scikit-bio's overall development and `BPTree`'s array-backed internals look like a natural fit for it. Whether the numbers justify the effort is exactly what the study is meant to answer. 
