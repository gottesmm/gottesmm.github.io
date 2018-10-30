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

## Ownership SSA Form

Ownership SSA Form or `OSSA` is a derived form of SSA that augments the normal
SSA concepts (e.x. dominance) of SIL by defining the following rules: Let _p_ be
a point in a SIL program then,

1. **Values are statically available or unavailable**: For any dominating value
   _v_ of _p_, there must exist a statically derivable mapping of _v_ to one of
   ``{available, unavailable}`` at _p_.

2. **Unavailable values can not be used**: A program is ill-formed if there
   exists a dominating value _v_ of _p_ that is both used at _p_ and is
   unavailable at _p_.

3. **Values have static borrowed value sets**: All dominating values _v_ of _p_
   must be statically mappable to a (potentially) empty set of "borrowed" values
   at _p_.

4. **Values in a borrowed set must be available**: A program is ill-formed if
   there exists dominating values _v_, _v'_ of _p_ with _v'_ being unavailable
   at _p_ and _v'_ being an element of the borrowed set of _v_ at _p_.

5. **Values with non-empty borrow sets must be available**: A program is
   ill-formed if there exists a dominating value _v_ of _p_ that is both
   unavailable at _p_ and has a non-empty borrow set at _p_.

6. **States are well-defined**: A value _v_ which dominates a block _B_ must
   have the same incoming availability state and incoming borrow set from all
   predecessors of _B_.

7. **Values are never abandoned**: Given a block _B_, a predecessor block
   _Pred(B)_ of _B_, and a value _v_ that does not dominate _B_, if _v_
   dominates _Pred(B)_ then _v_ must be unavailable in _B_. For the purposes of
   this rule, the terminators with no successors are treated as follows:
     - ``return``, ``throw``, and ``unwind`` are considered to have a destination which
       is never dominated by anything.
     - ``unreachable`` is considered to not have a destination and thus is
       unconstrained.

8. **Values cannot be resurrected**: A dominating value _v_ of _p_ that is
   unavailable at _p_ can not be available at any point reachable from _p_.

These rules naturally lead us to two important corollaries:

* **Uses can only cause values with empty borrow sets to become unavailable**:
  If the evaluation of a use _u_ of _v_ causes _v_ to become unavailable, then
  _v_ must have an empty borrow set at _u_. Otherwise, we would be violating
  rule 4.

* **Uses can only add to borrow sets of available values**. If the evaluation
  of a use _u_ of _v_ results in the addition of a value _v'_ to _v_'s borrow
  set, _v_ must be available at _u_.

Using this model, we define 4 different categories of ownership semantics that a
value in SIL can have:

* **Any** - An immortal value that after definition is always available. It can
  not have a non-empty borrow set and can not be placed into another value's
  borrow set. All trivially typed values have ``Any`` ownership as well as all
  SIL values in non-OSSA sil.

* **Owned** - A value with a static set of joint-postdominating uses that
  consume the value. ``Owned`` values can have a borrow set, but are not allowed
  to be in another value's borrow set. This is used to model move only values as
  well as individual copies of reference counted values.

* **Guaranteed** - A value _v_ that is available within a single entry-multiple
  exit region of code and must be in the borrow set of a single value _v'_ at
  all points within that region. This can be used to model "shared borrow" like
  constructs. Since a guaranteed value is in a borrow set at all points where it
  is available, naturally it cannot be consumed due to rule 4.

* **Unowned** - A value that is only available up to the first side-effect
  having instruction after its definition. This is a special case needed for
  objc-compatibility and the ordering of the side-effect constraint is unmodeled
  in the ownership model today. We do model that unowned must be copied before
  being used in a guaranteed or owned context.

These are modeled as cases of the enum ``ValueOwnershipKind``. Just like we
categorize the ownership semantics of values, we categorize the ownership
semantics of the effects of these upon their operand values:

* **Point** - A read only use of an available value. The value can have any type
of ownership semantics.

* **Consuming** and **Forwarding** - A use of either an Any or Owned available
value. If the input value has ``Any`` ownership, this is a point use since the
input value is immortal. If the input value has ``Owned`` ownership, the value
becomes unavailable after the use's execution.

* **Reborrow** - A use that takes in an available value _v_ with either ``Any``
or ``Guaranteed`` ownership. If _v_ has ``Any`` ownership, this is a point use
of the input value since it is immortal. If _v_ has ``Guaranteed`` ownership,
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
  ``Owned`` (``Guaranteed``) ownership, then a non-trivially typed specified
  result must have ``Owned`` (``Guaranteed``) ownership. If the result value is
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

The instructions are used to introduce new guaranteed values:

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

#### ForwardingOrReborrow Instructions

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

3. Any SILArgument with ``Guaranteed`` ownership is not required to have pairing
   ``end_borrow``s. This is because a ``SILFunctionArgument`` naturally has an
   ``end_borrow`` like scope (the function itself) and ``SILPhiArgument``s with
   ``Guaranteed`` ownership rely on the ``end_borrow`` set of its incoming
   values.
