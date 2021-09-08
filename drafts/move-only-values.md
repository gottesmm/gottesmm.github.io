---
layout: page
title: Move Only Values
categories: draft
---

This document contains a proposal to add a new ``@_moveOnly`` attribute to
Swift. This will be allowed upon lets, vars, arguments, results, computed
properties, and fields of class types. This will allow for System programmers to
have more exact control of the performance of their programs. The general
outline of our discussion will be:

1. First we introduce the Swift level syntax for the ``@_moveOnly`` attributes
   and its implications on the Swift level.

1. We then discuss how when SIL is OSSA form, move only values are naturally
   represented as values that due to their type are unable to be passed to copy
   instructions. We will show how we can use the SIL level type system to
   maintain this property by using the type system. We will also show how we can
   handle trivial and address only types with this scheme.

2. Then we will talk about how this goal of representing move only values as
   uncopyable values conflicts with the current SILGen implementation since it
   often times is forced to emit copies. We will introduce a strategy for
   working around this problem using a new diagnostic pass called **Diagnostic
   Copy Propagation** that lets us emit copies of move only values in SILGen and
   then remove those copies as a SIL pass (and emit an error for any that can
   not be removed).

3. Then we will introduce the move operator that we will use to end lifetimes of
   objects at the Swift level. This will require a different diagnostic pass
   than Diagnostic Copy Propagation.

4. Finally, we will conclude by proposing a bring up strategy for this feature.

## The `@_moveOnly` attribute

In order to represent the `@_moveOnly` concept at the Swift level, we use the `@_moveOnly` attribute and allow for it to be placed on let, var bindings, function arguments, return values, as well as class fields. Example:

```
@_moveOnly let global: Int = ...
@_moveOnly var global2: Klass = ...

class X {
  @_moveOnly let x: Int = ...
}

func foo(@_moveOnly _ x: Klass) -> @_moveOnly Klass {
  // This is going to be a copy since y isn't marked @_moveOnly.
  let y = x
  // This is a move of x into z and x can no longer be used afterwards.
  @_moveOnly let z = x
  ...
}
```

NOTE: We do not allow for struct fields to be marked `@_moveOnly` since that
would necessarily imply that the struct itself is a move only type (which we do
not support yet).

Importantly, we do not actually represent `@_moveOnly` in the type system at the
Swift level. This ensures that the type checker does not need to know about
`@_moveOnly` and avoids the many implementation issues around the type checker
that we discovered with inout types. Instead we:

* Put a bit in TypeBase that signifies that a type is moveOnly. We do not allow
  for this bit to be set by the type checker and one can not create such a type
  at the AST level.

* Rely on SILGen and TypeLowering to as appropriate produce values with the
  moveOnly type so we can enforce no-copying at the SIL level.

Now that we understand how we will represent move only values at the Swift
level, lets consider the design space below at the SIL level.

## Representing Move Only Values in SIL

**NOTE:** In the following I am assuming that one has read the [Ownership SSA documentation](https://github.com/apple/swift/blob/main/docs/SIL.rst#id42) in SIL.rst.

In the following, I first discuss the natural form for representing move only
values in SIL. Then I talk about how we extend this model from non-trivial
loadable values to trivial loadable values and address only types.

**NOTE:** In the following section, I am going to use a straw man type attribute
(`@_moveOnly`) to signal that a value (even if normally not move only) is move
only. This will just act as a bit on a type at the SIL level.

#### Move Only Values in OSSA

In order for us to represent move only values, we need to first consider what
their form looks like in SIL. Put simply copies in SIL are represented via
special instructions (e.x.: `copy_value`) and a move only value in SIL is a SSA
value that is never passed to such an instruction. Instead, we manipulate the
value by using forwarding instructions, consuming parameters, and consuming
results. Example:

```
sil @_moveOnly_only_value_example : $@convention(thin) (@owned @_moveOnly Optional<Klass>) -> () {
bb0(%0 : @owned $@_moveOnly Optional<Klass>):
  // Inserting a copy_value of %0 will cause the IR verifier to assert!
  switch_enum %0 : $@_moveOnly Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2

bb1(%1 : @owned $@_moveOnly Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@owned @_moveOnly SubKlass) -> ()
  %2 = unchecked_ref_cast %1 : $@_moveOnly Klass to $@_moveOnly SubKlass
  apply %f(%2) : $@convention(thin) (@owned @_moveOnly SubKlass) -> ()
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

Naturally, one would in Ownership SSA form ban any value of such a type being an
operand to any copy instruction, guaranteeing the property that we wish to
preserve.

For memory, we can use a similar methodology, banning instructions like
``copy_addr`` from copying a "move only value". Example:

```
sil @_moveOnly_only_value_memory_example : $@convention(thin) (@owned @_moveOnly Optional<Klass>) -> () {
bb0(%0 : @owned $@_moveOnly Optional<Klass>):
  %1 = alloc_stack $@_moveOnly Optional<Klass>

  // Store using a move...
  store %0 to [init] %1 : $*@_moveOnly Optional<Klass>

  // load [take] is legal here. A load [copy] would be illegal and would cause the IR
  // verifier to assert.
  %3 = load [take] %1 : $*@_moveOnly Optional<Klass>
  store %3 to [init] %1 : $*@_moveOnly Optional<Klass>

  %2 = alloc_stack $@_moveOnly Optional<Klass>
  // IR verifier would assert if this did not have a [take].
  copy_addr [take] %1 to [initialization] %2 : $*@_moveOnly Optional<Klass>
  ...
}
```

#### Modeling Address Only and Trivial Move Only Values

The model above naturally flows from how non-trivial loadable values are
represented in OSSA. That being said, we must consider two other types of values
that do not fit into that bucket today but that we must support: address only
values and trivial loadable values.

* Address Only Move Only Values. Since we want to fully take advantage of
Ownership SSA and its invariants, we naturally will rely upon opaque values to
ensure that beyond temporaries generated by SILGen, move only values will be
passed around as loadable values:

```
sil [ossa] @opaque_value_move_only : $@convention(thin) <T> (@in @_moveOnly T) -> @out @_moveOnly T {
bb0(%0 : @owned $@_moveOnly T):
  %3 = function_ref @opaque_copy : $@convention(thin) <T> (@in_guaranteed @_moveOnly T) -> @out @_moveOnly T
  %4 = apply %3<T>(%2) : $@convention(thin) <T> (@in_guaranteed @_moveOnly T) -> @out @_moveOnly T
  destroy_value %0 : $T
  return %4 : $T
}
```

* Trivial Move Only Values: In order for us to represent trivial move only
values in SIL, we must work around the invariant in SIL that only non-trivial
values can be copied. In order to prevent breaking these invariants, we will
introduce a new SIL instruction that can be used to convert a trivial value into
a non-trivial value by adding the ``@_moveOnly`` attribute to the type. As a straw
man (since we have not talked about `unique` yet), we will call the instruction
``make_move_only`` in the following example. In order to ensure that we do not
hurt performance, we will lower away the ``make_move_only`` instruction when we
lower Ownership SSA allowing for normal optimization of trivial values to
occur. Example:

```
sil [ossa] @trivial_value_move_only : $@convention(thin) (Int) -> () {
bb0(%0 : $Int):
  // %1 is a non-trivial value of type $@_moveOnly Int
  %1 = make_moveOnly %0 : $Int

  %f = function_ref @trivial_use : $@convention(thin) (Int) -> ()
  // We can only pass %0 (not %1) to %f since %f expects an Int, not an @_moveOnly Int.
  apply %f(%0) : $@convention(thin) (Int) -> ()

  %f2 = function_ref @trivial_move_only_use : $@convention(thin) (@owned @_moveOnly Int) -> ()
  // We can pass both %0 and %1 to %f2 since we allow for $Int to be passed as an @owned @_moveOnly Int value.
  apply %f2(%0) : $@convention(thin) (@owned @_moveOnly Int) -> ()
  apply %f2(%1) : $@convention(thin) (@owned @_moveOnly Int) -> ()
  ...
}
```

By using opaque values and the ``make_moveOnly`` instruction, we are able to
recast introducing move only values for these two sorts of values in terms of
non-trivial loadable values simplifying the implementation.

## SILGen, the "Ensure Plus One Problem", and Copying Move Only Values

#### Problem: SILGen assumes all values are copyable

A characteristic of SILGen today is that parts of SILGen have been written to
assume +0 and others +1 values causing SILGen to have to transition in between
such contexts. The natural way to do so is to introduce additional copies into
the IR using APIs such as ManagedValue::ensurePlusOne() or creating new
ManagedValues without cleanups. This creates a significant problem for bringing
up move only types since without significant engineering work, we /cannot/
guarantee that SILGen will not insert copies on a move only value, directly
conflicting with the design of move only types at the SIL level that we want to
achieve as described above. Instead, we must be creative and come up with a
different approach to solve our problem that /works around/ SILGen's current
behavior. Luckily for us a recent technical advance in OSSA SIL can help us to
escape from our predicament: Copy Propagation.

#### Solution: Use Copy Propagation!

As a result of implementing Ownership SSA (OSSA) in SIL, the Swift compiler can
now infer correct lifetimes at compile time of loadable typed values based off
of the SSA uses of that value. The inferred lifetimes are able to be statically
verified by the compiler as being OSSA correct (1) and allows the compiler to
guarantee that a programs ARC traffic will be the local minimal set of
operations needed to express the programs semantics. This is done by finding all
of the places where the value has lifetime ending uses (consuming uses) and then
determining the appropriate places where a copy would be needed if a lifetime
ending use is reachable from another lifetime ending use. These are exactly the
places where a programmer would need to manually insert an explicit copy when
using a move only type! This transformation is called **Copy Propagation** and
is currently implemented just for optimization purposes in a SILOptimizer pass
called **Performance Copy Propagation**. To get a visual sense of how this pass
works in action, see the section below called [Performance Copy Propagation in
Action](#reference-performance-copy-propagation-in-action). The authors have
refactored the underlying implementation into a prototype diagnostic pass called
**Diagnostic Copy Propagation** that we can use to implement support for move
only values in Swift!

To do so, we allow for values with the "moveOnly" bit set to be copied when we
are in Raw SIL. We run Diagnostic Copy Propagation on move only arguments, apply
results, and the result of all ``make_moveOnly`` instructions. The pass will
eliminate any of the extra copies that SILGen inserts and will emit errors in
any situation where semantically a copy is required to make the IR correct. Example:

```
sil [ossa] @flag_double_consume_of_move_value : $@convention(thin) (@owned Klass) -> () {
bb0(%0 : @owned $Klass):
  %1 = make_moveOnly %0 : $Klass // expected-error \{\{'x' consumed more than once}}
  debug_value %1 : $@_moveOnly Klass, let, name "x"
  %2 = copy_value %1 : $@_moveOnly Klass
  %f = function_ref @consume_move_only_value : $@convention(thin) (@owned @_moveOnly Klass) -> ()
  // expected-note @-1 \{\{consuming use}}
  apply %f(%2) : $@convention(thin) (@owned @_moveOnly Klass) -> ()
  // expected-note @-1 \{\{consuming use}}
  apply %f(%1) : $@convention(thin) (@owned @_moveOnly Klass) -> ()
  %9999 = tuple()
  return %9999 : $()
}
```

## The move operator

One thing that we wish to get out of this work is the creation of a move
operator. For moveOnly values, this is equivalent to ending the lifetime of the
value, e.x.:

```
@_moveOnly let x = ...
let _ = move(x) // Ends the lifetime of x
```

Additionally, we want to be able to have something similar for copyable
values. This will necessarily imply a different analysis since we want to ensure
that when we move a copyable value, there aren't any further copyable
values. This ensures that we can move a copyable value into a move only value
and not have to worry about the copyable value having further uses later in the
code. Example:

```
let x = Klass()
@_moveOnly let y = move(x)
// Illegal to use x after this point.
```

## Bring up

I outline the bringup strategy below:

* Introduce the moveOnly attributes for let, var, params. Done.
* Introduce a Builtin that can be used to emit make_moveOnly. Done.
* Use make_moveOnly builtin to bringup the move only value checker. Done.
* Implement function return attributes and use that to define @_moveOnly there. In progress.
* Implement Type System support for SIL level types that can be marked as moveOnly. Done.
* Implement SILGen/TypeLowering support for loading moveOnly values and converting the type as appropriate. Not Done.
* Implement the move operator and the copyable analysis.
* Prepare a toolchain to give to stakeholders to try out the moveOnly attributes and get feedback. Not done.

<!--0
## Move Only Values in Swift

We propose introducing the addition of a new ``@_moveOnly`` attribute that can be
applied in the following places:

* Local and global let, var bindings.
* Function arguments and return values.
* Computed properties.
* Class fields.

An example of this in practice is:

```
```

Some important notes:

* By using attributes in this way, we are avoiding the need to have the AST
  level of the Swift level type system know anything about what a move only
  value is. Importantly this means that there will not be any effect on the
  TypeChecker at the Swift level. It also means that one will not be able to
  overload a function over whether or not an argument or result is move only.

* Once we are at the SIL level, it is important for instructions to be able to
  propagate around if a value's type is move only. As shown above, the natural
  way to do this is by using a bit on the type. To do so, we introduce a bit on
  TypeBase that means that a type is moveOnly, but to ensure that we do not
  violate the previous bullet point, we only allow for this to be done once we
  are at the SIL level.
  
* We specifically do not propose allowing for stored struct fields to be marked
  as ``@_moveOnly`` since that would imply that the struct itself would need to
  be a move only type. This is not an issue for enums or tuples since they can
  not have a payload or element marked with the ``@_moveOnly`` attribute.
-->
<!--
As part of doing this we 

Usage of such attributes would for the first time expose ``@_moveOnly`` semantics
 in Swift. All of these three places can act as a ``@_moveOnly`` binding by
 adding the ``@_moveOnly`` annotation to the binding. Example:

```
sil hidden [ossa] @$s2ex20access_field_exampleyyF : $@convention(thin) () -> () {
bb0:
  %0 = metatype $@thick Klass.Type
  // function_ref Klass.head.unsafeMutableAddressor
  %1 = function_ref @$s2ex5KlassC4headACvau : $@convention(thin) () -> Builtin.RawPointer // user: %2
  %2 = apply %1() : $@convention(thin) () -> Builtin.RawPointer // user: %3
  %3 = pointer_to_address %2 : $Builtin.RawPointer to [strict] $*Klass // user: %4
  %4 = begin_access [read] [dynamic] %3 : $*Klass // users: %6, %5
  %5 = load [copy] %4 : $*Klass                   // users: %19, %8, %7
  %6 = move_value %5 : $*Klass
  end_access %4 : $*Klass                         // id: %6
  debug_value %6 : $@_moveOnly Klass, let, name "x"          // id: %7
  %8 = begin_borrow %5 : $@_moveOnly Klass                   // users: %16, %12, %10, %9
  %9 = class_method %8 : $@_moveOnly Klass, #Klass.next!getter : (Klass) -> () -> Klass?, $@convention(method) (@guaranteed Klass) -> @owned Optional<Klass> // user: %10
  %10 = apply %9(%8) : $@convention(method) (@guaranteed Klass) -> @owned Optional<Klass> // user: %11
  switch_enum %10 : $Optional<Klass>, case #Optional.some!enumelt: bb2, case #Optional.none!enumelt: bb1 // id: %11

bb1:                                              // Preds: bb0
  end_borrow %8 : $Klass                          // id: %12
  br bb3                                          // id: %13

// %14                                            // users: %17, %15
bb2(%14 : @owned $Klass):                         // Preds: bb0
  debug_value %14 : $Klass, let, name "y"         // id: %15
  end_borrow %8 : $Klass                          // id: %16
  destroy_value %14 : $Klass                      // id: %17
  br bb3                                          // id: %18

bb3:                                              // Preds: bb2 bb1
  destroy_value %5 : $Klass                       // id: %19
  %20 = tuple ()                                  // user: %21
  return %20 : $()                                // id: %21
} // end sil function '$s2ex20access_field_exampleyyF'
```
-->

<!--

## The Bringup Plan

The overall bringup plan is as follows:

* 

only value semantics from SILGen to the back of the compiler. We will describe
eac

* Add a bit of information to SILType that would state if a type was
  ``@_moveOnly``. This is necessary to ensure that SIL optimization passes know
  that they must preserve the ``@_moveOnly`` property of a value and not perform
  optimizations that depend on inserting copies for correctness.

* Only allow for a type to be ``@_moveOnly`` if it is a loadable type. This means
  that initially, ``@_moveOnly`` would not be able to be used in generic code or
  with existentials. Once we have opaque values though, the implementation
  should be easy to bring up so such code will follow the same pattern.

* Allowing for ``@_moveOnly`` typed values to be copied in when SIL is in the Raw
  stage by instructions like ``copy_value``. Once we are in Canonical SIL, the
  IR verifier will begin asserting if one of these instructions has an operand
  that is a ``@_moveOnly`` typed value. This guarantees that once we have
  Canonical SIL, a ``@_moveOnly`` typed value can be proven as never copied.

* Banning copying of ``@_moveOnly`` when performing direct operations on
  memory. Example: ``copy_addr, copy_addr [initialization], load [copy]``. To
  copy an address ``@_moveOnly`` value, one must instead perform a ``load
  [take]`` and copy the value as an object. This simplifies our ability to
  verify code by ensuring that all copy operations are actually on values,
  ensuring that we only need an object verifier.

* If a user wishes to escape a value from the `@_moveOnly` jail, they must either
  implement their own deep copy function or use a special SIL instruction
  ``remove_moveOnly`` that unsafely escapes the `@_moveOnly` value and removes
  the type bit.

* TypeLowering will be modified to map ``@_moveOnly T`` arguments and local
  bindings to a Box-like wrapper type called a ``MoveOnlySILType``. This would
  just wrap an underlying type and would act as the extra bit of information
  that SILGen can use to emit ``@_moveOnly T`` code.

* SILGen will need to be modified such that any 

* Implementing a new diagnostic pass called **Diagnostic Copy Propagation** that
  uses the copy propagation infrastructure to rewrite all copies of such values
  and emit diagnostics when we would need to insert copies. The pass would know
  the specific uses that caused the copy to be needed and would be able to emit
  an error that explains which uses imply the need for a copy. We /could/ also
  add a fixit, but looking at other languages like Rust, it seems that such
  functionality was removed from the borrow checker since it was usually the
  wrong answer to a problem. To give good diagnostics to address scenarios where
  we are copying a var into a var, we will add special logic such that when we
  run **Diagnostic Copy Propagation**

* Once **Diagnostic Copy Propagation** has run, all move only types will be
  guaranteed to no longer be copied.

* Finally, once we are in Canonical SIL (after all diagnostic passes have run),
  we will then enforce the IR constraints via the verifier that a ``copy_value``
  can never be used on a move only value. NOTE: This means that passes after
  Diagnostic Copy Propagation will need to preserve the property created by
  **Diagnostic Copy Propagation**.

This will provide a flexible implementation that will let us achieve our goals
for being able to represent a value that is never copied without needing to
rewrite large parts of SILGen.

## The Bringup Plan

Now that we have a methodology for accomplishing our goals, the natural
questions to ask are:

* What do we want this feature to actually look like at the source level?

* What are our goals, non-goals, and anti-goals for this feature?

* In what ways can we incrementally bring up such a feature since we are using a
  new technique that has never been used before in the compiler (Copy
  Propagation for Diagnostics).

## Reference: Performance Copy Propagation in Action

Lets look through a few examples of how performance constant propagation works
to get a sense of how this works at the SIL level. Consider the following SIL:

```
class Klass {}

sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed $Optional<Klass>):
  %1 = copy_value %0 : $Optional<Klass>
  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)

bb1(%2 : @owned $Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@guaranteed SubKlass) -> ()
  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
  apply %f(%3) : $@convention(thin) (@guaranteed SubKlass) -> ()
  destroy_value %3 : $Klass                                                             (C)
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

PCP as it runs visits the transitive def-use graph rooted at ``%0`` looking
through uses that forward ownership from their operands to results ((A),
(B)). It sees that the only true consuming use of %0 is actually the
`destroy_value` (C) and additionally that all of the forwarding uses are able to
take a guaranteed value meaning no copies are needed. It then converts the SIL
to use guaranteed values and eliminates all of the old copies, yielding the
following SIL:

```
sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed $Optional<Klass>):
  switch_enum %0 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)

bb1(%1 : @guaranteed $Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@guaranteed SubKlass) -> ()
  %2 = unchecked_ref_cast %1 : $Klass to $SubKlass                                      (B)
  apply %f(%2) : $@convention(thin) (@guaranteed SubKlass) -> ()
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

Lets now consider a case where we actually need a copy due to a consuming use:

```
sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed $Optional<Klass>):
  %1 = copy_value %0 : $Optional<Klass>
  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)

bb1(%2 : @owned $Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@owned SubKlass) -> ()
  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
  apply %f(%3) : $@convention(thin) (@owned SubKlass) -> ()                             (C)
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

In this case, PCP will again ignore the ``copy_value`` in ``%1`` and will
compute directly from the uses that a copy is needed at (C). But it will see
that it is only needed in bb1, not along bb2. So it will insert a copy_value at
(C) and then eliminate the ``copy_value``, yielding the following SIL:

```
sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed $Optional<Klass>):
  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)

bb1(%2 : @guaranteed $Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@owned SubKlass) -> ()
  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
  %4 = copy_value %3 : $SubKlass
  apply %f(%4) : $@convention(thin) (@owned SubKlass) -> ()                             (C)
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

As a final case, lets consider a situation where we need multiple copies:

```
sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed $Optional<Klass>):
  %1 = copy_value %0 : $Optional<Klass>
  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)

bb1(%2 : @owned $Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@owned SubKlass) -> ()
  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
  apply %f(%3) : $@convention(thin) (@owned SubKlass) -> ()                             (C)
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

One part of the discussion below is the proposal to take advantage of this to
create a new diagnostic pass based off of the same infrastructure as
"performance copy propagation" (PCP), but instead of inserting copies, emits a
diagnostic that tells the user about why the copy was needed and as a fixit,
where the copy needs to be inserted. This would then make it very easy for the
programmer to insert an explicit copy if needed and go on with their day. This
theoretical pass I am going to refer to as "Diagnostic Copy Propagation" (DCP).



<!--

By representing move only values as SIL values in Ownership SSA form that can
never be copied, we are guaranteed via IR invariants that the optimizer will
respect the move only semantics of such a value and thus ensure

In OSSA SIL, we represent copies via the `copy_value` instruction and moves as 

Today SIL is always emitted by SILGen in OSSA form. SIL when in OSSA form
categorizes all values into belonging to an ontology of various ownership
kinds. For more information about this,

a non-trivial value's ownership invariants are enforced
statically at compile time by the IR itself. These enforced OSSA invariants
allow for the compiler to perform a proof when ever SIL is in OSSA form that the
SIL is "Ownership Correct". This means that all non-trivial values in the
verified SIL are never leaked, are never consumed twice, and only have
non-consuming uses within its actual lifetime. In this model

* The value is never leaked
* Is never consumed twice.
* Only has uses within its actual lifetime.

This is done by mandating that all non-trivial values must be able to split
their uses into a set of non-lifetime ending and lifetime ending uses. We
represent moves in this model as a consuming use of an owned value. In order to
change the form of such an owned value, we allow for a notion of what is called
a "forwarding" instruction that takes in an owned value as its operand
(consuming it), transforms the value, and then produces a new owned
value. Example:

```

```

Moves in this model are represented via the notion of a forwarding 


-->





































































### Reference: Performance Copy Propagation

As a result of implementing Ownership SSA (OSSA) in SIL, the Swift compiler can
now infer the lifetimes at compile time of loadable typed values based off of
the SSA uses of that value. The inferred lifetimes are able to be statically
verified by the compiler as being OSSA correct (1) and allows the compiler to
guarantee theoretically minimal ARC traffic. This is done by finding all of the
places where the value has lifetime ending uses (consuming uses) and then
determining the appropriate places where a copy would be needed if a lifetime
ending use is reachable from another lifetime ending use. These are exactly the
places where a programmer would need to manually insert an explicit copy when
using a move only type! Today we use this technique in an optimization pass
called Performance Copy Propagation that given a value, ignores all current
local copies, computes/inserts the minimal set of actual copies needed and
deletes all of the old local copies. Lets take a look at some SIL examples to
see how PCP works. Consider the following SIL:

```
class Klass {}

sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed $Optional<Klass>):
  %1 = copy_value %0 : $Optional<Klass>
  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)

bb1(%2 : @owned $Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@guaranteed SubKlass) -> ()
  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
  apply %f(%3) : $@convention(thin) (@guaranteed SubKlass) -> ()
  destroy_value %3 : $Klass                                                             (C)
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

PCP as it runs visits the transitive def-use graph rooted at ``%0`` looking
through uses that forward ownership from their operands to results ((A),
(B)). It sees that the only true consuming use of %0 is actually the
`destroy_value` (C) and additionally that all of the forwarding uses are able to
take a guaranteed value meaning no copies are needed. It then converts the SIL
to use guaranteed values and eliminates all of the old copies, yielding the
following SIL:

```
sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed $Optional<Klass>):
  switch_enum %0 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)

bb1(%1 : @guaranteed $Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@guaranteed SubKlass) -> ()
  %2 = unchecked_ref_cast %1 : $Klass to $SubKlass                                      (B)
  apply %f(%2) : $@convention(thin) (@guaranteed SubKlass) -> ()
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

Lets now consider a case where we actually need a copy due to a consuming use:

```
sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed $Optional<Klass>):
  %1 = copy_value %0 : $Optional<Klass>
  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)

bb1(%2 : @owned $Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@owned SubKlass) -> ()
  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
  apply %f(%3) : $@convention(thin) (@owned SubKlass) -> ()                             (C)
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

In this case, PCP will again ignore the ``copy_value`` in ``%1`` and will
compute directly from the uses that a copy is needed at (C). But it will see
that it is only needed in bb1, not along bb2. So it will insert a copy_value at
(C) and then eliminate the ``copy_value``, yielding the following SIL:

```
sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed $Optional<Klass>):
  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)

bb1(%2 : @guaranteed $Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@owned SubKlass) -> ()
  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
  %4 = copy_value %3 : $SubKlass
  apply %f(%4) : $@convention(thin) (@owned SubKlass) -> ()                             (C)
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

As a final case, lets consider a situation where we need multiple copies:

```
sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed $Optional<Klass>):
  %1 = copy_value %0 : $Optional<Klass>
  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)

bb1(%2 : @owned $Klass):
  %f = function_ref @myFoo2 : $@convention(thin) (@owned SubKlass) -> ()
  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
  apply %f(%3) : $@convention(thin) (@owned SubKlass) -> ()                             (C)
  br bb3

bb2:
  br bb3

bb3:
  %9999 = tuple()
  return %9999 : $()
}
```

One part of the discussion below is the proposal to take advantage of this to
create a new diagnostic pass based off of the same infrastructure as
"performance copy propagation" (PCP), but instead of inserting copies, emits a
diagnostic that tells the user about why the copy was needed and as a fixit,
where the copy needs to be inserted. This would then make it very easy for the
programmer to insert an explicit copy if needed and go on with their day. This
theoretical pass I am going to refer to as "Diagnostic Copy Propagation" (DCP).



(1) Ownership SSA correct means that the loadable value is guaranteed to never
be consumed twice, leaked, and all uses of the value are within the OSSA
lifetime.

<!--
----

Attendees: TimK, JohnMc, JoeG, DougG, MichaelG

DougG, JohnMc brought up that the two parts of concurrency that overlap are unique classes and mutable if unique classes.

MichaelG then laid out the discussions that we have been having. He first brought up the overall plan that we have been discussing in the Swift performance team:

1. Begin with No Escape Parameters
2. Then do Unique Values/Unique Values
3. Then with time do Unique Types

We will go through the individual discussions below on each section. Some context information is included above those other sections in order to provide context for the reader. Also note that we are assuming that we are working in a world where opaque values exist, so we do not need to worry about handling address only types.

Context: Diagnostic Copy Propagation


No Escape Parameters

#### Design

The overall proposal here was to allow for no-escape to be added to all parameters (instead of just function types). This would be implemented by adding a new diagnostic pass that would:

1. Emit an error diagnostic whenever a non-escape parameter was stored into memory.
2. Allow for trivial parameters to be passed to anything. There is no reason to restrict no escape parameters of trivial values since they are passed by value.
3. For non-trivial parameters, use a form of diagnostic copy propagation to perform the diagnostic. An owned non-trivial value will only be considered to escape if there are multiple consuming uses along the same path of the non-trivial value implying a copy would have been needed and would escape. A guaranteed non-trivial value is likewise only considered to escape if copy propagation found any consuming uses and wanted to insert copies for them. This would enable us to give really nice diagnostics that show the user what caused the escaping use. Additionally we would only allow for this value to be passed as no escape parameters.

#### Response

The end response from the group was that this was a sound idea that we want to pursue as part of this effort.

There were some questions around if we needed to use diagnostic copy propagation/connect this directly with move only value implementation. It was noted that we want no-escape to also work for trivial values like UnsafePointer. We want to prevent people from escaping these pointers from withUnsafePointer closures. The design here will need to be modified to take this into consideration. JohnMc and MikeG agreed that this would just mean a diagnostic pass that looked at uses which is really standard/simple. We would then just loosen the rules over time as people use the feature and we find patterns for trivial types that we are ok with. We could do something similar for non-trivial values by just doing the standard thing in these types of passes: looking through copies as we walk from def→use. We would require that the value is passed only as a no-escape function argument, providing a transitive guarantee. The author (MikeG) thinks this pass design is reasonable and is pretty standard/easy thing to do with a transitive def→use traversal. We also may want to consider adding a way to escape the ‘prison’ for library writers. I think it would just be a node on the def→use traversal.

We also spoke about how this composes with move only values vs unique values. JohnMc noted that a unique value and a move only value behave differently in this model. I talk about it below in the section around move only values/unique values since I believe we were actually talking about different notions of “unique”.

*Conclusion*: But overall, we agreed it was a diagnostic pass that we believe that we could write easily and that it would be useful to people.

*From The Author:* Thanks JohnMc for the great discussion! Part of the reason that the author brought up no escape parameters, was that he thought it would help us prove that copy propagation can be used as a diagnostic in a simpler situation than with move only values. The author was trying to come up with a design program that would help us tip-toe our toes into using copy propagation for diagnostics for the first time! That being said, the escape analysis thing is still something we should do and our users will be notice it!

Unique Values and Unique Value

As a quick online of this section, I start by describing the design from the SIL level up and hopefully show how SIL level constraints pushes us towards a certain design for move only values. Then I will show how I can use Ownership SSA and Diagnostic Copy Promotion to avoid us needing to do a large update to SILGen. Then, I will talk about how we can handle trivial move only types by using an introducer and conclude by talking about how unique types can fit into all of this/how they are different than COW uniqueness (but complementary).

The natural way to represent a move only value in SIL is as a non-trivial loadable value that can not be copied with a normal copy_value. This is because in SIL the notion of being copied is inherently tied to a value being non-trivial since we do not track ownership for trivial values. The ownership of the non-trivial value flows through the entire SIL program via the def→use graph and thus we can write a diagnostic pass based off of copy propagation to infer if a copy is needed somewhere in the program and give a fixit to the user at that spot to insert an explicit copy if they want:

sil @_moveOnly_only : $@convention(thin) (@owned @_moveOnly Klass, @guaranteed @_moveOnly Klass) -> () {
bb0(%0 : @owned @_moveOnly Klass, %1 : @guaranteed @_moveOnly Klass):
  // Safe, can never be copied.
  apply %f(%0) : $@convention(thin) (@guaranteed @_moveOnly Klass) -> ()
  // This is an explicit copy that copy propagation left alone. We treat it as
  // a TrivialUse (meaning an ignorable one) that produces an owned value.
  %3 = explicit_copy %0
  // Error! Diagnostic constant propagation says we need a copy here.
  // Display fixit telling the user that a copy is needed here due to use at SourceLoc()
  apply %f2(%1) : $@convention(thin) (@owned @_moveOnly Klass) -> ()
  // No error, we are passing off our value, moving it.
  apply %f3(%0) : $@convention(thin) (@owned @_moveOnly Klass) -> ()
  // Error! Diagnostic Propagation knows %0 was already destroyed by %f2. We 
  // emit as part of the fixit that we need to put our copy before the call to %f3.
  apply %f4(%0) : $@convention(thin) (@owned @_moveOnly Klass) -> ()
}

This model will make the life of programmers easier since we are not just throwing an error, we are also telling them actionable information to fix the issue!

This looks pretty, but as anyone who has worked in SILGen will tell you, SILGen loves to insert copies! In fact often times SILGen will assume that a value is at +0 to conditionally run at all. All of those places would need to be fixed and it would take time to do so. We could do that 

That all looks nice and good, but as anyone who has worked in SILGen will tell you, SILGen loves to insert copies and a lot of code triggers off of whether or not one is tracking a copy or not. Updating SILGen to support move only types out of the gate will be a lot of work and will require reimplementing/modifying a bunch of SILGen. This is something we can do, but it would be a significant amount of work. Instead, we propose that we:

1. Require move only values to be non-trivial


The author 

in SIL a move only type should naturally be represented as a non-trivial loadable value. By using the information in Ownership SSA, we can easily infer if/where a copy would be needed just based off of the uses of the value. This information is only based off the uses.

for loadable non-trivial values, a class is a very fundamental type. Beyond a class, only class bound protocols are loadable (with time address only types will be as well.. but that is once opaque values lands).

Thus if one wanted to design a notion of a move only type in SIL, it is natural to want to build it on top of non-trivial loadable unique types which are defined as values whose class fields are all unique classes. This would ensure that one could move around the class parts as a move only value and preserve the uniqueness of memory. It flows well with how SIL would want to represent these, as aggregate values. 

a unique class is a class that can never be copied, it can only be moved.

In the SIL type system classes are a foundational type of non-trivial type but there are others. For now since we are assuming that we will not allow for this on generic types until opaque values lands so one will not be able to use this with generics since we specialize code at the SIL level. So, thus any value we are talking about right now are loadable values. Then define a unique loadable value as a non-trivial loadable value that only contains unique non-trivial values. In this model, one would be able to move the value around as a move only value just as if one had a class ensuring that all of the memory structurally is unique. Even better, by taking advantage of Ownership SSA and Diagnostic Copy Propagation, we would be able to emit good diagnostics that explain to the user why the copy is needed and can provide a fixit at the appropriate place in the code where the copy would be needed. This would make working with this system much easier for the programmer. We could also identify explicit copies that were unneeded and provide a fixit to remove the copy. This will create a model where the compiler helps the user work with their code explicitly based off of the uses of the program.

The author then said, well we could bootstrap 

Specifically, a unique value is /ok/ with being copied as long as before the function ends the value is again unique. So in a sense the move only condition is a stronger condition on the def→use graph than the unique condition on values. The author believes upon reflection that the truth is that we were talking about two different forms of uniqueness. One is uniqueness in terms of the current model of a COW array vs uniqueness in the sense of a unique class. A unique class is a class that can never be copied.


-->

<!--
+
+This document contains a proposal to add three features into Swift:
+
+* No escaping arguments
+* Move only values
+* Move only types
+
+The general outline of our discussion will be:
+
+1. We begin by talking about how move only values are naturally represented in
+   SIL when in Ownership SSA (OSSA) form and how that affects how the feature
+   must be built.
+
+2. Then we will talk about how the structure of SILGen additionally restricts
+   our pathways towards bringing up this feature.
+
+3. We will then discuss a new technique Performance Copy Propagation that allows
+   the compiler to infer lifetimes for values based off of uses of a value. We
+   will show how we can take advantage of this technique to define a new
+   diagnostic pass called Diagnostic Copy Propagation that we can use as a
+   borrow checker.
+
+4. Using Diagnostic Copy Propagation, we will then show how we can use it to
+   bring up first a no escape argument feature, then move only values, and
+   finally move only types.
+
+## Introduction: Diagnostic Copy Propagation
+
+As a result of implementing Ownership SSA (OSSA) in SIL, the Swift compiler can
+now infer the lifetimes at compile time of loadable typed values based off of
+the SSA uses of that value. The inferred lifetimes are able to be statically
+verified by the compiler as being OSSA correct (1) and allows the compiler to
+guarantee theoretically minimal ARC traffic. This is done by finding all of the
+places where the value has lifetime ending uses (consuming uses) and then
+determining the appropriate places where a copy would be needed if a lifetime
+ending use is reachable from another lifetime ending use. These are exactly the
+places where a programmer would need to manually insert an explicit copy when
+using a move only type! Today we use this technique in an optimization pass
+called Performance Copy Propagation that given a value, ignores all current
+local copies, computes/inserts the minimal set of actual copies needed and
+deletes all of the old local copies. Lets take a look at some SIL examples to
+see how PCP works. Consider the following SIL:
+
+```
+class Klass {}
+
+sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
+bb0(%0 : @guaranteed $Optional<Klass>):
+  %1 = copy_value %0 : $Optional<Klass>
+  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)
+
+bb1(%2 : @owned $Klass):
+  %f = function_ref @myFoo2 : $@convention(thin) (@guaranteed SubKlass) -> ()
+  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
+  apply %f(%3) : $@convention(thin) (@guaranteed SubKlass) -> ()
+  destroy_value %3 : $Klass                                                             (C)
+  br bb3
+
+bb2:
+  br bb3
+
+bb3:
+  %9999 = tuple()
+  return %9999 : $()
+}
+```
+
+PCP as it runs visits the transitive def-use graph rooted at ``%0`` looking
+through uses that forward ownership from their operands to results ((A),
+(B)). It sees that the only true consuming use of %0 is actually the
+`destroy_value` (C) and additionally that all of the forwarding uses are able to
+take a guaranteed value meaning no copies are needed. It then converts the SIL
+to use guaranteed values and eliminates all of the old copies, yielding the
+following SIL:
+
+```
+sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
+bb0(%0 : @guaranteed $Optional<Klass>):
+  switch_enum %0 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)
+
+bb1(%1 : @guaranteed $Klass):
+  %f = function_ref @myFoo2 : $@convention(thin) (@guaranteed SubKlass) -> ()
+  %2 = unchecked_ref_cast %1 : $Klass to $SubKlass                                      (B)
+  apply %f(%2) : $@convention(thin) (@guaranteed SubKlass) -> ()
+  br bb3
+
+bb2:
+  br bb3
+
+bb3:
+  %9999 = tuple()
+  return %9999 : $()
+}
+```
+
+Lets now consider a case where we actually need a copy due to a consuming use:
+
+```
+sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
+bb0(%0 : @guaranteed $Optional<Klass>):
+  %1 = copy_value %0 : $Optional<Klass>
+  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)
+
+bb1(%2 : @owned $Klass):
+  %f = function_ref @myFoo2 : $@convention(thin) (@owned SubKlass) -> ()
+  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
+  apply %f(%3) : $@convention(thin) (@owned SubKlass) -> ()                             (C)
+  br bb3
+
+bb2:
+  br bb3
+
+bb3:
+  %9999 = tuple()
+  return %9999 : $()
+}
+```
+
+In this case, PCP will again ignore the ``copy_value`` in ``%1`` and will
+compute directly from the uses that a copy is needed at (C). But it will see
+that it is only needed in bb1, not along bb2. So it will insert a copy_value at
+(C) and then eliminate the ``copy_value``, yielding the following SIL:
+
+```
+sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
+bb0(%0 : @guaranteed $Optional<Klass>):
+  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)
+
+bb1(%2 : @guaranteed $Klass):
+  %f = function_ref @myFoo2 : $@convention(thin) (@owned SubKlass) -> ()
+  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
+  %4 = copy_value %3 : $SubKlass
+  apply %f(%4) : $@convention(thin) (@owned SubKlass) -> ()                             (C)
+  br bb3
+
+bb2:
+  br bb3
+
+bb3:
+  %9999 = tuple()
+  return %9999 : $()
+}
+```
+
+As a final case, lets consider a situation where we need multiple copies:
+
+```
+sil [ossa] @myFunc : $@convention(thin) (@guaranteed Klass) -> () {
+bb0(%0 : @guaranteed $Optional<Klass>):
+  %1 = copy_value %0 : $Optional<Klass>
+  switch_enum %1 : $Optional<Klass>, case #Optional.some: bb1, case #Optional.none: bb2 (A)
+
+bb1(%2 : @owned $Klass):
+  %f = function_ref @myFoo2 : $@convention(thin) (@owned SubKlass) -> ()
+  %3 = unchecked_ref_cast %2 : $Klass to $SubKlass                                      (B)
+  apply %f(%3) : $@convention(thin) (@owned SubKlass) -> ()                             (C)
+  br bb3
+
+bb2:
+  br bb3
+
+bb3:
+  %9999 = tuple()
+  return %9999 : $()
+}
+```
+
+One part of the discussion below is the proposal to take advantage of this to
+create a new diagnostic pass based off of the same infrastructure as
+"performance copy propagation" (PCP), but instead of inserting copies, emits a
+diagnostic that tells the user about why the copy was needed and as a fixit,
+where the copy needs to be inserted. This would then make it very easy for the
+programmer to insert an explicit copy if needed and go on with their day. This
+theoretical pass I am going to refer to as "Diagnostic Copy Propagation" (DCP).
+
+
+
+(1) Ownership SSA correct means that the loadable value is guaranteed to never
+be consumed twice, leaked, and all uses of the value are within the OSSA
+lifetime.
+
+----
+
+Attendees: TimK, JohnMc, JoeG, DougG, MichaelG
+
+DougG, JohnMc brought up that the two parts of concurrency that overlap are unique classes and mutable if unique classes.
+
+MichaelG then laid out the discussions that we have been having. He first brought up the overall plan that we have been discussing in the Swift performance team:
+
+1. Begin with No Escape Parameters
+2. Then do Unique Values/Unique Values
+3. Then with time do Unique Types
+
+We will go through the individual discussions below on each section. Some context information is included above those other sections in order to provide context for the reader. Also note that we are assuming that we are working in a world where opaque values exist, so we do not need to worry about handling address only types.
+
+Context: Diagnostic Copy Propagation
+
+
+No Escape Parameters
+
+#### Design
+
+The overall proposal here was to allow for no-escape to be added to all parameters (instead of just function types). This would be implemented by adding a new diagnostic pass that would:
+
+1. Emit an error diagnostic whenever a non-escape parameter was stored into memory.
+2. Allow for trivial parameters to be passed to anything. There is no reason to restrict no escape parameters of trivial values since they are passed by value.
+3. For non-trivial parameters, use a form of diagnostic copy propagation to perform the diagnostic. An owned non-trivial value will only be considered to escape if there are multiple consuming uses along the same path of the non-trivial value implying a copy would have been needed and would escape. A guaranteed non-trivial value is likewise only considered to escape if copy propagation found any consuming uses and wanted to insert copies for them. This would enable us to give really nice diagnostics that show the user what caused the escaping use. Additionally we would only allow for this value to be passed as no escape parameters.
+
+#### Response
+
+The end response from the group was that this was a sound idea that we want to pursue as part of this effort.
+
+There were some questions around if we needed to use diagnostic copy propagation/connect this directly with move only value implementation. It was noted that we want no-escape to also work for trivial values like UnsafePointer. We want to prevent people from escaping these pointers from withUnsafePointer closures. The design here will need to be modified to take this into consideration. JohnMc and MikeG agreed that this would just mean a diagnostic pass that looked at uses which is really standard/simple. We would then just loosen the rules over time as people use the feature and we find patterns for trivial types that we are ok with. We could do something similar for non-trivial values by just doing the standard thing in these types of passes: looking through copies as we walk from def→use. We would require that the value is passed only as a no-escape function argument, providing a transitive guarantee. The author (MikeG) thinks this pass design is reasonable and is pretty standard/easy thing to do with a transitive def→use traversal. We also may want to consider adding a way to escape the ‘prison’ for library writers. I think it would just be a node on the def→use traversal.
+
+We also spoke about how this composes with move only values vs unique values. JohnMc noted that a unique value and a move only value behave differently in this model. I talk about it below in the section around move only values/unique values since I believe we were actually talking about different notions of “unique”.
+
+*Conclusion*: But overall, we agreed it was a diagnostic pass that we believe that we could write easily and that it would be useful to people.
+
+*From The Author:* Thanks JohnMc for the great discussion! Part of the reason that the author brought up no escape parameters, was that he thought it would help us prove that copy propagation can be used as a diagnostic in a simpler situation than with move only values. The author was trying to come up with a design program that would help us tip-toe our toes into using copy propagation for diagnostics for the first time! That being said, the escape analysis thing is still something we should do and our users will be notice it!
+
+Unique Values and Unique Value
+
+As a quick online of this section, I start by describing the design from the SIL level up and hopefully show how SIL level constraints pushes us towards a certain design for move only values. Then I will show how I can use Ownership SSA and Diagnostic Copy Promotion to avoid us needing to do a large update to SILGen. Then, I will talk about how we can handle trivial move only types by using an introducer and conclude by talking about how unique types can fit into all of this/how they are different than COW uniqueness (but complementary).
+
+The natural way to represent a move only value in SIL is as a non-trivial loadable value that can not be copied with a normal copy_value. This is because in SIL the notion of being copied is inherently tied to a value being non-trivial since we do not track ownership for trivial values. The ownership of the non-trivial value flows through the entire SIL program via the def→use graph and thus we can write a diagnostic pass based off of copy propagation to infer if a copy is needed somewhere in the program and give a fixit to the user at that spot to insert an explicit copy if they want:
+
+sil @_moveOnly_only : $@convention(thin) (@owned @_moveOnly Klass, @guaranteed @_moveOnly Klass) -> () {
+bb0(%0 : @owned @_moveOnly Klass, %1 : @guaranteed @_moveOnly Klass):
+  // Safe, can never be copied.
+  apply %f(%0) : $@convention(thin) (@guaranteed @_moveOnly Klass) -> ()
+  // This is an explicit copy that copy propagation left alone. We treat it as
+  // a TrivialUse (meaning an ignorable one) that produces an owned value.
+  %3 = explicit_copy %0
+  // Error! Diagnostic constant propagation says we need a copy here.
+  // Display fixit telling the user that a copy is needed here due to use at SourceLoc()
+  apply %f2(%1) : $@convention(thin) (@owned @_moveOnly Klass) -> ()
+  // No error, we are passing off our value, moving it.
+  apply %f3(%0) : $@convention(thin) (@owned @_moveOnly Klass) -> ()
+  // Error! Diagnostic Propagation knows %0 was already destroyed by %f2. We 
+  // emit as part of the fixit that we need to put our copy before the call to %f3.
+  apply %f4(%0) : $@convention(thin) (@owned @_moveOnly Klass) -> ()
+}
+
+This model will make the life of programmers easier since we are not just throwing an error, we are also telling them actionable information to fix the issue!
+
+This looks pretty, but as anyone who has worked in SILGen will tell you, SILGen loves to insert copies! In fact often times SILGen will assume that a value is at +0 to conditionally run at all. All of those places would need to be fixed and it would take time to do so. We could do that 
+
+That all looks nice and good, but as anyone who has worked in SILGen will tell you, SILGen loves to insert copies and a lot of code triggers off of whether or not one is tracking a copy or not. Updating SILGen to support move only types out of the gate will be a lot of work and will require reimplementing/modifying a bunch of SILGen. This is something we can do, but it would be a significant amount of work. Instead, we propose that we:
+
+1. Require move only values to be non-trivial
+
+
+The author 
+
+in SIL a move only type should naturally be represented as a non-trivial loadable value. By using the information in Ownership SSA, we can easily infer if/where a copy would be needed just based off of the uses of the value. This information is only based off the uses.
+
+for loadable non-trivial values, a class is a very fundamental type. Beyond a class, only class bound protocols are loadable (with time address only types will be as well.. but that is once opaque values lands).
+
+Thus if one wanted to design a notion of a move only type in SIL, it is natural to want to build it on top of non-trivial loadable unique types which are defined as values whose class fields are all unique classes. This would ensure that one could move around the class parts as a move only value and preserve the uniqueness of memory. It flows well with how SIL would want to represent these, as aggregate values. 
+
+a unique class is a class that can never be copied, it can only be moved.
+
+In the SIL type system classes are a foundational type of non-trivial type but there are others. For now since we are assuming that we will not allow for this on generic types until opaque values lands so one will not be able to use this with generics since we specialize code at the SIL level. So, thus any value we are talking about right now are loadable values. Then define a unique loadable value as a non-trivial loadable value that only contains unique non-trivial values. In this model, one would be able to move the value around as a move only value just as if one had a class ensuring that all of the memory structurally is unique. Even better, by taking advantage of Ownership SSA and Diagnostic Copy Propagation, we would be able to emit good diagnostics that explain to the user why the copy is needed and can provide a fixit at the appropriate place in the code where the copy would be needed. This would make working with this system much easier for the programmer. We could also identify explicit copies that were unneeded and provide a fixit to remove the copy. This will create a model where the compiler helps the user work with their code explicitly based off of the uses of the program.
+
+The author then said, well we could bootstrap 
+
+Specifically, a unique value is /ok/ with being copied as long as before the function ends the value is again unique. So in a sense the move only condition is a stronger condition on the def→use graph than the unique condition on values. The author believes upon reflection that the truth is that we were talking about two different forms of uniqueness. One is uniqueness in terms of the current model of a COW array vs uniqueness in the sense of a unique class. A unique class is a class that can never be copied.
+
+

-->
