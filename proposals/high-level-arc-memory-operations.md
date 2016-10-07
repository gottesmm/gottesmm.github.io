---
layout: proposal
title: High Level ARC Memory Operations
categories: proposals
---

# {{ page.title }}

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Summary](#summary)
- [Definitions](#definitions)
    - [ownership qualified load](#ownership-qualified-load)
    - [ownership qualified store](#ownership-qualified-store)
- [Implementation](#implementation)
    - [Goals](#goals)
    - [Plan](#plan)
    - [Optimizer Changes](#optimizer-changes)
        - [store->load forwarding](#store-load-forwarding)
        - [ARC Code Motion](#arc-code-motion)
        - [Normal Code Motion](#normal-code-motion)
        - [ARC Optimization](#arc-optimization)
        - [Function Signature Optimization](#function-signature-optimization)
- [Appendix](#appendix)
    - [Partial Initialization of Loadable References in SIL](#partial-initialization-of-loadable-references-in-sil)
    - [Case Study: Partial Initialization and load [[copy]]](#case-study-partial-initialization-and-load-copy)
        - [The Problem](#the-problem)
        - [Defining load [[copy]]](#defining-load-copy)

<!-- markdown-toc end -->

# Summary

This document proposes:

1. adding the following ownership qualifiers to `load`: `[take]`, `[copy]`,
   `[guaranteed]`, `[trivial]`.
2. adding the following ownership qualifiers to `store`: `[init]`, `[assign]`,
   `[trivial]`.
3. requiring all `load` and `store` operations to have ownership qualifiers.
4. banning the use of `load [trivial]`, `store [trivial]` on memory locations of
   `non-trivial` type.

This will allow for:

1. eliminating optimizer miscompiles that occur due to releases being moved into
   the region in between a `load`/`retain`, `load`/`release`,
   `store`/`release`. (For a specific example, see the appendix).
2. explicitly modeling `load [trivial]`/`store [trivial]` as having `unsafe
   unowned` ownership semantics. This will be enforced via the verifier.
3. more aggressive ARC code motion.

# Definitions

## ownership qualified load

We propose four different ownership qualifiers for load. Define `load [trivial]`
as:

    %x = load [trivial] %x_ptr : $*Int

      =>

    %x = load %x_ptr : $*Int

A `load [trivial]` can not be used to load values of non-trivial type. Define
`load [copy]` as:

    %x = load [copy] %x_ptr : $*C

      =>

    %x = load %x_ptr : $*C
    retain_value %x : $C

Then define `load [take]` as:

    %x = load [take] %x_ptr : $*Builtin.NativeObject

      =>

    %x = load %x_ptr : $*Builtin.NativeObject

**NOTE** `load [take]` implies that the loaded from memory location no longer
owns the result object (i.e. a take is a move). Loading from the memory location
again without reinitialization is illegal.

Next we provide `load [guaranteed]`:

    %x = load [guaranteed] %x_ptr : $*Builtin.NativeObject
    ...
    fixLifetime(%x)

      =>

    %x = load %x_ptr : $*Builtin.NativeObject
    ...
    fixLifetime(%x)

`load [guaranteed]` implies that in the region before the fixLifetime,
the loaded object is guaranteed semantically to remain alive. The fixLifetime
communicates to the optimizer the location up to which the value's lifetime is
guaranteed to live. An example of where this construct is useful is when one has
a let binding to a class instance `c` that contains a let field `f`. In that
case `c`'s lifetime guarantees `f`'s lifetime.

## ownership qualified store

First define a `store [trivial]` as:

    store %x to [trivial] %x_ptr : $*Int

      =>

    store %x to %x_ptr : $*Int

The verifier will prevent this instruction from being used on types with
non-trivial ownership. Define a `store [assign]` as follows:

    store %x to [assign] %x_ptr : $*C

       =>

    %old_x = load %x_ptr : $*C
    store %new_x to %x_ptr : $*C
    release_value %old_x : $C

*NOTE* `store` is defined as a consuming operation. We also provide
`store [init]` in the case where we know statically that there is no
previous value in the memory location:

    store %x to [init] %x_ptr : $*C

       =>

    store %new_x to %x_ptr : $*C

# Implementation

## Goals

Our implementation strategy goals are:

1. zero impact on other compiler developers until the feature is fully
   developed. This implies all work will be done behind a flag.
2. separation of feature implementation from updating passes.

Goal 2 will be implemented via a pass that transforms ownership qualified
`load`/`store` instructions into unqualified `load`/`store` right after SILGen.

## Plan

We begin by adding initial infrastructure for our development. This means:

1. Adding to SILOptions a disabled by default flag called
 "EnableSILOwnershipModel". This flag will be set by a false by default frontend
 option called "-enable-sil-ownership-mode".

2. Bots will be brought up to test the compiler with
   "-enable-sil-ownership-model" set to true. The specific bots are:

   * RA-OSX+simulators
   * RA-Device
   * RA-Linux.

   The bots will run once a day until the feature is close to completion. Then a
   polling model will be followed.

Now that change isolation is guaranteed, we develop building blocks for the
optimization:

1. Two enums will be defined: `LoadInstOwnershipQualifier`,
   `StoreInstOwnershipQualifier`. The exact definition of these enums are as
   follows:

       enum class LoadOwnershipQualifier {
         Unqualified, Take, Copy, Guaranteed, Trivial
       };
       enum class StoreOwnershipQualifier {
         Unqualified, Init, Assign, Trivial
       };

    *NOTE* `LoadOwnershipQualifier::Unqualified` and
    `StoreOwnershipQualifier::Unqualified` are only needed for staging purposes.

2. Creating a `LoadInst`, `StoreInst` will be changed to require an ownership
qualifier. At this stage, this argument will default to `Unqualified`. "Bare"
`load`, `store` when parsed via textual SIL will be considered to be
unqualified. This implies that the rest of the compiler will not have to be
changed as a result of this step.

3. Support will be added to SIL, IRGen, Serialization, SILPrinting, and SIL
Parsing for the rest of the qualifiers. SILGen will not be modified at this
stage.

4. A pass called the "OwnershipModelEliminator" will be implemented. It will
   blow up all `load`, `store` instructions with non `*::Unqualified` ownership
   into their constituant ARC operations and `*::Unqualified` `load`, `store`
   insts.

3. An option called "EnforceSILOwnershipMode" will be added to the verifier. If
the option is set, the verifier will assert if:

   a. `load`, `store` operations with trivial ownership are applied to memory
      locations with non-trivial type.

   b. `load`, `store` operation with unqualified ownership type are present in
   the IR.

Finally, we wire up the building blocks:

1. If SILOption.EnableSILOwnershipModel is true, then the after SILGen SIL
   verification will be performed with EnforceSILOwnershipModel set to true.
2. If SILOption.EnableSILOwnershipModel is true, then the pass manager will run
   the OwnershipModelEliminator pass right after SILGen before the normal pass
   pipeline starts.
3. SILGen will be changed to emit non-unqualified ownership qualifiers on load,
   store instructions when the EnableSILOwnershipModel flag is set. We will use
   the verifier throwing to guarantee that we are not missing any specific
   cases.

Then once all of the bots are green, we change SILOption.EnableSILOwnershipModel
to be true by default. After a cooling off period, we move all of the code
behind the SILOwnershipModel flag in front of the flag. We do this so we can
reuse that flag for further SILOwnershipModel changes.

## Optimizer Changes

Since the SILOwnershipModel eliminator will eliminate the ownership qualifiers
on load, store instructions right after ownership verification, there will be no
immediate affects on the optimizer and thus the optimizer changes can be done in
parallel with the rest of the ARC optimization work.

But, in the long run, we want to enforce these ownership invariants all
throughout the SIL pipeline implying these ownership qualified `load`, `store`
instructions must be processed by IRGen, not eliminated by the SILOwnershipModel
eliminator. Thus we will need to update passes to handle these new
instructions. The main optimizer changes can be separated into the following
areas: memory forwarding, dead stores, ARC optimization. In all of these cases,
the necessary changes are relatively trivial to respond to. We give a quick
taste of two of them: store->load forwarding and ARC Code Motion.

### store->load forwarding

Currently we perform store->load forwarding as follows:

    store %x to %x_ptr : $C
    ... NO SIDE EFFECTS THAT TOUCH X_PTR ...
    %y = load %x_ptr : $C
    use(%y)

      =>

    store %x to %x_ptr : $C
    ... NO SIDE EFFECTS THAT TOUCH X_PTR ...
    use(%x)

In a world, where we are using ownership qualified load, store, we have to also
consider the ownership implications. *NOTE* Since we are not modifying the
store, `store [assign]` and `store [init]` are treated the same. Thus without
any loss of generality, lets consider solely `store`.

    store %x to [assign] %x_ptr : $C
    ... NO SIDE EFFECTS THAT TOUCH X_PTR ...
    %y = load [copy] %x_ptr : $C
    use(%y)

      =>

    store %x to [assign] %x_ptr : $C
    ... NO SIDE EFFECTS THAT TOUCH X_PTR ...
    strong_retain %x
    use(%x)

### ARC Code Motion

If ARC Code Motion wishes to move the ARC semantics of ownership qualified
`load`, `store` instructions, it must now consider read/write effects. On the
other hand, it will be able to now not consider the side-effects of destructors
when moving retain/release operations.

### Normal Code Motion

Normal code motion will lose some effectiveness since many of the load/store
operations that it used to be able to move now must consider ARC information. We
may need to consider running ARC code motion earlier in the pipeline where we
normally run Normal Code Motion to ensure that we are able to handle these
cases.

### ARC Optimization

The main implication for ARC optimization is that instead of eliminating just
retains, releases, it must be able to recognize ownership qualified `load`,
`store` and set their flags as appropriate.

### Function Signature Optimization

Semantic ARC affects function signature optimization in the context of the owned
to guaranteed optimization. Specifically:

1. A `store [assign]` must be recognized as a release of the old value that is
   being overridden. In such a case, we can move the `release` of the old value
   into the caller and change the `store [assign]` into a `store [init]`.
2. A `load [copy]` must be recognized as a retain in the callee. Then function
   signature optimization will transform the `load [copy]` into a `load
   [guaranteed]`. This would require the addition of a new `@guaranteed` return
   value convention.

# Appendix

## Partial Initialization of Loadable References in SIL

In SIL, a value of non-trivial loadable type is loaded from a memory location as
follows:

    %x = load %x_ptr : $*S
    ...
    retain_value %x_ptr : $S

At first glance, this looks reasonable, but in truth there is a hidden drawback:
the partially initialized zone in between the load and the retain
operation. This zone creates a period of time when an "evil optimizer" could
insert an instruction that causes x to be deallocated before the copy is
finished being initialized. Similar issues come up when trying to perform a
store of a non-trival value into a memory location.

Since this sort of partial initialization is allowed in SIL, the optimizer is
forced to be overly conservative when attempting to move releases passed retains
lest the release triggers a deinit that destroys a value like `%x`. Lets look at
two concrete examples that show how semantically providing ownership qualified
load, store instructions eliminate this problem.

**NOTE** Without any loss of generality, we will speak of values with reference
semantics instead of non-trivial values.

## Case Study: Partial Initialization and load [copy]

### The Problem

Consider the following swift program:

    func opaque_call()

    final class C {
      var int: Int = 0
      deinit {
        opaque_call()
      }
    }

    final class D {
      var int: Int = 0
    }

    var GLOBAL_C : C? = nil
    var GLOBAL_D : D? = nil

    func useC(_ c: C)
    func useD(_ d: D)

    func run() {
        let c = C()
        GLOBAL_C = c
        let d = D()
        GLOBAL_D = d
        useC(c)
        useD(d)
    }

Notice that both `C` and `D` have fixed layouts and separate class hierarchies,
but `C`'s deinit has a call to the function `opaque_call` which may write to
`GLOBAL_D` or `GLOBAL_C`. Additionally assume that both `useC` and `useD` are
known to the compiler to not have any affects on instances of type `D`, `C`
respectively and useC assigns `nil` to `GLOBAL_C`. Now consider the following
valid SIL lowering for `run`:

    sil_global GLOBAL_D : $D
    sil_global GLOBAL_C : $C

    final class C {
      var x: Int
      deinit
    }

    final class D {
      var x: Int
    }

    sil @useC : $@convention(thin) () -> ()
    sil @useD : $@convention(thin) () -> ()

    sil @run : $@convention(thin) () -> () {
    bb0:
      %c = alloc_ref $C
      %global_c = global_addr @GLOBAL_C : $*C
      strong_retain %c : $C
      store %c to %global_c : $*C                                              (1)

      %d = alloc_ref $D
      %global_d = global_addr @GLOBAL_D : $*D
      strong_retain %d : $D
      store %d to %global_d : $*D                                              (2)

      %c2 = load %global_c : $*C                                               (3)
      strong_retain %c2 : $C                                                   (4)
      %d2 = load %global_d : $*D                                               (5)
      strong_retain %d2 : $D                                                   (6)

      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c2) : $@convention(thin) (@owned C) -> ()              (7)

      %useD_func = function_ref @useD : $@convention(thin) (@owned D) -> ()
      apply %useD_func(%d2) : $@convention(thin) (@owned D) -> ()              (8)

      strong_release %d : $D                                                   (9)
      strong_release %c : $C                                                   (10)
    }

Lets optimize this function! First we perform the following operations:

1. Since `(2)` is storing to an identified object that can not be `GLOBAL_C`, we
   can store to load forward `(1)` to `(3)`.
2. Since a retain does not block store to load forwarding, we can forward `(2)`
   to `(5)`. But lets for the sake of argument, assume that the optimizer keeps
   such information as an analysis and does not perform the actual load->store
   forwarding.
3. Even though we do not foward `(2)` to `(5)`, we can still move `(4)` over
   `(6)` so that `(4)` is right before `(7)`.

This yields (using the ' marker to designate that a register has had load-store
forwarding applied to it):

    sil @run : $@convention(thin) () -> () {
    bb0:
      %c = alloc_ref $C
      %global_c = global_addr @GLOBAL_C : $*C
      strong_retain %c : $C
      store %c to %global_c : $*C                                              (1)

      %d = alloc_ref $D
      %global_d = global_addr @GLOBAL_D : $*D
      strong_retain %d : $D
      store %d to %global_d : $*D                                              (2)

      strong_retain %c : $C                                                    (4')
      %d2 = load %global_d : $*D                                               (5)
      strong_retain %d2 : $D                                                   (6)

      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c) : $@convention(thin) (@owned C) -> ()               (7')

      %useD_func = function_ref @useD : $@convention(thin) (@owned D) -> ()
      apply %useD_func(%d2) : $@convention(thin) (@owned D) -> ()              (8)

      strong_release %d : $D                                                   (9)
      strong_release %c : $C                                                   (10)
    }

Then by assumption, we know that `%useC` does not perform any releases of any
instances of class `D`. Thus `(6)` can be moved past `(7')` and we can then pair
and eliminate `(6)` and `(9)` via the rules of ARC optimization using the
analysis information that `%d2` and `%d` are th same due to the possibility of
performing store->load forwarding. After performing such transformations, `run`
looks as follows:

    sil @run : $@convention(thin) () -> () {
    bb0:
      %c = alloc_ref $C
      %global_c = global_addr @GLOBAL_C : $*C
      strong_retain %c : $C
      store %c to %global_c : $*C                                              (1)

      %d = alloc_ref $D
      %global_d = global_addr @GLOBAL_D : $*D
      strong_retain %d : $D
      store %d to %global_d : $*D

      %d2 = load %global_d : $*D                                               (5)
      strong_retain %c : $C                                                    (4')
      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c) : $@convention(thin) (@owned C) -> ()               (7')

      %useD_func = function_ref @useD : $@convention(thin) (@owned D) -> ()
      apply %useD_func(%d2) : $@convention(thin) (@owned D) -> ()              (8)

      strong_release %c : $C                                                   (10)
    }

Now by assumption, we know that `%useD_func` does not touch any instances of
class `C` and `%c` does not contain any ivars of type `D` and is final so none
can be added. At first glance, this seems to suggest that we can move `(10)`
before `(8')` and then pair/eliminate `(4')` and `(10)`. But is this a safe
optimization perform?  Absolutely Not! Why? Remember that since `useC_func`
assigns `nil` to `GLOBAL_C`, after `(7')`, `%c` could have a reference count
of 1.  Thus `(10)` _may_ invoke the destructor of `C`. Since this destructor
calls an opaque function that _could_ potentially write to `GLOBAL_D`, we may be
be passing `%d2`, an already deallocated object to `%useD_func`, an illegal
optimization!

Lets think a bit more about this example and consider this example at the
language level. Remember that while Swift's deinit are not asychronous, we do
not allow for user level code to create dependencies from the body of the
destructor into the normal control flow that has called it. This means that
there are two valid results of this code:

- Operation Sequence 1: No optimization is performed and `%d2` is passed to
  `%useD_func`.
- Operation Sequence 2: We shorten the lifetime of `%c` before `%useD_func` and
   a different instance of `$D` is passed into `%useD_func`.

The fact that 1 occurs without optimization is just as a result of an
implementation detail of SILGen. 2 is also a valid sequence of operations.

Given that:

1. As a principle, the optimizer does not consider such dependencies to avoid
   being overly conservative.
2. We provide constructs to ensure appropriate lifetimes via the usage of
   constructs such as fix_lifetime.

We need to figure out how to express our optimization such that 2
happens. Remember that one of the optimizations that we performed at the
beginning was to move `(6)` over `(7')`, i.e., transform this:

      %d = alloc_ref $D
      %global_d_addr = global_addr GLOBAL_D : $D
      %d = load %global_d_addr : $*D             (5)
      strong_retain %d : $D                      (6)

      // Call the user functions passing in the instances that we loaded from the globals.
      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c) : $@convention(thin) (@owned C) -> ()                (7')

into:

      %global_d_addr = global_addr GLOBAL_D : $D
      %d2 = load %global_d_addr : $*D             (5)

      // Call the user functions passing in the instances that we loaded from the globals.
      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c) : $@convention(thin) (@owned C) -> ()                (7')
      strong_retain %d2 : $D                      (6)

This transformation in Swift corresponds to transforming:

      let d = GLOBAL_D
      useC(c)

to:

      let d_raw = load_d_value(GLOBAL_D)
      useC(c)
      let d = take_ownership_of_d(d_raw)

This is clearly an instance where we have moved a side-effect in between the
loading of the data and the taking ownership of such data, that is before the
`let` is fully initialized. What if instead of just moving the retain, we moved
the entire let statement? This would then result in the following swift code:

      useC(c)
      let d = GLOBAL_D

and would correspond to the following SIL snippet:

      %global_d_addr = global_addr GLOBAL_D : $D

      // Call the user functions passing in the instances that we loaded from the globals.
      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c) : $@convention(thin) (@owned C) -> ()                (7')
      %d2 = load %global_d_addr : $*D                                         (5)
      strong_retain %d2 : $D                                                  (6)

Moving the load with the strong_retain to ensure that the full initialization is
performed even after code motion causes our SIL to look as follows:

    sil @run : $@convention(thin) () -> () {
    bb0:
      %c = alloc_ref $C
      %global_c = global_addr @GLOBAL_C : $*C
      strong_retain %c : $C
      store %c to %global_c : $*C                                              (1)

      %d = alloc_ref $D
      %global_d = global_addr @GLOBAL_D : $*D
      strong_retain %d : $D
      store %d to %global_d : $*D

      strong_retain %c : $C                                                    (4')
      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c) : $@convention(thin) (@owned C) -> ()               (7')

      %d2 = load %global_d : $*D                                               (5)
      %useD_func = function_ref @useD : $@convention(thin) (@owned D) -> ()
      apply %useD_func(%d2) : $@convention(thin) (@owned D) -> ()              (8)

      strong_release %c : $C                                                   (10)
    }

Giving us the exact result that we want: Operation Sequence 2!

### Defining load [copy]

Given that we wish the load, store to be tightly coupled together, it is natural
to express this operation as a `load [copy]` instruction. Lets define the `load
[copy]` instruction as follows:

    %1 = load [copy] %0 : $*C

      =>

    %1 = load %0 : $*C
    retain_value %1 : $C

Now lets transform our initial example to use this instruction:

Notice how now if we move `(7)` over `(3)` and `(6)` now, we get the following SIL:

    sil @run : $@convention(thin) () -> () {
    bb0:
      %c = alloc_ref $C
      %global_c = global_addr @GLOBAL_C : $*C
      strong_retain %c : $C
      store %c to %global_c : $*C                                              (1)

      %d = alloc_ref $D
      %global_d = global_addr @GLOBAL_D : $*D
      strong_retain %d : $D
      store %d to %global_d : $*D                                              (2)

      %c2 = load [copy] %global_c : $*C                                        (3)
      %d2 = load [copy] %global_d : $*D                                        (5)

      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c2) : $@convention(thin) (@owned C) -> ()              (7)

      %useD_func = function_ref @useD : $@convention(thin) (@owned D) -> ()
      apply %useD_func(%d2) : $@convention(thin) (@owned D) -> ()              (8)

      strong_release %d : $D                                                   (9)
      strong_release %c : $C                                                   (10)
    }

We then perform the previous code motion:

    sil @run : $@convention(thin) () -> () {
    bb0:
      %c = alloc_ref $C
      %global_c = global_addr @GLOBAL_C : $*C
      strong_retain %c : $C
      store %c to %global_c : $*C                                              (1)

      %d = alloc_ref $D
      %global_d = global_addr @GLOBAL_D : $*D
      strong_retain %d : $D
      store %d to %global_d : $*D                                              (2)

      %c2 = load [copy] %global_c : $*C                                        (3)
      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c2) : $@convention(thin) (@owned C) -> ()              (7)
      strong_release %d : $D                                                   (9)

      %d2 = load [copy] %global_d : $*D                                        (5)
      %useD_func = function_ref @useD : $@convention(thin) (@owned D) -> ()
      apply %useD_func(%d2) : $@convention(thin) (@owned D) -> ()              (8)
      strong_release %c : $C                                                   (10)
    }

We then would like to eliminate `(9)` and `(10)` by pairing them with `(3)` and
`(8)`. Can we still do so? One way we could do this is by introducing the
`[take]` flag. The `[take]` flag on a `load [take]` says that one is
semantically loading a value from a memory location and are taking ownership of
the value thus eliding the retain. In terms of SIL this flag is defined as:

    %x = load [take] %x_ptr : $*C

      =>

    %x = load %x_ptr : $*C

Why do we care about having such a `load [take]` instruction when we could just
use a `load`? The reason why is that a normal `load` has unsafe unowned
ownership (i.e. it has no implications on ownership). We would like for memory
that has non-trivial type to only be able to be loaded via instructions that
maintain said ownership. We will allow for casting to trivial types as usual to
provide such access if it is required.

Thus we have achieved the desired result:

    sil @run : $@convention(thin) () -> () {
    bb0:
      %c = alloc_ref $C
      %global_c = global_addr @GLOBAL_C : $*C
      strong_retain %c : $C
      store %c to %global_c : $*C                                              (1)

      %d = alloc_ref $D
      %global_d = global_addr @GLOBAL_D : $*D
      strong_retain %d : $D
      store %d to %global_d : $*D                                              (2)

      %c2 = load [take] %global_c : $*C                                        (3)
      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c2) : $@convention(thin) (@owned C) -> ()              (7)

      %d2 = load [take] %global_d : $*D                                        (5)
      %useD_func = function_ref @useD : $@convention(thin) (@owned D) -> ()
      apply %useD_func(%d2) : $@convention(thin) (@owned D) -> ()              (8)
    }
