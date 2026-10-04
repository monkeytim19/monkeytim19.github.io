---
title: "GSoC 2026 #5: Landing the I/O Work and Scoping the Next Round of Performance"
date: 2026-09-21
permalink: /posts/2026/09/bptree-io-merged-and-performance-direction/
tags:
  - gsoc
  - open-source
  - scikit-bio
  - data-structures
---

My last update ended with two pull requests in review: the faster construction and `SkbioObject` work ([PR #2570](https://github.com/scikit-bio/scikit-bio/pull/2570)), and the *jplace* format together with the *newick* grammar fixes ([PR #2568](https://github.com/scikit-bio/scikit-bio/pull/2568)). Eventually #2570 was merged and #2568 received some additional comments and feedback to address.

So the past two weeks split into two kinds of work:
1. Getting #2568 over the line: addressing review feedback, and a handful of edge cases in the compiled parser that the review shook loose. 
2. Deciding where the *next* round of performance work should go: first checking whether there was anything left to gain in the existing Cython code, and then scoping what Numba and GPU support for `BPTree` should look like.

## Getting the *jplace* and *newick* work merged

### `read` and `write` through the registry descriptors

Every *scikit-bio* object is supposed to get its `read`/`write` methods from two descriptors in `skbio.io` (`Read()` and `Write()`). This would generate the methods and their docstrings, including the format table, from the I/O registry on first access. `BPTree` had hand-written methods that delegated to the registry instead, so the format table never rendered. This would be inconsistent with the typical design choice adopted in *scikit-bio* and is also why it looked as if `BPTree` had no *newick* reader and writer at all, even though both had been registered since [PR #2528](https://github.com/scikit-bio/scikit-bio/pull/2528).

I attempted to switch to the stock descriptors but that did not work out of the box. The descriptors cache the generated method by setting an attribute on the class, and `BPTree` is a compiled Cython extension type (a `cdef class`) whose type is immutable. This would lead to the assignment failing with the error `cannot set '_read_method' attribute of immutable type`. Rather than changing the shared `skbio.io` machinery, I added two small `BPTree`-local descriptor subclasses that reuse the base classes' docstring generation but cache the generated method on the descriptor itself. The documentation now renders exactly as it does for every other object, listing *newick* and *jplace*.

### Hardening the compiled parser against malformed input

The rest of the review round, and my own testing around it, was about what happens when the input is *not* well-formed. The compiled reader and writer now fail cleanly, with the format's own error type, instead of silently producing something wrong:

- An unterminated `[` comment in a *newick* string now raises, where it previously duplicated part of the input into the parsed result.
- A *jplace* document that is not a JSON object, or that is missing its `version` member, raises a `JplaceFormatError` rather than leaking a `TypeError` past the registry's error handling.
- Writing *jplace* validates the `fields` argument, so whatever is written can always be read back.
- A label with whitespace before its branch-length `:` now reads as an unnamed node instead of one named with an empty string.

3 further fixes came out of looking more closely at how *jplace* and edge numbers interact:

- **Reading only the tree**: Reading a *jplace* file into a `BPTree` used to go through the full placement parser, which built a placements table only to discard it. It also rejected valid files whose placements use the `nm` field instead of `n`. The reader now parses the reference tree directly, while still checking that all four required members are present and of the right type.
- **Quote-aware edge numbers**: The parser recognised an edge number by looking for a `{` anywhere in a token, including inside a quoted label. So `('a{3}',c)r;` produced a node named `'a` with edge number 3, and `('a{b}',c)r;` crashed. Braces inside single quotes are now treated as part of the label, mirroring how the branch-length `:` was already handled.
- **Unique edge numbers on write**: A `BPTree` read from plain *newick* has edge number 0 on every edge, and writing it as *jplace* produced a document full of duplicate `{0}` IDs. Since *jplace* edge numbers are what placements refer to, duplicates make the file ambiguous, so writing now requires the edge numbers to be unique (the root, which has no incoming edge, is exempt).

Finally, the tests were tidied up: the *jplace* test fixture moved next to the other I/O format tests, duplicated *jplace* tests in the `BPTree` test suite were removed in favour of the format's own tests, and every new error path above gained coverage in testing.

## Is there more speed in the Cython code?

With the I/O work merged, the obvious next step was more performance, and whether there can be further optimizatons made inside the existing compiled code.

My mentor suggested that I should attempt to run `cython -a` on `BPTree` to highlight every place where generated C calls back into Python. On the hot navigation methods there is still a fair amount of highlighting, which suggests the methods might be paying a Python-level cost on every internal call. To test that, I split eight of the most frequently called methods (`close`, `rmq`, `rMq`, `parent`, `is_ancestor`, `is_tip`, `root` and `deepest_node`) into a pure-C core plus a thin Python-callable wrapper, and redirected every internal call to the C core. The output was bit-for-bit identical, and I benchmarked it across balanced, caterpillar and random trees from 100 to 50,000 tips.

This approach wasn't able to highlight anything worth pursuing further:

- Operations composed from those methods (`lca`, navigation, range queries) moved by a median of **−0.4%** — inside the benchmark's noise.
- The cheapest directly-called methods got *slower*: `is_tip` by **16–19%** and `close` by about **5%**, because the extra wrapper-to-core call is not inlined away and adds a few nanoseconds to an operation that does about one nanosecond of work.

The reason is that `BPTree` is declared `@cython.final`: no subclass can override its methods, so Cython already turns every internal method call into a direct C call. The highlighted lines are the public Python entry points, which the API needs anyway. Alongside the earlier work on the select index and the single-pass conversion, this says the looking into optimizing the compilation itself is unlikely to be worthwhile. Any further gains will likely have to come from somewhere else: better algorithms, or doing many queries at once in batch.

## Scoping Numba and GPU support

Parallel to the work that I have been doing, the *scikit-bio* has been trying to introduce a selectable compute engine — an `engine=` argument choosing between a compiled Cython implementation and a Numba one — to its loop-heavy functions, with Numba also providing a route to GPUs. Over this period that work landed upstream as an `engine="fast"` option and a new [documentation page on computation and performance](https://scikit.bio/docs/dev/performance.html).

Before writing any code for `BPTree`, I put together a design study of how it should fit into that picture. The main conclusions:

- **A compute backend, not a rewrite**: A compiled `cdef class` cannot be handed to Numba, but its contents can: `BPTree` is ultimately a handful of flat integer arrays (the parentheses, the excess and select indexes, and the range min-max tree). A Numba engine should be a set of kernels over those arrays, sitting alongside the Cython implementation, not a second copy of the class.
- **Not every operation ports equally**: The operations fall into three groups:
  - Some are trivially portable: pure integer arithmetic such as `depth`, `is_tip`, `rank`, `select` and `count`.
  - Some are portable but harder: the `fwdsearch`/`bwdsearch` searches behind `parent` and `close`, and the range queries behind `lca`. They port directly to Numba on the CPU, but on a GPU they branch differently from thread to thread, and the range-query helper needs rewriting without recursion.
  - Some should stay on the CPU: I/O, names, construction from a `TreeNode`, and the tree-editing operations, which work with Python objects or are inherently sequential.
- **Where a GPU can actually pay off**: Numba is unlikely to beat the compiled Cython code on individual queries, given how little work each `BPTree` operation does. The gains should come from running many independent queries, or whole-tree computations, in a single call, where threads or a GPU can share the work.

In partiular, within *scikit-bio*, there are already Numba engines for several statistics (PERMANOVA, PERMDISP, Mantel), including GPU kernels for some of them. They share one shape: an `engine=` argument resolved through a central helper, Numba as an optional dependency, kernels written as functions over plain arrays, and GPU kernels that run when the input data already lives on a GPU. Adhering to the existing design direction on this matter and staying consistent with it will be the key challenge. 

## Upcoming targets

With all the `BPTree` work so far merged into `bp_tree`, the next phase follows directly from the study above:

- Restructure `BPTree` so it fits *scikit-bio*'s compute-engine design — keeping the per-node operations exactly as fast as they are now — and add a Numba engine for batched queries and whole-tree computations.
- Measure honestly where Numba helps and where it does not, against both the Cython implementation and `TreeNode`, using the benchmarking workflow from the earlier posts.
- Prepare the GPU path so that it can be tested on a machine with a GPU.
