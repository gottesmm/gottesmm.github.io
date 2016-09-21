<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Summary](#summary)
- [Partial Initialization of Loadable References in SIL](#partial-initialization-of-loadable-references-in-sil)
- [Case Study: Partial Initialization and load_strong](#case-study-partial-initialization-and-loadstrong)
    - [The Problem](#the-problem)
    - [Defining load_strong](#defining-loadstrong)
- [Formal Definitions](#formal-definitions)
    - [load_strong](#loadstrong)
    - [store_strong](#storestrong)
- [Optimizer Changes](#optimizer-changes)
    - [load->load forwarding](#load-load-forwarding)
    - [store->load forwarding](#store-load-forwarding)

<!-- markdown-toc end -->

# Summary

This document proposes new instructions that simplify the representation of ARC
operations at the SIL level by eliminating the ability to perform partial
initialization of bindings to non-trivial values. Specifically, we propose here
two new SIL instructions:

1. load_strong: This instruction loads a stored value and then performs a
   retain_value upon the newly loaded value.
2. store_strong: This combines the operations of retaining the new value,
   loading the old value, storing the new value, and then releasing the old
   value.

Combining those operations into single SIL instructions allows the optimizer to
avoid miscompiles that can occur due to releases being moved into the partial
initialization region in between a load/retain, load/release. As a result of
this, we are able to be more aggressive when performing code motion of ARC
operations since we no longer have to worry about a release triggering a deinit
before ownership of an object is fully taken.

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

Define the following:

    var GLOBAL_C : $C
    var GLOBAL_D : $D

    final class C {
      var x: Int = 0

      deinit {
        opaque_call()
      }
    }

    final class D {
      var x: Int = 1
    }

    def c_user() -> ()
    def d_user() -> ()
    def opaque_store_into_GLOBAL_C(C) -> ()
    def opaque_store_into_GLOBAL_D(D) -> ()

Notice that both `C` and `D` have fixed layouts and separate class hierarchies,
but `C`'s deinit has a call to the function `opaque_call` which may write to
`GLOBAL_D` or `GLOBAL_C`. Additionally assume that both `c_user` and `d_user`
are known to the compiler to not have any affects on instances of type `D`,
`C` respectively. Now consider the function `foo()` in Swift:

    func foo() {
      let c_orig = C()
      let d_orig = D()

      opaque_store_into_GLOBAL_C(c_orig)
      opaque_store_into_GLOBAL_D(d_orig)

      let c = GLOBAL_C
      let d = GLOBAL_D

      c_user(c)
      d_user(d)
    }

This lowers to approximately the following SIL:

    sil_global GLOBAL_C : $C
    sil_global GLOBAL_D : $D

    final class C {
      var x: Int
      deinit
    }

    final class D {
      var x: Int
    }

    sil @c_user : $@convention(thin) () -> ()
    sil @d_user : $@convention(thin) () -> ()
    sil @opaque_store_into_GLOBAL_C : $@convention(thin) (@owned C) -> ()
    sil @opaque_store_into_GLOBAL_D : $@convention(thin) (@owned D) -> ()

    sil @foo : $@convention(thin) () -> () {
    bb0():
      // Allocate new instances of C, D.
      %c_orig = alloc_ref $C
      %d_orig = alloc_ref $D

      // Opaquely initialize GLOBAL_C, GLOBAL_D.
      strong_retain %c_orig : $C
      %c_global_init_fun = function_ref @opaque_store_into_GLOBAL_C : $@convention(thin) (@owned C) -> ()
      apply %c_global_init_fun(%c_orig) : $@convention(thin) (@owned C) -> ()

      strong_retain %d_orig : $D
      %d_global_init_fun = function_ref @opaque_store_into_GLOBAL_D : $@convention(thin) (@owned D) -> ()
      apply %d_global_init_fun(%d_orig) : $@convention(thin) (@owned D) -> ()

      // Reload from GLOBAL_C and GLOBAL_D
      %global_c_addr = global_addr GLOBAL_C : $C
      %c = load %global_c_addr : $*C             (1)
      strong_retain %c : $C                      (2)
      %global_d_addr = global_addr GLOBAL_D : $D
      %d = load %global_d_addr : $*D             (3)
      strong_retain %d : $D                      (4)

      // Call the user functions passing in the instances that we loaded from the globals.
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (5)
      %d_user_fn = function_ref @d_user : $@convention(thin) (@owned D) -> ()
      apply %d_user_fn(%d) : $@convention(thin) (@owned D) -> ()                (6)

      // Deallocate the value from the original allocation.
      strong_release %c_orig : $C        (7)
      strong_release %d_orig : $D        (8)

      ...
    }

Lets optimize this function! First we perform the following operations:

1. By assumption, we know that `%c_user_fn` does not perform any releases of
   `%d`. Thus `(4)` can be moved past `(5)`.
2. Since we know that loading from a global or retaining a value does not cause
   any reference counts to be decremented, we can move `(2)` right before `(5)`.
3. Since we do not allow for any dependencies in between deinits, we are also
   allowed to swap the order of `(7)`, `(8)`.

After performing such transformations, `foo` looks as follows:

    sil @foo : $@convention(thin) () -> () {
    bb0():
      // Allocate new instances of C, D.
      %c_orig = alloc_ref $C
      %d_orig = alloc_ref $D

      // Opaquely initialize GLOBAL_C, GLOBAL_D.
      strong_retain %c_orig : $C
      %c_global_init_fun = function_ref @opaque_store_into_GLOBAL_C : $@convention(thin) (@owned C) -> ()
      apply %c_global_init_fun(%c_orig) : $@convention(thin) (@owned C) -> ()

      strong_retain %d_orig : $D
      %d_global_init_fun = function_ref @opaque_store_into_GLOBAL_D : $@convention(thin) (@owned D) -> ()
      apply %d_global_init_fun(%d_orig) : $@convention(thin) (@owned D) -> ()

      // Reload from GLOBAL_C and GLOBAL_D
      %global_c_addr = global_addr GLOBAL_C : $C
      %c = load %global_c_addr : $*C             (1)
      %global_d_addr = global_addr GLOBAL_D : $D
      %d = load %global_d_addr : $*D             (3)

      // Call the user functions passing in the instances that we loaded from the globals.
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      strong_retain %c : $C                      (2)
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (5)

      %d_user_fn = function_ref @d_user : $@convention(thin) (@owned D) -> ()
      strong_retain %d : $D                      (4)
      apply %d_user_fn(%d) : $@convention(thin) (@owned D) -> ()                (6)

      // Deallocate the value from the original allocation.
      strong_release %d_orig : $D        (8)
      strong_release %c_orig : $C        (7)

      ...
    }

Then as per the ARC optimization model, we are then able to remove `(4)` and
`(8)` yielding:

    sil @foo : $@convention(thin) () -> () {
    bb0():
      // Allocate new instances of C, D.
      %c_orig = alloc_ref $C
      %d_orig = alloc_ref $D

      // Opaquely initialize GLOBAL_C, GLOBAL_D.
      strong_retain %c_orig : $C
      %c_global_init_fun = function_ref @opaque_store_into_GLOBAL_C : $@convention(thin) (@owned C) -> ()
      apply %c_global_init_fun(%c_orig) : $@convention(thin) (@owned C) -> ()

      strong_retain %d_orig : $D
      %d_global_init_fun = function_ref @opaque_store_into_GLOBAL_D : $@convention(thin) (@owned D) -> ()
      apply %d_global_init_fun(%d_orig) : $@convention(thin) (@owned D) -> ()

      // Reload from GLOBAL_C and GLOBAL_D
      %global_c_addr = global_addr GLOBAL_C : $C
      %c = load %global_c_addr : $*C             (1)
      %global_d_addr = global_addr GLOBAL_D : $D
      %d = load %global_d_addr : $*D             (3)

      // Call the user functions passing in the instances that we loaded from the globals.
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      strong_retain %c : $C                      (2)
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (5)

      %d_user_fn = function_ref @d_user : $@convention(thin) (@owned D) -> ()
      apply %d_user_fn(%d) : $@convention(thin) (@owned D) -> ()                (6)

      // Deallocate the value from the original allocation.
      strong_release %c_orig : $C        (7)

      ...
    }

Now by assumption, we know that `%d_user_fn` does not touch any instances of
class `C` and `%c` does not contain any ivars of type `%d` and is final so none
can be added. At first glance, this seems to suggest that we can move `(7)`
before `(6)` and thus shorten the lifetime of `%c` and then (potentially)
eliminate a retain `(2)`, release `(7)` pair. But is this a safe optimization
perform? Absolutely Not! Why? The destructor of `%c` calls an opaque function
that _could_ potentially write to `%global_d_addr`. If that does occur, we will
be passing `%d`, an already deallocated object to `%d_user_fn`, an illegal
optimization!

Lets think a bit more about this example and consider this example at the
language level. Remember that while Swift's deinit are not asychronous, we do
not allow for user level code to create dependencies from the body of the
destructor into the normal control flow that has called it. This means that
there are two valid results of this code:

- Operation Sequence 1: No optimization is performed and `%d` is passed to `%d_user_fn`.
- Operation Sequence 2: We shorten the lifetime of `%c` before `%d_user_fn` and
   a different instance of `$D` is passed into `%d_user_fn`.

The fact that 1 occurs without optimization is just as a result of an
implementation detail of SILGen. 2 is also a valid sequence of operations.

Given that:

1. As a principle, the optimizer does not consider such dependencies to avoid
   being overly conservative.
2. We provide constructs to ensure appropriate lifetimes via the usage of
   constructs such as fix_lifetime.

we need to figure out how to express our optimization such that 2
happens. Remember that one of the first optimizations that we performed at the
beginning was to move `(4)` over `(5)`, i.e., transform this:

      %global_d_addr = global_addr GLOBAL_D : $D
      %d = load %global_d_addr : $*D             (3)
      strong_retain %d : $D                      (4)

      // Call the user functions passing in the instances that we loaded from the globals.
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (5)

into:

      %global_d_addr = global_addr GLOBAL_D : $D
      %d = load %global_d_addr : $*D             (3)

      // Call the user functions passing in the instances that we loaded from the globals.
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (5)
      strong_retain %d : $D                      (4)

This transformation in Swift corresponds to transforming:

      let d = GLOBAL_D
      c_user(c)

to:

      let d_raw = load_d_value(GLOBAL_D)
      c_user(c)
      let d = take_ownership_of_d(d_raw)

This is clearly an instance where we have moved a side-effect in between the
loading of the data and the taking ownership of such data, that is before the
`let` is fully initialized. What if instead of just moving the retain, we moved
the entire let statement? This would then result in the following swift code:

      c_user(c)
      let d = GLOBAL_D

and would correspond to the following Swift snippet:
 
      %global_d_addr = global_addr GLOBAL_D : $D

      // Call the user functions passing in the instances that we loaded from the globals.
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (5)
      %d = load %global_d_addr : $*D             (3)
      strong_retain %d : $D                      (4)     

Moving the load with the strong_retain to ensure that the full initialization is
performed even after code motion causes our final SIL to look as follows:

    sil @foo : $@convention(thin) () -> () {
    bb0():
      // Allocate new instances of C, D.
      %c_orig = alloc_ref $C
      %d_orig = alloc_ref $D

      // Opaquely initialize GLOBAL_C, GLOBAL_D.
      strong_retain %c_orig : $C
      %c_global_init_fun = function_ref @opaque_store_into_GLOBAL_C : $@convention(thin) (@owned C) -> ()
      apply %c_global_init_fun(%c_orig) : $@convention(thin) (@owned C) -> ()

      strong_retain %d_orig : $D
      %d_global_init_fun = function_ref @opaque_store_into_GLOBAL_D : $@convention(thin) (@owned D) -> ()
      apply %d_global_init_fun(%d_orig) : $@convention(thin) (@owned D) -> ()

      // Reload from GLOBAL_C and GLOBAL_D
      %global_c_addr = global_addr GLOBAL_C : $C
      %c = load %global_c_addr : $*C             (1)
      %global_d_addr = global_addr GLOBAL_D : $D

      // Call the user functions passing in the instances that we loaded from the globals.
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      strong_retain %c : $C                      (2)
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (5)

      %d = load %global_d_addr : $*D             (3)
      %d_user_fn = function_ref @d_user : $@convention(thin) (@owned D) -> ()
      apply %d_user_fn(%d) : $@convention(thin) (@owned D) -> ()                (6)

      // Deallocate the value from the original allocation.
      strong_release %c_orig : $C        (7)

      ...
    }

Notice how now if we move `(7)` over `(3)` and `(6)` now, we get the following SIL:

    sil @foo : $@convention(thin) () -> () {
    bb0():
      // Allocate new instances of C, D.
      %c_orig = alloc_ref $C
      %d_orig = alloc_ref $D

      // Opaquely initialize GLOBAL_C, GLOBAL_D.
      strong_retain %c_orig : $C
      %c_global_init_fun = function_ref @opaque_store_into_GLOBAL_C : $@convention(thin) (@owned C) -> ()
      apply %c_global_init_fun(%c_orig) : $@convention(thin) (@owned C) -> ()

      strong_retain %d_orig : $D
      %d_global_init_fun = function_ref @opaque_store_into_GLOBAL_D : $@convention(thin) (@owned D) -> ()
      apply %d_global_init_fun(%d_orig) : $@convention(thin) (@owned D) -> ()

      // Reload from GLOBAL_C and GLOBAL_D
      %global_c_addr = global_addr GLOBAL_C : $C
      %c = load %global_c_addr : $*C             (1)
      %global_d_addr = global_addr GLOBAL_D : $D

      // Call the user functions passing in the instances that we loaded from the globals.
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      strong_retain %c : $C                      (2)
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (5)
      strong_release %c_orig : $C        (7)

      %d = load %global_d_addr : $*D             (3)
      %d_user_fn = function_ref @d_user : $@convention(thin) (@owned D) -> ()
      apply %d_user_fn(%d) : $@convention(thin) (@owned D) -> ()                (6)

      ...
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

    sil @foo : $@convention(thin) () -> () {
    bb0():
      ...

      // Reload from GLOBAL_C and GLOBAL_D
      %global_c_addr = global_addr GLOBAL_C : $C
      %c = load_strong %global_c_addr : $*C      (1)
      %global_d_addr = global_addr GLOBAL_D : $D
      %d = load_strong %global_d_addr : $*D      (2)

      // Call the user functions passing in the instances that we loaded from the globals.
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (3)
      %d_user_fn = function_ref @d_user : $@convention(thin) (@owned D) -> ()
      apply %d_user_fn(%d) : $@convention(thin) (@owned D) -> ()                (4)

      // Deallocate the value from the original allocation.
      strong_release %c_orig : $C        (5)
      strong_release %d_orig : $D        (6)

      ...
    }

We then perform the previous code motion:

    sil @foo : $@convention(thin) () -> () {
    bb0():
      ...

      // Reload from GLOBAL_C and GLOBAL_D
      %global_c_addr = global_addr GLOBAL_C : $C
      %global_d_addr = global_addr GLOBAL_D : $D

      // Call the user functions passing in the instances that we loaded from the globals.
      %c = load_strong %global_c_addr : $*C      (1)
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (3)
      strong_release %c_orig : $C        (5)

      %d = load_strong %global_d_addr : $*D      (2)
      %d_user_fn = function_ref @d_user : $@convention(thin) (@owned D) -> ()
      apply %d_user_fn(%d) : $@convention(thin) (@owned D) -> ()                (4)
      strong_release %d_orig : $D        (6)

      ...
    }

We then would like to eliminate `(7)` and `(8)`. Can we still do so? The way
this is done by introducing the `[take]` flag. The `[take]` flag on a
load_strong says that one is semantically loading a value from a memory
location, but are taking over ownership of that value and thus eliding the
retain. In terms of SIL this flag is defined as:

    %c = load_strong [take] %c_ptr : $*C

      =>

    %c = load %c_ptr : %*C

Why do we care about having such a `load_strong [take]` instruction when we
could just use a `load`? The reason why is that to ensure correctness in
ownership in the compiler, we wish to create a requirement that `load` has a
result with `@unsafe @unowned` ownership, while `load_strong [take]` has an
`@owned` ownership. Even without Ownership at the SIL level, we can still
enforce some of this invariant by in the SILVerifier saying that a load can not
be applied to memory locations that are not of trivial type.

Thus when ARC optimization is run, we perform pairings and get back a version of
the final SIL that we wanted: 

    sil @foo : $@convention(thin) () -> () {
    bb0():
      ...

      // Reload from GLOBAL_C and GLOBAL_D
      %global_c_addr = global_addr GLOBAL_C : $C
      %global_d_addr = global_addr GLOBAL_D : $D

      // Call the user functions passing in the instances that we loaded from the globals.
      %c = load_strong [take] %global_c_addr : $*C      (1)
      %c_user_fn = function_ref @c_user : $@convention(thin) (@owned C) -> ()
      apply %c_user_fn(%c) : $@convention(thin) (@owned C) -> ()                (3)

      %d = load_strong [take] %global_d_addr : $*D      (2)
      %d_user_fn = function_ref @d_user : $@convention(thin) (@owned D) -> ()
      apply %d_user_fn(%d) : $@convention(thin) (@owned D) -> ()                (4)

      ...
    }

# Formal Definitions

## load_strong

Following the lead of load_unowned and load_weak, we allow for a [take] flag to
be applied to the instruction implying that we have two forms. The full
load_strong is defined as follows:

    %x = load_strong %x_ptr : $*C

      =>
  
    %x = load %x_ptr : $*C
    retain_value %x : $C

Since a load_strong is meant to be viewed as a manner of creating a new
reference to a stored value, it can not be reduced to a load, retain until 

    %x = load_strong [take] %x_ptr : $*Builtin.NativeObject

      => 

    %x = load %x_ptr : $*Builtin.NativeObject

## store_strong

Now define a store_strong as follows:

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

# Optimizer Changes

The main optimizer changes can be separated into the following areas: load->load
forwarding, dead store elimination, store->load forwarding, ARC
optimization, mem2reg. We go through each optimization below:

## load->load forwarding

Currently we perform load -> load forwarding as follows:

    %x = load %x_ptr : $C
    ... NO SIDE EFFECTS THAT TOUCH X_PTR ...
    %y = load %x_ptr : $C
    use(%y)

      =>
    
    %x = load %x_ptr : $C
    ...
    use(%x)

In a load_strong world, the main implication is that the second load from
`%x_ptr` is not just a load, but also the creation of a new ownership
state. This means that if we are performing a full `load_strong`, we must insert
a retain and if we are performing a `load_strong [take]`, we just forward as
before. Thus:

    %x = load_strong %x_ptr : $C
    ... NO SIDE EFFECTS THAT WRITE TO X_PTR ...
    %y = load_strong %x_ptr : $C
    readonly_use1(%y)
    ... NO SIDE EFFECTS THAT WRITE TO X_PTR ...
    %z = load_strong [take] %x_ptr : $C
    readonly_use2(%z)

      =>

    %x = load_strong %x_ptr : $C
    ... NO SIDE EFFECTS THAT WRITE TO X_PTR ...
    strong_retain %x
    readonly_use1(%x)
    ... NO SIDE EFFECTS THAT WRITE TO X_PTR ...
    readonly_use2(%x)

## store->load forwarding

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

Then for `store_strong` and `load_strong [take]`:

    store_strong %x to %x_ptr : $C
    ... NO SIDE EFFECTS THAT TOUCH X_PTR ...
    %y = load_strong [take] %x_ptr : $C
    use(%y)

      =>

    store_strong %x to %x_ptr : $C
    ... NO SIDE EFFECTS THAT TOUCH X_PTR ...
    use(%x)

## Dead Store Elimination

Dead store elimination is changed as follows:

1. If the dead `store_strong` is a full `store_strong`, we just delete the old
   dead_store.
2. If the dead `store_strong` is a `store_strong [init]`, then the
post-dominating store 
Dead store elimination works just like before, except that the dead store's
attribute must be taken on by the post-dominating store. I.e.:

    store_strong %x to [init] %x_ptr : $C
    store_strong %y to %x_ptr : $C

       =>
    
    store_strong %y to [init] %x_ptr : $C
    
