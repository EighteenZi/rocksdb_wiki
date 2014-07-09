# Motivation
Currently, we observed the following needs in universal compactions:
* Avoid doubling in space during compaction.
* Allow universal compaction to be multi-threaded.

To satisfy the above needs, a reasonable solution is to partition large SST files in universal compaction into multiple smaller SST files, which allows deleting some of the SST files during the compaction to avoid doubling in space and provides multi-thread opportunities.  Such solution makes universal compaction behaves very similar to level compaction.

In the meantime, we also observed the following needs in level compaction:
* Allow compaction across multiple levels to reduce write amplification.

This undoubtably makes level compaction behaves similar to universal compaction.  This leads us to a global solution --- unifying level and universal compaction.

# The Goals
The goal of unifying level and universal compactions is to take the advantages from the two compactions.  Specifically, we would like to achieve the following goals:

* Reduce write amplification by considering compacting files across multiple levels.
* Reduce storage space requirement by deleting unused files during the compaction.
* Improve scalability by partitioning one big compaction work into multiple smaller compaction jobs.
* Reduce maintenance effort by unifying the two code paths into one.

# Possible Solutions
There are basically three approaches to unify universal and level compactions, where I found option 1 might be the best among the all:

1. Add universal compaction features to level compaction and deprecate universal compaction.
 * pros: Changes can be incremental, easy to maintain and test.  Can reuse existing optimizations on level compaction.
2. Add level compaction features to universal compaction and deprecate level compaction.
 * pros: Changes can be incremental, easy to maintain and test.
 * cons: Requires redo many optimizations that are already done in level compaction.
3. Create a new compaction algorithm and depreciate both level and universal compactions.
 * pros: More flexibility to do code refactor.
 * cons: Changes are likely not incremental.  Requires more effort on integrating with existing code.


# The Proposed Approach
To support universal compaction in the current level compaction, we need to enable the following features:
* Allow compaction across multiple levels.
* Allow cross-level compaction to be partitioned into multiple smaller compaction jobs to improve concurrency.
* Allow number of levels to be dynamically increase or decrease, optionally without upper-bound limit.

To allow compaction across multiple levels, we need to have a way to determine when to extend the current compaction to cover the next level, and here's my proposal, which basically search for the best output level for the current compaction based on the write amplification factor.

Let `SM` denote the size multiplier between two consecutive levels, describing the size ratio between the target size of `Level N` and the target size of `Level N + 1` (in universal compaction, this number is configured to be 2.).  Consider a situation where we would like to determine whether to compaction a set `f_N`, where files in `f_N` can be any file in `level 0` to `level N`.  We use `|f_N|` to denote the size of all files in `f_N`, and use `F_N+1` to denote the set of files in `level N + 1` having overlapping key-range with `f_N`, and `|F_N + 1|` to denote the size of all files in `F_N + 1`.  Let `WAF(f_N) = min(f_N, |F_N+1|) / (|f_N| + |F_N+1)` be the write amplification factor.  Then consider the following two cases:

* Case 1 --- `WAF(f_N) <= SM`: we have no-lower (no-worse) than average write amplification than average.  It is a good timing to compact `F` into `level N`.
* Case 2 --- `WAF(f_N) > SM`: we have higher (worse) write amplification than average.  It is not a good timing to compact `F` into `level N`.  Should try to include more files in `F` or remove some files from `F` to improve write amplification.

Let `f_N+1` be the union of `f_N` and `F_N+1`, and we find the biggest `n >= N` such that `WAF(f_n) <= SM`.  If none of the possible `n` yields `WAF(f_n) <= SM`, then we pick the best `n` which yields minimal `WAF(f_n)`.

## Example
TBD

## Partition one big compaction work into multiple sub-jobs