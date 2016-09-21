<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Summary](#summary)
- [Partial Initialization of Loadable References in SIL](#partial-initialization-of-loadable-references-in-sil)
- [Case Study: Partial Initialization and load_strong](#case-study-partial-initialization-and-loadstrong)
    - [The Problem](#the-problem)
    - [Defining load_strong](#defining-loadstrong)
- [Formal Definitions](#formal-definitions)
    - [loads_strong](#loadsstrong)
    - [store_strong](#storestrong)
- [Implementation Strategy](#implementation-strategy)
    - [Initial Bringup](#initial-bringup)
    - [Optimizer Changes](#optimizer-changes)
        - [store->load forwarding](#store-load-forwarding)
        - [ARC Code Motion](#arc-code-motion)

<!-- markdown-toc end -->

# Summary

This document proposes new instructions that simplify the representation of ARC
operations at the SIL level by:

1. Eliminating the ability to perform partial initialization of bindings to
non-trivial values.
2. Eliminating the ability to perform unsafe unowned memory operations on
   non-trivially typed memory.

Specifically, we propose here two new SIL instructions:

1. load_strong: This instruction loads a stored value and then performs a
   retain_value upon the newly loaded value.
2. store_strong: This combines the operations of retaining the new value,
   loading the old value, storing the new value, and then releasing the old
   value.

Combining those operations into single SIL instructions allows:

1. The optimizer to avoid miscompiles that can occur due to releases being moved
   into the partial initialization region in between a load/retain, load/release.
2. The SIL verifier to ban the usage of unsafe unowned loads from memory
   locations with non-trivial type.

As a result of this:

1. More aggressive ARC code motion can be performed since releases can no longer
trigger a deinit before ownership of an object is fully taken.
2. SIL will be able to via a use-def list represent the loading/gaining
   ownership of a value loaded from memory in a robust manner.

We elaborate on the partial initialization point in depth below since it may not
be obvious to readers:

# Partial Initialization of Loadable References in SIL

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
two concrete examples that show how semantically providing load_strong,
store_strong instructions eliminate this problem.

**NOTE** Without any loss of generality, we will speak of values with reference
semantics instead of non-trivial values.

# Case Study: Partial Initialization and load_strong

## The Problem

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

## Defining load_strong

Given that we wish the load, store to be tightly coupled together, it is natural
to express this operation as a `load_strong` instruction. Lets define the
`load_strong` instruction as follows:

    %1 = load_strong %0 : $*C

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

      %c2 = load_strong %global_c : $*C                                        (3)
      %d2 = load_strong %global_d : $*D                                        (5)

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

      %c2 = load_strong %global_c : $*C                                        (3)
      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c2) : $@convention(thin) (@owned C) -> ()              (7)
      strong_release %d : $D                                                   (9)

      %d2 = load_strong %global_d : $*D                                        (5)
      %useD_func = function_ref @useD : $@convention(thin) (@owned D) -> ()
      apply %useD_func(%d2) : $@convention(thin) (@owned D) -> ()              (8)
      strong_release %c : $C                                                   (10)
    }

We then would like to eliminate `(9)` and `(10)` by pairing them with `(3)` and
`(8)`. Can we still do so? One way we could do this is by introducing the
`[take]` flag. The `[take]` flag on a load_strong says that one is semantically
loading a value from a memory location and are taking ownership of the value
thus eliding the retain. In terms of SIL this flag is defined as:

    %x = load_strong [take] %x_ptr : $*C

      =>

    %x = load %x_ptr : $*C

Why do we care about having such a `load_strong [take]` instruction when we
could just use a `load`? The reason why is that a normal `load` has unsafe
unowned ownership (i.e. it has no implications on ownership). We would like for
memory that has non-trivial type to only be able to be loaded via instructions
that maintain said ownership. We will allow for casting to trivial types as
usual to provide such access if it is required.

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

      %c2 = load_strong [take] %global_c : $*C                                 (3)
      %useC_func = function_ref @useC : $@convention(thin) (@owned C) -> ()
      apply %useC_func(%c2) : $@convention(thin) (@owned C) -> ()              (7)

      %d2 = load_strong [take] %global_d : $*D                                 (5)
      %useD_func = function_ref @useD : $@convention(thin) (@owned D) -> ()
      apply %useD_func(%d2) : $@convention(thin) (@owned D) -> ()              (8)
    }

# Formal Definitions

## loads_strong

Define load_strong as follows:

    %x = load_strong %x_ptr : $*C

      =>

    %x = load %x_ptr : $*C
    retain_value %x : $C

We allow for a `[take]` flag to be applied to the `load_strong`:

    %x = load_strong [take] %x_ptr : $*Builtin.NativeObject

      =>

    %x = load %x_ptr : $*Builtin.NativeObject

The take implies that the loaded value is no longer owned by the memory location
(i.e. it is a move). One concern here is are we able to truly guarantee that no
other value will load from the given memory location given potential instruction
re-ordering. Given this concern, we also will allow a `[guaranteed]` flag:

    %x = load_strong [guaranteed] %x_ptr : $*Builtin.NativeObject
    ...
    lifetime_barrier %x : $Builtin.NativeObject

      =>

    %x = load %x_ptr : $*Builtin.NativeObject
    ...
    lifetime_barrier(%x)

The `[guaranteed]` flag will enable us to express that we are loading this value
unconditionally without retaining it and that even though the object's lifetime
_may_ end before lifetime_barrier, the optimizer is not allowed to cause the
object's lifetime if it ends after the lifetime_barrier to end before the
lifetime_barrier. It remains to be seen if this becomes an issue, but if it
does, this simple approach can fix the problem. *NOTE* The reason why we can not use a
fix lifetime here is that a fix lifetime implies that the object's lifetime can
not end earlier than the fix lifetime instruction.

## store_strong

Define a store_strong as follows:

    store_strong %x to %x_ptr : $*C

       =>

    %old_x = load %x_ptr : $*C
    retain_value %new_x : $C
    store %new_x to %x_ptr : $*C
    release_value %old_x : $C

We also provide an init flag in the case where we know statically that there is
no previous value in the memory location:

    store_strong %x to [init] %x_ptr : $*C

       =>

    retain_value %new_x : $C
    store %new_x to %x_ptr : $*C

# Implementation Strategy

## Initial Bringup

The initial bringup strategy here is:

1. A new flag will be added to SILOptions: "EnableSILOwnershipModel". This flag
   will be disabled by default and will able to be triggered by passing into the
   frontend the -enable-sil-ownership-model flag.

2. Bots will be brought up to test the compiler with this flag
   enabled. Specifically, we will create a separate RA-OSX+simulatrs, RA-Device,
   RA-Linux. These will run once a day during bring up. As we get closer to
   turning on this code, we will run it in a polling configuration.

3. load_strong, store_strong will be implemented in SIL but will not be emitted
   by SILGen. IRGen/serialization/printing/parsing support will be implemented.

4. A small pass called the "OwnershipModelEliminator" will be
   implemented. Initially it will just to blow up load_strong/store_strong into
   their constituent operations. It will serve as a grab bag pass to eliminate
   parts of the SIL Ownership Model from the IR to ensure that we do not lose
   performance while bringing up this code.

5. The verifier will have a disabled by default option called
   "EnforceSILOwnershipModel" added to it. If the option is set, the verifier
   will verify that unsafe unowned loads, stores are not used to load from
   non-trivial memory locations. This flag will be used to trigger further
   verification at later stages of the implementation of the SIL Ownership model
   as well.

6. SILGen will be changed to emit load_strong, store_strong instructions when
   the EnableSILOwnershipModel flag is set.

7. The pass manager will be changed such that if the EnableSILOwnershipModel
   flag is set, the verifier that is run right after SILGen will have the
   EnforceSILOwnershipModel flag set. Then once ownership verification is
   complete, the OwnershipModelEliminator pass will be run and then the normal
   pipeline.

8. Once the compiler is able to pass the bots with the
   "-enable-sil-ownership-model" flag enabled, we will turn that flag on by
   default on trunk and after a cooling down period, move all of the
   aforementioned code in front of the flag. After that point, we will reuse
   said flag for other stages of the implementation of the Ownership Model.

## Optimizer Changes

Since the SILOwnershipModel eliminator will eliminate the load_strong,
store_strong instructions right after ownership verification, there will be no
immediate affects on the optimizer and thus the optimizer changes can be done in
parallel with the rest of the ARC optimization work.

But, in the long run, we need IRGen to eliminate the load_strong, store_strong
instructions, not the SILOwnershipModel eliminator, so that we can enforce
Ownership invariants all through the SIL pipeline. Thus we will need to update
passes to handle these new instructions. The main optimizer changes can be
separated into the following areas: memory forwarding, dead stores, ARC
optimization. In all of these cases, the necessary changes are relatively
trivial to respond to. We give a quick taste of two of them: store->load
forwarding and ARC Code Motion.

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

In a world, where we are using load_strong, store_strong, we have to also
consider the ownership implications. *NOTE* Since we are not modifying the
store_strong, `store_strong` and `store_strong [init]` are treated the
same. Thus without any loss of generality, lets consider solely `store_strong`.

    store_strong %x to %x_ptr : $C
    ... NO SIDE EFFECTS THAT TOUCH X_PTR ...
    %y = load_strong %x_ptr : $C
    use(%y)

      =>

    store_strong %x to %x_ptr : $C
    ... NO SIDE EFFECTS THAT TOUCH X_PTR ...
    strong_retain %x
    use(%x)

### ARC Code Motion

The main implication for ARC Code Motion is that we can no longer move the ARC
operations implied by load_strong, store_strong separately from said
instructions. This means that when we want to move these instructions since
there is now a load/store involved, we must also consider the read/writes to
memory.

### ARC Optimization

The main implication for ARC optimization is that instead of eliminating just
retains, releases, it must be able to recognize load_strong/store_strong and set
their flags as appropriate.

### Function Signature Optimization

The main implication for FSO is that FSO must be able to recognize a `store strong`
as releasing. Then it will change the `store_strong` to be a `store_store
[init]` and move the release into the caller.
