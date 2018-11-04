---
layout: page
title: SIL Ownership Model
categories: draft
---
<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Abstract](#abstract)
- [Ownership SSA Form](#ownership-ssa-form)
- [SIL Changes for OSSA](#sil-changes-for-ossa)
    - [Functions](#functions)
    - [SILValue](#silvalue)
    - [Operand](#operand)
    - [Instruction](#instruction)
        - [Borrow Instructions](#borrow-instructions)
        - [Lifetime Maintenance Instructions](#lifetime-maintenance-instructions)
        - [Function Application/Coroutine Instructions](#function-applicationcoroutine-instructions)
        - [ForwardingOrReborrow Instructions](#forwardingorreborrow-instructions)
        - [Value Projections and Destructures](#value-projections-and-destructures)
     - [SILArguments](#silarguments)

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

In SIL, an SSA value _v_ is immortal and can be referenced at any point in the
program dominated by _v_'s definition. When SIL is in OSSA form, we extend SSA
by introducing the notion of a _semantic value_. A semantic value _s_ is a
non-empty set of SSA values, _SSA(s)_, that all refer to the same underlying
semantic program entity (e.g. a class instance or a struct with non-trivial
fields). We require that all values _v_ in _SSA(s)_ have a static map from any
point _p_ that _v_ dominates to a boolean discriminator that describes if a
program is ill-formed if _v_ is used at _p_. We call this the _availability map_
for _v_ at _p_ and write _Availability(v)(p)_. We require _Availability(v)(p)_
to have the following properties:

* **Availability is Well defined**: If _v_ dominates a block _B_, then _Available(v)_ must be
  the same along all incoming edges into _B_.

* **Availability does not Abandon Values**: If _v_ does not dominate a block _B_, but does
  dominate a predecessor of _B_, then _Available(v)_ must be false in
  _B_. _NOTE: There are special rules for terminators without successors, we
  explain them below in the section Ownership SSA in SIL._

* **Availability does not Resurrect Values**: A program is ill-formed if there
  exists points _p_, _p'_ with _p'_ being reachable from _p_ and
  _Availability(v)(p')_ being true and _Availability(v)(p)_ being false.

* **Availability does not allow Immortal Values**: A program is ill-formed if
  there exists a value _v_ in _SSA(s)_ for which there does not exist a program
  point _p_ dominated by _v_ where _Availability(v)(p)_ is false.

Given any such _v_, we define that any value _b_ derived from _v_ by a scoped
_borrow operation_ must also be an element of _SSA(s)_. We define a borrow
operation as an operation that:

1. Uses a value _v_ and defines a new value _b_ whose availability depends on
_v_'s within the scope in which _b_ is available. This dependence is enforced by
_b_ having the property that if _Availability(v)(p)_ is false, then
_Availability(b)(p)_ must also be false.

2. Is joint-post dominated by a set of _end borrow_ operations that after which
_b_ is no longer available.

Since _SSA(s)_ is closed under borrow operations, a natural equivalence
class structure arises if one considers the elements of _SSA(s)_ that are
derived via iterative borrowed operations from the same underlying value to be
equivalent. For our purposes, we wish to model semantics where there is only one
such dominating value, so we restrict our definition of semantic values by
stating that given any semantic value _s_ there must exist a single static value
_D(s)_ in _SSA(s)_ that dominates all other _v_ in _SSA(s)_. We call this _D(s)_
an _owned_ value and classify all other _v_ in _SSA(s)_ as _borrowed_ values.

Beyond borrow operations, we model a few other types of operations that we
describe below:

1. Simple uses. An instruction that uses a _v_ in _SSA(s)_ and does not affect
   _v_'s availability as a result of the use.

2. Copy operations. An instruction that takes in any _v_ in _SSA(s)_ and
   produces a new _owned_ value _o_ that acts as _D(s)_ for a new semantic value
   _s'_. Since _s'_ is a different semantic value from _s_, there are no
   lifetime dependencies in between any elements of _SSA(s)_ or _SSA(s')_.

3. Consuming operations. An instruction that takes in an _owned_ value and
   invalidates it. Naturally this ends the lifetime of a semantic value since
   all borrowed values must be at that point unavailable and values can not be
   resurrected. By contraposition this implies that if a use does not have an
   empty borrow set, then it can not be consumed. An important corollary of this
   definition is that since owned values can not be immortal and can not be
   resurrected, all owned values must necessarily be consumed exactly once along
   all paths through the program.

Now that we have our abstract model of Ownership SSA, we define how this is
implemented in SIL.

## Ownership SSA In SIL

In order to define this model in SIL, we begin by defining our _owned_ and
_borrowed_ values. We extend the categorization slightly since in SIL we must
also consider objc-bridging and trivial values. This results in us classifying
values in the following four categories:

* **Owned** - _D(s)_ for a semantic value _s_. Just like in our model, we
  require _owned_ values to be used by a consuming operation exactly once along
  any program path dominated by its definition.

* **Borrowed** - A value _v_ that is an element of some _SSA(s)_ that is
  produced from an _owned_ value via iterative scoped borrow operations. We
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
