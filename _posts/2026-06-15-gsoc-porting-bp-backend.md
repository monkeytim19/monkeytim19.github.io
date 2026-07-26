---
title: "GSoC 2026 #2: Porting Existing *iow* Implementation to as Main Backend to BP Tree"
date: 2026-06-20
permalink: /posts/2026/06/porting-bp-backend/
tags:
  - gsoc
  - open-source
  - scikit-bio
  - data-structures
---

During the first month, the main focus has been to port an existing implementation of the balanced-parantheses (BP) tree backend into *scikit-bio* and to establish that, given the environment of *scikit-bio*, the performance uplift from the BP tree still holds. 

## *improved-octo-waddle* BP backend

[improved-octo-waddle](https://github.com/biocore/improved-octo-waddle) (iow) is a repository that contains a working BP tree backend which was implemented previously years ago using Cython. In this initial phase of the project, I agreed with my mentors that, instead of implementing a BP tree backend from scratch, it would be more efficient to leverage this existing implementation and to handle all the supplementary matters first, such as the class design, I/O, performance benchmarking, etc. There are several aspects that were part of this consideration:

- **Correctness risk**: The bitwise primitives and memory management are subtle --- *iow*'s implementation is already correct and been tested at scale on trees up to billions of nodes. Re-deriving it in Numba would require much more care and caution to ensure that everything works from scratch.
- **Toolchain fit**: *scikit-bio* already ships Cython extensions and compiles C in its build, so a Cython port isn't a foreign dependency — it slots into machinery the project already maintains.

The current implementation from *iow*, although has support for the *scikit-bio* framework already and has some compatability with the classic `TreeNode` class for trees, it remains a separate package and not easily accessible by *scikit-bio* users. Thus, porting it into *scikit-bio* is not trivial as it requires ensuring that it is functional within the *scikit-bio* environment and can still deliver the same performance uplift as expected. 

## Porting the backend into a `skbio.tree.bp` subpackage

The primary task was to get *iow*'s code to actually live inside *scikit-bio*. Rather than scatter the modules, the initial decision was to have all the BP-related items in a new self-contained subpackage, `skbio.tree.bp`.  

I could have dropped these modules directly into `skbio.tree` alongside `TreeNode`, but I did not do so, in order to avoid interference with the functionality of `TreeNode`. Isolating everything under `skbio.tree.bp` and re-exporting a deliberately small set of names keeps the succinct-tree machinery — which is inherently low-level and full of C-adjacent detail — from leaking into *scikit-bio*'s public namespace. It also means the internals can be reorganized later without breaking anyone. 

The port itself involves several tightly-coupled Cython modules, each of which I brought over together with its `.pxd` header (the header is what lets other Cython modules call these routines at the C level, without going back through Python):

- **`_bp.pyx`**: the `mM` class that builds the range min-max tree, and the primary `BPTree` class that exposes `rank`, `select`, `excess`, `fwdsearch`, and `bwdsearch`, with every tree operation composed from those five primitives.
- **`_ba.pyx`**: a thin Cython wrapper over the vendored C bit array.
- **`_bp_binary_tree.pxd`**: header-only inline helpers for walking the *implicit* complete binary tree that sits over the parenthesis blocks (`bt_left_child`, `bt_parent`, `bt_node_from_left`, and so on).
- **`_bp_io.pyx`**: Newick parsing straight into the bit array, Newick writing, and jplace reading.

As I wired `BPTree` up through `skbio/tree/__init__.py`, the user can call it as they would call `TreeNode` through the `skbio.tree` submodule, exactly where they'd expect a tree type to live. Also, Since source files don't compile themselves. I added the BP modules to *scikit-bio*'s `setup.py`, appending them to the existing extension list so they flow through the same `cythonize(...)` step the rest of the library already uses. 

One point to note is the I/O functionality, which *scikit-bio* has its own way of handling it separately via its `skbio.io` submodule. However, the current porting will stick to the *iow* implementation. 

## Vendoring the *BitArray* C library

A major factor for why the BP tree is fast is due to its usage of **bit-arrays** and its efficient manipulation of such a data structure. The *iow* implementation delegates the backend for this data structure to an external open-source C library, [BitArray](https://github.com/noporpoise/BitArray). Here, I chose to **vendor** it and copy the C source directly into the subpackage, where I extracted the key files from the repository (`bit_array.c`, `bit_array.h`, `bit_macros.h`). 

The alternative would be to require every *scikit-bio* user to install *BitArray* on their system before the tree module would build or import. This would complicates the installation from the user's end and would require a separate C package installed. Vendoring the source means the C compiles as part of *scikit-bio*'s own build and the user never knows it's there. 

With the library being vendored and integrated into the system, one additional aspect that I had to note was to handle the liscensing as well and conform to the practice established within *scikit-bio* -- I recorded *BitArray*'s license alongside *scikit-bio*'s own at `licenses/bitarray.txt`.

## Adding the test suite

Finally, the tests. I added a `skbio/tree/tests/bp/` package covering the structure (`test_bp.py`), the I/O layer (`test_bp_io.py`), and conversions (`test_bp_conv.py`), together with a 200-tip jplace fixture as test data. They are essentially replica of the tests that existed already from the *iow* implementation.

It is noteworthy that many of `BPTree`'s methods are `cdef`. This means that they exist only at the C level and are simply unreachable from ordinary Python, meaning that a normal `pytest` file cannot call them. Rather than leave those internals untested (they're precisely the subtle, easy-to-get-wrong primitives the whole structure rests on), the existing tests were migrated into `test_bp_cy.pyx`, and registered it as its own compiled extension so it can reach in at the C level. 

## Building a basic benchmark

The latter part of the work was to build a simple and basic benchmark to establish whether the BP implementation, after being ported into *scikit-bio*, is functions correctly and delivers the performance benefits as expected. The benchmark was built comparing the memory performance of `BPTree` against `TreeNode` across a range of tree sizes, and also the per-operation runtime for the various primitive operations available to them (e.g. `depth`, `lca`): 

- **Identical topology**: Both structures are built from the *same* tree — one `TreeNode` converted into a `BPTree` — so I'm measuring the two representations of one tree, not two different trees that happen to be the same size.
- **Pre-sampled node ids**: I draw the random node ids *before* the timed loop starts, so the timer measures the operation and not the random-number generator.
- **Memory in a fresh subprocess:** I measure resident-set-size in a separate child process per structure and size, so the two object graphs never coexist in one interpreter, and so RSS captures BP's NumPy and Cython buffers — memory that `tracemalloc` would quietly miss because it only tracks Python-level allocations.

The results showed that:

- **Memory**: `BPTree` consistently uses **1.6–2.5×** less memory than `TreeNode` does (~115–193 bytes/node vs ~250–330).
- **Global queries** (`depth`, `lca`): BP stays roughly flat / `O(lg n)` as trees grow while `TreeNode` grows linearly — `depth` is ~14× faster at 100 tips, widening to ~21× at 10k tips.
- **Local queries** (`parent`): `TreeNode` is faster as this operation is constant in complexity over the number of nodes.

However, after initial feedback with my mentors, there might be flaws in the benchmarking and it is postponed for a review later after the integration of `BPTree` into *scikit-bio* is more refined with better documentation, API naming, etc.

