---
layout: proposal
title: High Level ARC Value Operations
categories: proposals
---

# {{ page.title }}

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Summary](#summary)
- [Definitions](#definitions)
    - [copy_value](#copyvalue)
    - [copy_unowned_value](#copyunownedvalue)
    - [destroy_value](#destroyvalue)
- [Implementation](#implementation)
    - [Plan](#plan)
    - [Optimizer Changes](#optimizer-changes)

<!-- markdown-toc end -->

# Summary

This document proposes:

1. removing the `strong_retain`, `retain_value`, `unowned_retain`, and
   `strong_retain_unowned` instructions.
2. adding the `copy_value` and `copy_unowned_value` instructions.
3. removing the `strong_release`, `release_value`, `unowned_release`
   instructions.
4. adding the `destroy_value` instruction.

This will allow for the SIL IR to express ownership conventions via SSA use-def
chains.

# Definitions

## copy_value

Define a `copy_value` as follows:

    %y = copy_value %x : $*C
    use(%y)

      =>

    retain_value %x : $C
    use(%x)

## copy_unowned_value

Define a `copy_unowned_value` as:

    %y = copy_unowned_value %x : $@unowned T

      =>

    strong_retain_unowned %x : $@unowned T

where `%y` has type `$T`. This is necessary to enable the strong reference count
associated with the `copy_unowned_value` to be balanced by a
`destroy_addr`. Thus normally one will see this instruction used as follows:

    %y = copy_unowned_value %x : $@unowned T
    ...
    destroy_value %y : $T

## destroy_value

Define a `destroy_value` as follows:

    destroy_value %x : $C

      =>

    release_value %x : $C

# Implementation

## Plan

We assume that
the [High Level ARC Memory Operations](high-level-arc-memory-operations)
proposal has been implemented. First we perform the following preliminary work:

1. `copy_value`, `destroy_value`, `copy_unowned_value` support will be added to
   SIL, IRGen, serialization, printing, SIL parsing. SILGen will not be modified
   at this stage.

2. The "OwnershipModelEliminator" will be taught to transform `copy_value`,
   `copy_unowned_value`, `destroy_value` into their constituant operations.

3. The verifier when in EnforceSILOwnershipMode will verify that none of the
   instructions that we wish to remove are in the IR.

Then we wire up the building blocks:

1. SILGen will be taught to emit `copy_value`, `copy_unowned_value`, and
   `destroy_value`.

2. The pass manager will not require any changes due to previous work
   in [High Level ARC Memory Operations](high-level-arc-memory-operations).

## Optimizer Changes

Since the SILOwnershipModel eliminator will eliminate the `copy_value`,
`copy_unowned_value`, and `destroy_value` operations right after ownership
verification, there will be no immediate effects on the optimizer and thus the
optimizer changes can be done in parallel with the rest of the ARC optimization
work. But in the long run, IRGen must handle these instructions so ownership
invariants can be enforced all throughout the SIL pipeline.

The high level implementation plan is to 
The high level plan is to eliminate all direct references to the deleted
instructions in the SIL Optimization utilities and passes and instead use free
standing utility functions to allow a pass to support both cases. In order to
distinguish in between `strong_*` and `*_value` cases, we will use the type of
the value passed into the `copy_value` or `destroy_value`. This in combination
with the verifier will ensure that we are able to catch any cases where we
missed

We now go through all of the passes that will need updating to handle these
changes:

### ARC Optimizer

The main changes to the ARC optimizer is that the ARC optimizer will have to
emit `copy_value`, `destroy_value` instructions instead of retain, release
instructions. Since we are not enforcing pairing now, the ARC optimizer does not
in response to this change need to ensure proper pairings are created after
eliminating retain/release pairs.

### ARC Code Motion and Function Signature Optimization

Both of these passes will need to recognize `copy_value`, `destroy_value`
instructions as retain, release and be changed to emit `copy_value` or
`destroy_value` instead of retain, release instructions.

### Lower Aggregate Instrs

This pass will need to no longer blow up destroy_value operations.

### SILCombine

There are some peepholes here that will need to be rewritten to recognize
copy_value, destroy_value.
