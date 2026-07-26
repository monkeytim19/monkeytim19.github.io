---
title: "GSoC 2026 #3: Refining the `BPTree` API and Documentation"
date: 2026-07-20
permalink: /posts/2026/07/integrating-bptree/
tags:
  - gsoc
  - open-source
  - scikit-bio
  - data-structures
---

After the initial focus on the porting of the *iow* implementation of the BP Tree into *scikit-bio* as `BPTree` and ensuring that it works within the *scikit-bio* environment, the next target is to add in the relevant documentation for the class and to refine the design of the API that would maximize the user-friendiness and consistency with the rest of the library. All the work so far all came together as [PR #2498](https://github.com/scikit-bio/scikit-bio/pull/2498). 

## Harmonizing the API with `TreeNode`

Although the main class `BPTree` was made available within *scikit-bio* as the primary API for the user to access the BP tree functionalities, it has inconsistencies with other parts of the package. For instance, relative to the existing `TreeNode` API, many of the `BPTree` methods, such as `fchild`, `nsibling`, `isleaf`, `isancestor`, `preorderselect`, have overloaded or misleading meaning to the user. Given that the former has been (and will likely remain as) the default way that the users of the library interacts with its functionalities for trees, it is crucial that the `BPTree` namings respect it for user-friendiness and to avoid confusion.  

As such, I adapated the existing namings and attributes within `BPTree` to the convention within `TreeNode`:

- The core attribute `B` → `data` (self-describing beats single-letter).
- Some of the methods renamed: 
  - `isleaf`→`is_tip`, 
  - `isancestor`→`is_ancestor`, 
  - `preorderselect`→`preorder_select`

The items above relate to changes that were relatively straightforward as the semantics and logic of the methods achieve the same objective. There are other parts of the `BPTree` API that require more care and modification due to the absence of an equivalent correspondence within the `TreeNode` API. This includes `BPTree` methods like `preorder_rank`, `postorder_rank`, `fchild`, `nsibling`, etc. Naturally, this is a consequence of the inherent difference that `BPTree` is supposed to represent be a static tree, whereas `TreeNode` can accomodate for dynamic updates in the tree. As such, I had to modify the naming such that it respects the convention in `BPTree` whilst still being correct about the actual operation of the methods themselves.

## Documentation and deciding the public surface

A significant part of the work that followed focused on the documentation side of things. The prior work within *iow* already included much of the docstrings, type-setting, and other forms documentations for to the methods and classes that are being ported into *scikit-bio*. However, a key chore handle was to ensure that the documentation all adhered to the standard of *scikit-bio*. Alongside this, another key consideration was to decide on the actual public surface of the `BPTree` APIs. The final organization was to keep the public surface and documentation as similar to the structure for `BPTree` as much as possible. Namely, I organized the public API into five documented groups:

- **IO:** `read`, `write`
- **Navigation:** `root`, `parent`, `first_child`, `last_child`, `next_sibling`, `previous_sibling`, `lca`, `is_ancestor`, `deepest_node`
- **Traversal:** `preorder_rank`, `postorder_rank`, `preorder_select`, `postorder_select`, `level_ancestor`, `level_next`
- **Analysis:** `is_tip`, `depth`, `height`, `count`
- **Manipulation:** `shear`, `collapse`
- **Conversion:** `from_treenode`, `to_array`


## Adjusting based on mentor feedback

After organizing the initial idea for the documentation and structure of the API into a PR draft, I received comments from my mentors on areas to work on. 

One of the areas was on the conversion between `TreeNode` and `BPTree`. Previously, conversion was a general function that takes a `BPTree`/`TreeNode` and outputs it counterpart object. The suggestion was to have **class methods** within `TreeNode` to do this instead to carry out the conversion directly. 

Another piece of feedback that I acted upon was regarding comestic changes to redundant code and imports that existed within the Cython files. Also, there were some changes made to simplifying the code within the test cases too. 

## Deferred items

There were other pieces of feedback that were given and a major theme of the remaining comments tied to the integration of the `BPTree` I/O functionality into the submodule that was available within *scikit-bio* already (`skbio.io`). Cruially, since *iow* implementation can support multiple different formats to represent a BP tree, including *jplace* which is new to *scikit-bio*'s existing I/O registry, it likely would likely involve a massive body of work. Given that the current PR is quite large already, it is recommended to merge the changes first and defer the actual changes here required to a later PR. 

Another topic that would be deferred is on some of the optimization of the `BPTree` performance. Through my discussion with my mentors, they advised me that there is likely a need to review what level of the `BPTree` API should be exposed as Python and what should be kept as Cython. This is motivated by the trade-off that, despite exposing API as pure Python to be more transparent in terms of debugging and user interfacing, the bulk of the performance gains from `BPTree` likely comes from submerging much of the `BPTree` backend into the Cython and having the computation handled through C. Therefore, this optimization exercise would be beneficial.

## Upcoming targets

Having handled the work that deliver the PR, the upcoming tasks planned involve:
- Working on the I/O integration issue to accomodate for *newick* and *jplace* formats
- Creating a more uniform and robust benchmarking workflow for the performance of the `BPTree`, such that any further optimization of the class can be measured and justified