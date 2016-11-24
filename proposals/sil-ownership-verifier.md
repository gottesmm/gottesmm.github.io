---
layout: proposal
title: SIL Ownership Verification
categories: proposals
---

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Summary](#summary)
- [Representing Ownership in SIL](#representing-ownership-in-sil)
- [Mapping ValueBase to ValueOwnershipKind](#mapping-valuebase-to-valueownershipkind)
- [Proving Def-Use Convention Correctness](#proving-def-use-convention-correctness)
- [Identifying Ownership Dataflow Errors](#identifying-ownership-dataflow-errors)

<!-- markdown-toc end -->

# Summary

This document defines a SIL ownership model and a verifier that enforces this
model. Both of these together allow for a program in SIL to be verified as
satisfying all Swift lifetime constraints.

# SIL Ownership Model

We define the SIL ownership model by embedding ownership into SIL's SSA
def-use edges. This is done by:

1. Defining a meet lattice of ownership kinds
2. Defining all def-use edges as having an ownership kind produced by the edge's
   def and consumed by the edge's use.
3. Defining a program as being malformed if all def-use edges do not have
   compatible produced and consumed ownership kinds.

First, define the meet pseudo-lattice via the enum `ValueOwnershipKind` and the
following cases:

* `Trivial`
* `Unowned`
* `Owned`
* `Guaranteed`
* `InOut`
* `Any`
* `Unknown`

Each one of these ownership kinds corresponds to already well known categories
of SIL ownership, but there are two new cases: `Any` and `Unknown`. We define
these as follows:

* `Any`. `⊤` of the lattice. A def that can be paired with a use accepting any
ownership kind. This is needed for SILUndef. A use that accepts `Any` ownership
kind is able to be paired with a def with any ownership kind. This is needed for
instructions like `copy_value`. `Any` is the `top` element of the lattice.
* `Unknown`. `⊥` of the lattice. This is only produced when attempting to
intersect lattice elements that are incompatible with each other.

We define our meet operation (with `*` meaning `Unknown`) via the following chart:

| Meet Op      | `Trivial` | `Unowned` | `Owned` | `Guaranteed` | `InOut` | `Any`        | `Unknown` |
|--------------|-----------|-----------|---------|--------------|---------|--------------|-----------|
| `Trivial`    | `Trivial` | `*`       | `*`     | `*`          | `*`     | `Trivial`    | `*`       |
| `Unowned`    | `*`       | `Unowned` | `*`     | `*`          | `*`     | `Unowned`    | `*`       |
| `Owned`      | `*`       | `*`       | `Owned` | `*`          | `*`     | `Owned`      | `*`       |
| `Guaranteed` | `*`       | `*`       | `*`     | `Guaranteed` | `*`     | `Guaranteed` | `*`       |
| `InOut`      | `*`       | `*`       | `*`     | `*`          | `InOut` | `InOut`      | `*`       |
| `Any`        | `Trivial` | `Unowned` | `Owned` | `Guaranteed` | `InOut` | `Any`        | `*`       |
| `Unknown`    | `*`       | `*`       | `*`     | `*`          | `*`     | `*`          | `*`       |

<!--

     /// The ownership semantics of a def-use edge.
     enum class ValueOwnershipKind {

       /// Represents unaddited ownership.
       ///
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


1. Defining an enum called `ValueOwnershipKind` that specifies possible
ownership along a def-use edge.
2. Implementing the API `ValueOwnershipKind ValueBase::getOwnershipKind() const`
to vend these values.
3. Implementing the API `void SILInstruction::verifyOperandOwnership() const`
that verifies that a `SILInstruction`'s operands have ownership that is
compatible with the `SILInstruction`'s ownership.

These 3 points will enable for all def-use edges in SIL to be statically
verified as obeying ownership semantics.

Define `ValueOwnershipKind` as follows:




# Mapping ValueBase to ValueOwnershipKind

# Proving Def-Use Convention Correctness

# Identifying Ownership Dataflow Errors
-->
