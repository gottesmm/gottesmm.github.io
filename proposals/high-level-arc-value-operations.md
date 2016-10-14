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

Then we wire up the building blocks:

1. SILGen will be taught to emit `copy_value`, `copy_unowned_value`, and
   `destroy_value` instead of `strong_retain`, `retain_value`, `strong_release`,
   `release_value`, and `strong_retain_unowned`.

2. The pass manager will not require any changes due to previous work
   in [High Level ARC Memory Operations](high-level-arc-memory-operations).

3. The verifier when in EnforceSILOwnershipMode will verify that none of the
   instructions that we wish to remove are in the IR.

## Optimizer Changes

Since the SILOwnershipModel eliminator will eliminate the `copy_value`,
`copy_unowned_value`, and `destroy_value` operations right after ownership
verification, there will be no immediate effects on the optimizer and thus the
optimizer changes can be done in parallel with the rest of the SIL Ownership
work. But in the long run, we want these instructions to be lowered by IRGen
implying that we must consider how this will affect the rest of the optimizer
pipeline.

We now go through all of the passes that will need updating to handle these
changes:

### ARC Optimizer

Since the ARC optimizer does not perform code motion any more, only minimal
changes will be required. Specifically, all we must do is recognize copy_value
as a retain instruction and destroy_value as a release instruction. Everything
then will *just* work.

### ARC Code Motion and Function Signature Optimization

Both of these passes will need to recognize `copy_value`, `destroy_value`
instructions as retain, release and be changed to emit `copy_value` or
`destroy_value` instead of retain, release instructions. In the case of ARC code
motion rather than just re-emitting retain, release instructions without
considering use-def lists, it must now consider such issues to ensure that we do
not violate use-def dominance.

### Misc compiler Peepholes: SILCombine, Mandatory Inlining, etc.

There are many peepholes in the compiler that emit retain, release instructions
for ARC optimizations to clean up later. These must be updated to use the new
instructions. This will be mechanical.

### Side Effects

The side effects subsystem needs to be updated to handle copy_value like it does
a retain and destroy_value like it does a release. This should be mechanical.
