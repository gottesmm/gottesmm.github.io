---
layout: page
title: SIL Ownership Model
categories: draft
---
<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Abstract](#abstract)
- [Ownership SSA Formally](#ownership-ssa-formally)
- [Ownership SSA In SIL](#ownership-ssa-in-sil)
- [SIL Changes for OSSA](#sil-changes-for-ossa)
    - [Functions](#functions)
    - [SILValue](#silvalue)
    - [Operand](#operand)
    - [Instruction](#instruction)
        - [Borrow Instructions](#borrow-instructions)
        - [Lifetime Maintenance Instructions](#lifetime-maintenance-instructions)
        - [Function Application/Coroutine Instructions](#function-applicationcoroutine-instructions)
        - [ForwardOrReborrow Instructions](#forwardorreborrow-instructions)
        - [Value Projections and Destructures](#value-projections-and-destructures)
    - [SILArgument](#silargument)
<!-- markdown-toc end -->

## Abstract

Today, SIL does not express or validate ownership properties internally to a
``SILFunction``. This increases the difficulty of SILGening "ownership correct"
SIL and optimizing ownership in an effective, safe way. This document proposes
an ownership model for SIL based around an augmented form of SSA called
Ownership SSA (``OSSA``). This in combination with SIL expressing ownership
semantics along call graph edges will allow for entire Swift programs to be
verified as statically preserving ownership invariants. We describe ``OSSA``
below:

## Ownership SSA Formally

Let _f_ be a SIL function. A SSA value _v_ in _f_ is immortal and can be
referenced at any point dominated by _v_'s definition. When SIL is in
OSSA form, we extend SSA by requiring that all such _v_ provide a static map
from any point _p_ dominated by _v_ to a boolean discriminator that defines
whether or not a use of _v_ at _p_ results in _f_ being ill-formed. We call this
the _availability map_ for _v_ at _p_ and write _Availability(v)(p)_. We require
_Availability(v)(p)_ to have the following properties:

* **Availability is Well Defined**: If _v_ dominates a block _B_, then _Available(v)_ must be
  the same along all incoming edges into _B_.

* **Availability does not Abandon Values**: If _v_ does not dominate a block _B_, but does
  dominate a predecessor of _B_, then _Available(v)_ must be false in
  _B_.

* **Availability does not Resurrect Values**: A program is ill-formed if there
  exists points _p_, _p'_ with _p'_ being reachable from _p_ and
  _Availability(v)(p')_ being true and _Availability(v)(p)_ being false.

By specifying properties of a value's availability map, we can define ownership
semantics for the value and enforce them in SIL. The first kind of ownership
semantic that we define is that of the "owned" value. Let _v_ be an _owned_
value in _f_. Then _v_ must have the following properties:

* **Owned values have Independent Availability**: If there exists a value _v'_
  and point _p_ in _f_ such that _Availability(v')(p)_ implies
  _Availability(v)(p)_, then there must exist a point _p'_ in _f_ that is not
  reachable from the definition of _v'_ but for which _Availability(v)(p)_ is
  true.

* **Owned values are Mortal**: There must exist a set of "consuming" uses,
  _Consume(v)_, of _v_ that joint-post dominate all other uses of _v_ and after
  which _v_ is no longer available.

* **Owned values are consumed exactly once**: _f_ is ill formed if there exists
  a program path in between any uses _u_, _u'_ in _Consume(v)_.

Given an owned value _o_, we can derive scoped "borrowed" values from _o_ via
"borrow operations". We require any such _b_ to have the following properties:

* **Borrowed values have Dependent Availability**: _b_'s availability must
depend on _o_'s within the scope in which _b_ is available. This dependence is
enforced by _b_ having the property that if _Availability(o)(p)_ is false, then
_Availability(b)(p)_ must also be false.

* **Borrowed values have Scoped Availability**: _b_ must be joint-post dominated
by a set of _end borrow_ operations that invalidate _b_.

Naturally, we can iterative apply additional borrow operations to _b_ to get
additional _b'_ that have dependent lifetime on _b_ (and thus _o_ as well). An
owned value and the associated borrow values in OSSA form a _semantic
value_. Define a _semantic value_, _s_, as a rooted subgraph of the def-use
graph of _f_ with a singular owned value root, _Root(s)_ and for which the set
of all non-root values, _Borrow(s)_, are derived from _Root(s)_ via iterative
borrow operations. Conceptually all values in _s_ refer to the same underlying
semantic program entity (e.g. a class instance or a struct with non-trivial
fields). Via corrolaries to our previous definitions, it is easy to see that:

* **Semantic Values can only be used if the root is alive**.

* **Semantic Values have Independent Availability** Independent semantic values
   _s_, _s'_ must have independent availability. This follows from the
   independent availability of _Root(s)_ and _Root(s')_.



1. _Root(s)_ must be available if any _Borrow(s)_ is available. This follows
   from borrowed values having dependent availability.

2. 

Now that we have our abstract model of Ownership SSA, we define how this is
implemented in SIL.

## Ownership SSA In SIL

In order to define this model in SIL, we begin by defining our _owned_ and
_borrowed_ values. We extend the categorization slightly since in SIL we must
also consider objc-bridging and trivial values. This results in us classifying
values in the following four categories:

* **Owned** - _Root(s)_ for a semantic value _s_. Just like in our model, we
  require _owned_ values to be used by a consuming operation exactly once along
  any program path dominated by its definition.

* **Borrowed** - A value _v_ that is an element of some _SSA(s)_ that is
  produced from an _owned_ value via nested scoped borrow operations. We
  loosen our definition slightly by allowing for the _owned_ value to be in a
  different function (function argument), the result of a terminator instruction
  (block argument), in coroutine storage (a return value from coroutine
  application), or in memory. In all such cases, we require the same
  availability constraint to apply and that the lifetime of the borrow is
  similarly scoped via a semantic program entity.

* **Any** - An immortal value that after is always available after being
  defined. This is a divergence from the formal model since the formal model
  does not address such values. We necessarily add it to our model so that we
  can model trivial values as well as non-trivial sum types that act like
  trivial values from an ownership perspective (e.g. enum cases with trivial or
  without any payload). We rely on the type system to enforce that these values
  are passed only to places that can accept these values. In all such locations,
  we are necessarily performing an ownership merge. See below for more detail
  about ownership merging.

* **Unowned** - A value that is only available up to the first side-effect
  having instruction after its definition. This is a special case needed for
  objc-compatibility and the ordering of the side-effect constraint is unmodeled
  in the ownership model today. We do model that unowned must be copied before
  being used in a borrowed or owned context.

In code, these are modeled as cases of the enum ``ValueOwnershipKind``. Just
like we categorize the ownership semantics of values, we categorize the
ownership semantics of the effects of these upon their operand values:

* **Simple** - A read only use of an available value. The value can have any type
of ownership semantics.

* **Consuming** and **Forwarding** - A use of either an Any or Owned available
value. If the input value has ``Any`` ownership, this is a point use since the
input value is immortal. If the input value has ``Owned`` ownership, the value
becomes unavailable after the use's execution.

* **Reborrow** - A use that takes in an available value _v_ with either ``Any``
or ``Borrowed`` ownership. If _v_ has ``Any`` ownership, this is a point use
of the input value since it is immortal. If _v_ has ``Borrowed`` ownership,
then the evaluation of the use adds all non-``Any`` results of the use to all
borrow sets containing _v_. The program is ill-formed if any of these results of
the uses of _v_ are available when _v_ is unavailable. This results in recursive
validation of the borrow scopes.

While ``Point`` and ``Consuming`` only effect the incoming value to the
instruction, ``Forwarding`` and ``Reborrow`` operands additionally constrain the
results of instructions as follows:

* All ``Forwarding`` and ``Reborrow`` operands of an instruction must specify
  via an instruction specific rule the set of instruction results whose
  ownership semantics are dependent on the operand's ownership.

* If all of the ``Forwarding`` and ``Reborrow`` operands mapped to a single
  result have ``Any`` ownership, then the result must have ``Any`` ownership.

* If any of the ``Forwarding`` (``Reborrow``) operands mapped to a result have
  ``Owned`` (``Borrowed``) ownership, then a non-trivially typed specified
  result must have ``Owned`` (``Borrowed``) ownership. If the result value is
  trivially typed, then it will have ``Any`` ownership.

* Ownership Consistency: A program is ill-formed if a ``Forwarding`` operand and
  ``Reborrow`` operand of an instruction specify that they effect the same
  instruction result.

These categories will be modeled in the compiler via the enum
OperandOwnershipUseKind.

## SIL Changes for OSSA

To implement this proposal, several structural changes must be made to SIL. We
go through each below:

### Functions

To distinguish in between functions in OSSA form and those that are not, we
define a new SILFunction attribute called ``[ossa]`` that signifies that a
function is in OSSA form and must pass the ownership verifier.

### SILValue

We add two new methods to SILValue's API:

```
ValueOwnershipKind SILValue::getOwnershipKind() const;
bool SILValue::verifyOwnership() const;
```

The first API returns the ownership kind associated with the value. This is
implemented via a visitor struct that statically can map a value via its
ValueKind to the relevant ownership semantics.

The second API returns true if the value considered as a def is compatible with
its users.

### Operand

The only changed required of Operand is the addition of a new API:

```
OperandOwnershipUseKind Operand::getOwnershipUseKind() const;
```

This returns the statically mapped use semantics of this operand.

### Instruction

The addition of ownership does not change the textual or in memory
representation of SILInstructions but semantically adds new requirements upon
each instruction. Each of these ownership semantics will be documented in
SIL.rst and will be enforced via the ownership verifier. We describe the more
important categories of instructions below.

#### Borrow Instructions

The instructions are used to introduce new borrow values:

- ``begin_borrow`` requires its argument to be available and adds its result
  value to the borrow set of its argument.
- ``end_borrow`` requires that its argument value be an available result value
  of a ``begin_borrow`` that has an empty borrow set. Upon execution the
  ``end_borrow`` removes the ``begin_borrow`` value from /all/ borrow sets at
  the ``end_borrow``.
- ``load_borrow`` works analogously to ``begin_borrow``.

#### Lifetime Maintenance Instructions

These instructions are used to manipulate the lifetime of non-trivial
values. Specifically:

- ``copy_value`` acts as a ``Point`` use of its argument value and produces a
  new copy of the value with ``Owned`` ownership.
- ``destroy_value`` acts as a consuming use of its argument.

#### Function Application/Coroutine Instructions

Function Application instructions such as ``apply``, ``try_apply`` generally
have OperandOwnershipUseKind semantics that one would expect except for one
special case: function parameters that are passed ``@guaranteed``. In this case
since the use of the ``@guaranteed`` value is immediate upon function
application, we treat the use as a point use.

In constrast, since coroutines are not evaluated instantaneously, we require
that coroutine instructions perform a form of "pseudo"
reborrowing. Specifically:

- ``begin_apply`` can only accept available borrowed arguments in
  ``@guaranteed`` positions and adds itself to their borrow sets.
- ``end_apply`` and ``abort_apply`` remove the ``begin_apply`` from the borrow
  sets of all its borrowed arguments. That the ``begin_apply`` is still in the
  borrow sets at that point is implied by the separate formation rules of
  ``begin_apply``.

#### ForwardOrReborrow Instructions

There are certain classes of instructions that we allow to have either
``Forwarding`` or ``Reborrow`` operands. To ensure consistency, we only allow
for this property to be set when the instruction is constructed. Some examples
of these types of instructions:

* Aggregate Forming Instructions - ``struct``, ``tuple``, ``enum``.
* Casts - ``unchecked_ref_cast``, ``upcast``, ``unconditional_checked_cast``
* Enum extraction operations - ``unchecked_enum_data``.
* Existential opening of classes - ``open_existential_ref``

_NOTE: That ``unchecked_enum_data`` is a special case since enums always contain
conceptually a single tuple as their payload._

#### Value Projections and Destructures

Value projections such as ``tuple_extract``, ``struct_extract`` that allow for
one to access the internal fields of a value without destroying the value. This
implies that the original value can not be destroyed. As such we model these as
instructions with ``Reborrow`` semantics. In contrast, destructure operations
such as ``destructure_struct`` and ``destructure_tuple`` take in a single value
and split it up into the constituant first level fields of the value. As such,
we model destructure operations as having ``Forwarding`` ownership.

### SILArgument

In order to support the ownership model, we require the following changes to
SILArgument:

1. All SILArguments must specify a fixed ``ValueOwnershipKind`` on
   construction.

2. All SILPhiArguments are treated like an instruction result of the set of
   predecessor terminators. This means that the rules around ``Forwarding`` and
   ``Reborrow`` operands apply to any such SILPhiArguments.

3. Any SILArgument with ``Borrowed`` ownership is not required to have pairing
   ``end_borrow``s. This is because a ``SILFunctionArgument`` naturally has an
   ``end_borrow`` like scope (the function itself) and ``SILPhiArgument``s with
   ``Guaranteed`` ownership rely on the ``end_borrow`` set of its incoming
   values.
