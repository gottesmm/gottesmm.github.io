---
layout: proposal
title: SIL Ownership Verification
categories: proposals
---

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Summary](#summary)
    - [](#)

<!-- markdown-toc end -->


# Summary

This document proposes an ownership verifier to ensure that a SIL program is
ownership correct. To implement this we:

1. Define a lattice of ownership kinds. (`ValueOwnershipKind`)
2. Define a method of assigning a `ValueOwnershipKind` to all `ValueBase`.
3. Define a method of verifying that all uses of a SIL def satisfy the def's
   ownership constraints.
4. Define a method for verifying that ownership dataflow constraints are
   preserved by SIL SSA values and defs.

# Representing Ownership in SIL

In Semantic SIL, all ownership relations are represented in SSA form along
def-use edges. We propose implementing this by:

1. Defining an enum called `ValueOwnershipKind` that specifies possible
ownership along a def-use edge.
2. Implementing the API `ValueOwnershipKind ValueBase::getOwnershipKind() const`
to vend these values.
3. Implementing the API `void SILInstruction::verifyOperandOwnership() const`
that verifies that a `SILInstruction`'s operands have ownership that is
compatible with the `SILInstruction`'s ownership.

Define `ValueOwnershipKind` as follows:

     /// The ownership semantics of a use-def edge.
     enum class ValueOwnershipKind {

       /// Represents the ownership of a value that has not been audited or is
       /// actually undefined. A def that produces a value with unknown
       /// ownership can not be paired with any use that does not have Unknown
       /// or Any ownership.
       Undefined,

       /// Represents the ownership of a trivially typed SSA value. Can only be
       /// paired with uses with Any or Trivial ownership.
       Trivial,

       /// Represents the ownership of a non-trivial value that must be
       /// immediately retained before use. This is used generally in
       /// objective-c conventions.
       Unowned,

       /// Represents the ownership of a non-trivial value that is being passed
       /// at +1.
       Owned,

       /// Represents the ownership of an immutably borrowed value being passed
       /// at +0.
       Guaranteed,

       /// Represents the ownership of a mutably borrowed value being passed at
       /// +0.
       InOut,

       /// Top. Represents any ownership.
       Any,
     };



# Mapping ValueBase to ValueOwnershipKind

# Proving Def-Use Convention Correctness

# Identifying Ownership Dataflow Errors
