---
layout: page
title: SIL Ownership Model
categories: draft
---

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Abstract](#abstract)
- [Ownership SSA Form](#ownership-ssa-form)
- [ValueOwnershipKind, UseLifetimeConstraints, and Dataflow Constraints](#valueownershipkind-uselifetimeconstraints-and-dataflow-constraints)
    - [Trivial](#trivial)
    - [Owned](#owned)
    - [Guaranteed](#guaranteed)
        - [Shared Borrows](#shared-borrows)
        - [Immediate Shared Borrow and Guaranteed Function Arguments](#immediate-shared-borrow-and-guaranteed-function-arguments)
        - [Guaranteed Block Arguments](#guaranteed-block-arguments)
    - [Unowned](#unowned)
    - [Any](#any)
- [Structural SIL Changes](#structural-sil-changes)
    - [Functions](#functions)
    - [SILValue](#silvalue)
    - [Operand](#operand)
    - [Instruction](#instruction)
        - [Constant Ownership Instructions](#constant-ownership-instructions)
        - [Aggregates with Non-Trivial and Trivial Fields](#aggregates-with-non-trivial-and-trivial-fields)
        - [Value Projections vs Destructures](#value-projections-vs-destructures)
        - [Forwarding instructions](#forwarding-instructions)
    - [SILArgument](#silargument)
        - [Explicit ownership](#explicit-ownership)
        - [Enums with Trivial/Non-Trivial Cases](#enums-with-trivialnon-trivial-cases)

<!-- markdown-toc end -->

## Abstract

Today, SIL does not express or validate ownership properties internally to
SILFunctions. This increases the difficulty of SILGening "ownership correct" SIL
and optimizing ownership in an effective, safe way. This document proposes an
ownership model for SIL based around an augmented form of SSA called Ownership
SSA or OSSA that propagates ownership invariants along def-use edges. This
entails adding new types to SIL enable ownership invariants to be specified,
changing SILValue (def) and Operand (use) to express constraints using these new
types, and finally enforcing these constraints via a verifier. This in
combination with SIL expressing ownership semantics along call graph edges will
allow for entire Swift programs to be verified as statically preserving
ownership invariants.

The general layout of this document is:

* First, Ownership SSA is defined abstractly.
* Then for each case of ``ValueOwnershipKind``, we define the relevant dataflow
  rules and explain how the ``UseLifetimeConstraint`` provided by a using
  operand ties into those dataflow rules.
* Finally, the structural, semantic changes to SIL necessary to implement this
  model are described.

## Ownership SSA Form

`Ownership SSA` or `OSSA` is a derived form of SSA that expresses ownership
invariants along def-use edges. SILGen initially emits SIL in OSSA form. Once
optimizations that depend on ownership have been completed, the SIL is lowered
out of OSSA by a SIL pass called the ``OwnershipModelEliminator``. OSSA augments
the normal SSA concepts (e.x. dominance) in SIL by introducing the following
new data types:

* ``ValueOwnershipKind`` - An enum whose cases categorizes the ownership
  semantics that a def conforms to.
* ``UseLifetimeConstraint`` - An enum that categorizes a use of a def as either
  invalidating the def or requiring that the def still be live (i.e. not
  invalidated).
* ``OperandOwnershipKindMap`` - A per operand map from a ``ValueOwnershipKind``
  that is compatible with the operand to the ``UseLifetimeConstraint`` that a
  value with said ``ValueOwnershipKind`` would have placed upon it as a result
  of being used by the operand.

and then defining the following rules:

* All defs (``SILValue``) must define a static ``ValueOwnershipKind``.
* Every use (``Operand``) must define a static ``OperandOwnershipKindMap`` that
  maps a ``ValueOwnershipKind`` that the operand can accept to the
  ``UseLifetimeConstraint`` constraint that the use places on the value at the
  use-site.
* A SIL program is ill-formed if there exists any def-use edge whose def's
  ``ValueOwnershipKind`` is not a key in the use's ``OperandOwnershipKindMap``.
* A SIL program is ill-formed if any def-use edge violates the dataflow rule of
  the def's ``ValueOwnershipKind`` given the ``UseLifetimeConstraint`` produced
  by use's ``OperandOwnershipKindMap`` for the ownership kind.

These dataflow rules enable us to make program guarantees such as eliminating
the possibility of leaks, use-after-frees, and dangling pointers. We describe
the specific dataflow constraints in the next section: ``ValueOwnershipKinds and
UseLifetimeConstraints``.

## ValueOwnershipKind, UseLifetimeConstraints, and Dataflow Constraints

In OSSA, all SILValues must produce a ``ValueOwnershipKind`` that describes the
value's dataflow rules and how the value relates to its uses. A quick summary of
each case of ``ValueOwnershipKind``:

* **Trivial** - ``@trivial`` - A pure SSA value that is never invalidated and is
  live throughout the program.
* **Owned** - ``@owned`` - A value with a unique, independent lifetime that is
  created once and invalidated exactly once along any path through the program.
* **Guaranteed** - ``@guaranteed`` - A value whose scoped lifetime is dependent
  on the liveness of a separate ``@owned`` value. Naturally, the dependence
  constrains the ``@owned`` value's lifetime such that the ``@owned`` value must
  be alive for the entire lifetime of the guaranteed value.
* **Unowned** - ``@unowned`` - A value with a transient lifetime that must be
  copied before any side-effect having uses.
* **Any** - Undefined ownership. This is used to model ``SILUndef`` and
  ownership in SIL once ownership has been stripped out.

Every ``Operand`` in SIL defines an ``OperandOwnershipKindMap`` that maps an
individual ``ValueOwnershipKind`` that the operand can be paired with to a
``UseLifetimeConstraint``. This defines the effect that the use has upon the
lifetime of the value. A quick summary of each case of
``UseLifetimeConstraint``:

* **MustBeLive** - The use requires that the value not be invalidated at the use
  point and does not invalidate the value.
* **MustBeInvalidated** - The use requires the value to be live at the use point
  and invalidates the value. After the use point can no longer be referenced or
  used without triggering undefined behavior.

In combination, these two constructs define a set of dataflow rules that the def
and its uses (and potentially derived uses) must obey. This is implemented by
all defs having the ability to partition their use set into two according to
each individual use's ``UseLifetimeConstraint`` and then applying the
``ValueOwnershipKind`` specific dataflow rules to those use sets. We describe
these dataflow rules below.

### Trivial

A SILValue with ``@trivial`` ownership is an independent, unmanaged value like
``Int``, ``UnsafePointer<T>``, ``Builtin.RawPointer``, and ``Unmanaged<T>``. It
is assumed that all values with trivial ownership are of trivial type. Ownership
SIL classifies all uses of a trivial value as putting a ``MustBeLive``
constraint on the value and thus does not provide any dataflow guarantees on
trivial values. A SIL module that attempts to put a ``MustBeInvalidated``
lifetime constraint on a trivial value is ill formed.

### Owned

A SILValue with ``@owned`` ownership is an independent, managed value like
``Array<T>`` or a class. It is assumed that all values with ``@owned`` ownership
are of non-trivial type. Ownership SIL defines ``@owned`` values as having the
following dataflow guarantees:

1. The value will have exactly one ``MustBeInvalidated`` use along any path
   through the program. This ensures that the compiler can prevent memory leaks
   and double frees.
2. Any ``MustBeLive`` uses of the value will occur before any
   ``MustBeInvalidated`` uses of the value. This ensures that the compiler can
   prevent any use after frees.
3. Any guaranteed values produced from the ``@owned`` value via a "shared
   borrow" must be invalidated before the ``@owned`` value is destroyed.

### Guaranteed

A value with ``@guaranteed`` ownership is a scoped-lifetime value that is
derived from an ``@owned`` value. The ``@guaranteed`` value's lifetime acts as a
``MustBeLive`` on the lifetime of the ``@owned`` value. This is implemented by
considering the ``end_borrow`` of a ``@guaranteed`` value to be an implicit use
of the original ``@owned`` value. A ``@guaranteed`` value can be introduced via
the following SILNodes:

* ``load_borrow``
* ``begin_borrow``
* ``@guaranteed SILArgument``

The values derivable from these SILNodes must be paired with exactly one
``end_borrow`` use along all paths through the program.

NOTE: In the case of ``load_borrow``, the ``@owned`` value is stored in
memory. This value while the borrow is live can not be destroyed. For the case
of arbitrary memory this is enforced by describing such an operation as
undefined behavior. In the future, this may be enforced statically.

#### Shared Borrows

The act of producing an ``@guaranteed`` value is also known by the term of art
"shared borrow". Colloquially a ``@guaranteed`` value is sometimes called a
"borrowed" value and the scoped-lifetime of the value a "borrow scope".

It is important to note that a SIL-borrow has different semantics than a
Swift-borrow since SIL-borrow is a lower level implementation detail of a
Swift-borrow. For instance, a Swift-borrow enforces additional dynamic language
level guarantees around exclusive access to memory that a SIL-borrow avoids by
stating such conditions are undefined behavior. This is not an issue in practice
since SILGen emits Swift-level borrow as a combination of a ``begin_access``
(providing the additional guarantees) and a SIL-level borrow as a low level
implementation detail to perform the actual +0 operation.

#### Immediate Shared Borrow and Guaranteed Function Arguments

In order to reduce the size of the emitted SIL, we do not require ``@owned``
parameters to be explicitly borrowed when used by instructions that are known to
create a borrow scope and then immediately close the borrow scope. We call this
an immediately shared borrow. The premier example of such an instruction would
be an apply with a guaranteed argument. In this case, 

In contrast, an instruction that 

#### Guaranteed Block Arguments

``@guaranteed`` block arguments are treated specially 

### Unowned

A SILValue with ``@unowned`` ownership kind is an independent value that has a
lifetime that is only guaranteed to last until the next program visible
side-effect. To maintain the lifetime of an ``@unowned`` value, it must be
converted to an owned representation via a ``copy_value`` before any other uses
of it.

``@unowned`` ownership kind occurs mainly along method/function boundaries in
between Swift and Objective-C code. This is because ``@unowned`` is the normal
argument convention used in Objective-C.

### Any

A SILValue with undefined ownership. It can pair with /any/ ownership kind. This
means that it could take on /any/ ownership semantics. This is meant only to
model ``SILUndef`` and to express certain situations where we use unqualified
ownership. This is a tool in migration that over time will be increasingly
restricted until in the SIL ownership model, it will only be used to model
``SILUndef``.

## Structural SIL Changes

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
OperandOwnershipKindMap Operand::getOwnershipKindMap() const;
```

This defines a set of possible ownership kind that this operand can be paired
with. It is defined via a SILVisitor subclass that creates the ownership kind
map by switching over all SILInstructionKind and deriving ValueOwnershipKinds
via an operand specific ownership rule.

### Instruction

The addition of ownership, does not change the textual or in memory
representaiton of SILInstructions, but semantically, adds new requirements upon
each instruction since each Operand's OperandOwnershipKindMap is defined by the
operand's position in an instruction's operand list. Each of these ownership
semantics will be documented in SIL.rst and will be enforced via the ownership
verifier. We describe some of the more interesting categories below to give the
reader a gist of the behavior of various common instructions.

#### Constant Ownership Instructions

There are many instructions that always produce a value with the same ownership
requirement no matter the input. Some examples:

* ``alloc_box``, ``copy_value``, ``partial_apply`` always produce a value with
  ``Owned`` ownership.
* ``begin_borrow``, ``load_borrow``, ``struct_extract``, ``tuple_extract``
  always produce a value with ``Guaranteed`` ownership.
* Instructions such as ``ref_to_raw_pointer``, ``tuple_element_addr`` that work
  with addresses and trivially typed values always produce values with
  ``Trivial`` ownership.

#### Aggregates with Non-Trivial and Trivial Fields

#### Value Projections vs Destructures

A value projection is an instruction like ``struct_extract``, ``tuple_extract``,
and ``unchecked_enum_data`` that enables one to get a reference to an internal
field of an aggregate without destroying the entire underlying aggregate. This
creates a natural tension with the ownership model since an ``@owned`` SSA value
can never be partially dead since that would the semantics of the SSA
value. Rather than change these value projections semantics, we instead treat
them as read only views into the underlying SSA value and enforce that by
requiring their operands to be ``@guaranteed``.

Sometimes one does want to completely extract out all of the fields in a struct
in a destructive way. To perform such functionality, one can use the destructure
instructions. A destructure completely splits up an aggregate operand into its
constituate subtypes. This enables a destructure to consume an entire ``@owned``
value and produce new ``@owned`` values for each of 

get a read only view into a larger aggregate type. This creates a natural
tension with the ownership model since a value projection does been destroy the
entire original value. Thus we model these instructions by requiring their
operands to be ``@guaranteed`` to ensure that the underlying ``@owned`` value
that we are extracting from remains alive.

#### Forwarding instructions

In the previous sections, we mentioned instructions that "forwarded
ownership". What does that actually entail?

A forwarding instruction forwards ownership from the operands of the instruction
to the results of the instruction. For an ``Owned`` operand, the forwarding
instruction invalidates the operand's lifetime by mapping ``Owned`` ownership to
``MustBeInvalidated`` and then produces a new value with owned ownership. This
ensure that the original ``Owned`` value does not have any uses after the
forwarding instruction, preserving ``Owned`` ownership semantics. For a value
with ``Guaranteed`` ownership, the forwarding instruction propagates forward the
guaranteed ownership and does not invalidate the lifetime of the input
guaranteed value. The forwarding instruction during ownership verification will
be looked though to verify that additional uses of the guaranteed value are
before the ``end_borrow`` of the original value. Some examples of simple
forwarding instructions are:

1. casts in between object types.
2. destructure_struct, destructure_tuple

There are also more complicated forms of forwarding that involve aggregate
forming instructions such as ``struct`` and ``tuple``. For these aggregates, we
allow for the operand's of the aggregate to be either trivial or a specific
consistent ownership kind.

NOTE: Terminator and aggregate forming instructions have special ownership
semantics to allow for us.

Terminator instructions

### SILArgument

#### Explicit ownership when in Memory Ownership, but Implicit when written.

As mentioned above, we require a SILArgument's in to specify its ownership
explicitly to ensure that we do not need to consider loops. This causes some
changes in textual SIL since we now need to print out the ownership attribute
when we print out the SILArgument, e.x.:

```
sil @f : $@convention(thin) (@owned Builtin.NativeObject) -> () {
bb0(%0 : $Builtin.NativeObject):
  br bb1(%0 : $Builtin.NativeObject)

bb1(%1 : $Builtin.NativeObject):
  cond_br ..., bb2, bb3

bb2:
  br bb1(%1 : $Builtin.NativeObject)

bb3:
  destroy_value %1 : $Builtin.NativeObject
  %9999 = tuple()
  return %9999 : $()
}
```

Notice how in the previous example since we have specified ownership upon the
arguments, we do not need to analyze the loop and can instead just validate that
every owned value is properly consumed by a terminator or a destroy. Naturally
we require all incoming values to a SILArgument to have the same ownership as.

#### Enums with Trivial/Non-Trivial Cases

* If a SIL argument is of enum type, we allow for trivial enum cases to be
  passed to the SIL argument even if the argument has non-trivial
  ownership. E.x.::

```
    enum MyEnum {
    case trivial(Int)
    case nonTrivial(Klass)
    }

    sil @useMyEnum : $@convention(thin) (@owned MyEnum) -> ()
    sil @f : $@convention(thin) (Int) -> () {
    bb0(%0 : @trivial Int):
      %1 = function_ref @useMyEnum : $@convention(thin) (@owned MyEnum) -> ()
      %2 = enum $MyEnum, #MyEnum.trivial!enumelt.1, %0
      apply %1(%2) : $@convention(thin) (@owned MyEnum) -> ()
      ...
    }
```

  We do this by allowing for block and function arguments to be places where a
  trivial enum case can migrate to either a guaranteed or owned ownership. This
  is safe since the trivial value is always live and the compiler knows how to
  handle destroying a trivial case of an enum by no-oping appropriately. We do
  require that all non-trivial inputs to such arguments have matching ownership.

* We require that basic block arguments with guaranteed ownership must have
  associated end_borrows to ease verification and optimization. The end_borrow
  must occur before all input guaranteed parameter's end_borrow so that the
  argument's lifetime is strictly contained withint its incoming value's
  lifetimes.
