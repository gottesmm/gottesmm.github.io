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

This document defines a SIL ownership model and a verifier for the ownership
model. Both of these together allow for the static verification that a SIL
program satisfies all ownership model constraints. This provides a model of
semantics that SILGen can then target and use to implement ownership model
correct Swift programs.

**TODO** Make this better.

# SIL Ownership Model

The SIL ownership model embeds ownership into SIL's SSA def-use edges. This is
accomplished by:

1. Specifying a bottom pseudo-lattice of ownership kinds.
2. Requiring all def-use edges as having an ownership kind produced by the
   edge's def and consumed by the edge's use.
3. Defining SIL program as being well formed iff all def-use edges have def-use
   ownership kinds that when intersected do not yield
   `ValueOwnershipKind::Invalid`.

First, define the bottom pseudo-lattice via the enum `ValueOwnershipKind` and the
following cases:

* `Trivial`
* `Unowned`
* `Owned`
* `Guaranteed`
* `InOut`
* `Any`
* `Invalid`

Each one of these ownership kinds corresponds to already well known categories
of SIL ownership, but there are two new cases: `Any` and `Invalid`. We define
these as follows:

* `Any`. `⊤` of the lattice. A def that can be paired with a use accepting any
ownership kind. This is needed for SILUndef. A use that accepts `Any` ownership
kind is able to be paired with a def with any ownership kind. This is needed for
instructions like `copy_value`. `Any` is the `top` element of the lattice.
* `Invalid`. `⊥` of the lattice. This is only produced when attempting to
intersect lattice elements that are incompatible with each other.

We define our bottom operation (with `*` meaning `Invalid`) via the following chart:

| Bottom Op    | `Trivial` | `Unowned` | `Owned` | `Guaranteed` | `InOut` | `Any`        | `Invalid` |
|--------------|-----------|-----------|---------|--------------|---------|--------------|-----------|
| `Trivial`    | `Trivial` | `*`       | `*`     | `*`          | `*`     | `Trivial`    | `*`       |
| `Unowned`    | `*`       | `Unowned` | `*`     | `*`          | `*`     | `Unowned`    | `*`       |
| `Owned`      | `*`       | `*`       | `Owned` | `*`          | `*`     | `Owned`      | `*`       |
| `Guaranteed` | `*`       | `*`       | `*`     | `Guaranteed` | `*`     | `Guaranteed` | `*`       |
| `InOut`      | `*`       | `*`       | `*`     | `*`          | `InOut` | `InOut`      | `*`       |
| `Any`        | `Trivial` | `Unowned` | `Owned` | `Guaranteed` | `InOut` | `Any`        | `*`       |
| `Invalid`    | `*`       | `*`       | `*`     | `*`          | `*`     | `*`          | `*`       |

Now that we have defined intersection, we categorize all ValueBase into sets
depending on the ValueBase's result ownership properties (if a result
exists). These categories are:

1. *No Result*. A ValueBase without a result.
2. *Constant Ownership*. A ValueBase with a result that always produces the same
   ownership kind.
3. *Forwarding Ownership*. A ValueBase that is an Instruction whose result is
   always equivalent to the same ownership kind as one of the instruction's operands.
4. *Special Ownership*. An instruction with special rules for propagating
   ownership. This includes ValueBase such as ApplyInst, SILArgument, and
   TryApply.

Using these categories, we implement a method on ValueBase called
ValueBase::getOwnershipKind(). This will be implemented using a visitor to
ensure that warnings are provided when engineers add new ValueBase and do not 

A visitor will be used to implement this code ensuring that a warning will be
provided if an engineer adds a new ValueBase and does not add support for
implementing getOwnershipKind();

<!--

     /// The ownership semantics of a def-use edge.
     enum class ValueOwnershipKind {

       /// Represents unaddited ownership.
       ///
       /// Represents the ownership of a value that has not been audited or is
       /// actually undefined. A def that produces a value with unknown
       /// ownership can not be paired with any use that does not have Invalid
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
