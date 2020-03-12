---
layout: page
title: Ownership SSA and Safe Interior Pointers
categories: complete
---

**NOTE** This is intended to be a document for compiler
developers/language enthusiasts. It assumes familiarity with compiler
concepts and low level details. It also assumes some familiarity with
ownership SSA/SIL and that one has watched my talk at
[LLVM Dev 2019 talk on Ownership SSA](https://www.youtube.com/watch?v=qy3iZPHZ88o). The
reason for these assumptions is for the expediency of the author
writing this up.

## Background

An interior pointer is a bare pointer into the innards of an object. A
simple example of this in C++ would be using the method
`std::vector::data()` to get to the innards of a `std::vector`. In
general interior pointers are unsafe to use since languages do not
provide any guarantees that the interior pointer will not be used
after the underlying object has been deallocated. To see this,
consider the following C++ example:

```
int unfortunateFunction() {
  int *unsafeInteriorPointer = nullptr;
  {
    std::vector<int> vector;
    vector.push_back(5);
    unsafeInteriorPointer = vector.data();
    printf("%d\n", *unsafeInteriorPointer); // Prints "5".
  } // vector deallocated here
  return *unsafeInteriorPointer; // Kaboom
}
```

In words, C++ allows for us to get the interior pointer into the
vector, but then lets us do whatever we want with the pointer,
including use it after the underlying memory has been invalidated.

From a user's perspective, interior pointers are really useful since
one can use it to pass data to other APIs that are only expecting a
pointer and also since one can use it to sometimes get better
performance. But from a language designer perspective, this sort of
API verbotten and leads to bugs, crashes, and security
vulnerabilities. That being said, clearly users have a need for such
functionality, so we, as language designers, should figure out manners
to express these sorts of patterns in our various languages in a safe
way that prevents user's from foot-gunning themselves. In SIL, we have
solved this problem via the direct modeling of interior pointer
instructions as a high level concept in our IR.

## Interior Pointers in SIL

In contrast to LLVM-IR, SIL provides mechanisms that language
designers can use to express concepts like the above in a manner that
allows for users to use interior pointers in a safe manner. This is
expressable in SIL today and is exposed in a very limited form in the
Swift language itself via the usage of _read accessor on computed
properties. Before we get to that though, we need to start at a more
basic level about how interior pointers are represented in SIL. In
sum, SIL has a number of instructions for accessing constructing
interior pointers into different sorts of objects. They are (as of the
time when the author was writing this document):

1. `project_box` - projects a pointer out of a reference counted box.
2. `ref_element_addr` - projects a field out of a reference counted class.
3. `ref_tail_addr` - projects out a pointer to a class's tail allocated
   array memory (assuming the class was initialized to have such an
   array).
4. `open_existential_box` - projects the address of the value out of a
   boxed existential container using the current function
   context/protocol conformance to create an "opened archetype".
5. `project_existential_box` - projects a pointer to the value inside
   a boxed existential container. Must be the type for which the box
   was initially allocated for and not for an "opened" archetype.

As of today in the compiler, safe interior pointers are implemented
for `ref_element_addr`, `ref_tail_addr`, `open_existential_box`, and
`project_existential_box`. `alloc_box` will be fixed with time but
requires additional work in Swift's SIL generation around the manner
how initializers are implemented. That being said, for our purposes,
we can ignore that issue.

**NOTE** SIL allows for interior pointers to be unsafely converted
into "unsafe" interior pointers as an escape hatch via the instruction
`address_to_pointer`. In such a case, we rely on the language frontend
to provide correctness and do not provide any guarantees on the
address's liveness beyond the actual conversion instruction. For
instance, one could unsafely convert the pointer back to an address
using `pointer_to_address` and OSSA would not provide any guarantees.

## Guaranteed Values and Safe Interior Pointers

In OSSA, one of our forms of ownership is "guaranteed" ownership. A
value with "guaranteed" ownership is an immutable value with a scoped
based lifetime on another "owned" value. As an example of such a
guaranteed value, consider the following SIL:

```
class Klass {
  let optionalField: Optional<Klass>
}

sil [ossa] @func : $@convention(thin) () -> () {
bb0:
  ...
  // Get a +1 owned reference to a class.
  %0 = apply %getKlass() : $@convention(thin) () -> @owned Klass
  // Create a guaranteed reference to the class. %0 can not be destroyed until %1's end_borrow
  %1 = begin_borrow %0 : $Klass
  // Project out the address from the guaranteed Klass.
  %2 = ref_element_addr %1 : $Klass, #Klass.optionalField
  // Pass the address as an in_guaranteed parameter to opaqueUser.
  %opaqueUser = function_ref @opaqueUser : $@convention(thin) (@in_guaranteed Optional<Klass>) -> ()
  apply %opaqueUser(%2) : $@convention(thin) (@in_guaranteed Optional<Klass>) -> ()
  // Invalidate %1
  end_borrow %1 : $Klass
  // Invalidate %0
  destroy_value %0 : $Klass
  ...
}
```

Lets break down what is happening in this SIL. We first call the
function `%getKlass()` that returns to us a +1 `@owned` Klass. This is
a reference counted value that has an independent linear lifetime. So
it must be destroyed exactly once along all paths through the program
that go through its allocation. Then we assign the name `%0` to that
`@owned Klass`.

Then `%0` is "borrowed" to produce a new `@guaranteed` value
`%1`. Since `%1` is borrowed from `%0`, we know that `%0` can not be
destroyed while `%1` is alive. This means that if we were to re-write
the example above in any of the following manners, the SIL ownership
verifier would immediately catch our error:

```
  // Ex 1.
  %0 = apply %getKlass() : $@convention(thin) () -> @owned Klass
  %1 = begin_borrow %0 : $Klass
  %2 = ref_element_addr %1 : $Klass, #Klass.optionalField
  ...
  destroy_value %0 : $Klass
  end_borrow %1 : $Klass    // use after free.

  // Ex 2.
  %0 = apply %getKlass() : $@convention(thin) () -> @owned Klass
  %1 = begin_borrow %0 : $Klass
  %2 = ref_element_addr %1 : $Klass, #Klass.optionalField
  ...
  destroy_value %1 : $Klass // Incompatible ownership kind
  destroy_value %0 : $Klass
```

This provides a very powerful way of guaranteeing the lifetime of an
`@owned` value in a region of code that can be easily worked with. We
take advantage of this in the case of interior pointers by:

1. Requiring that all interior pointer instructions can only have
   `@guaranteed` operands.
2. Requiring that all uses of the interior pointer be completely
   enclosed within the scope of the `@guaranteed` operand that the
   interior pointer is derived from.

The end result of this is if we were to write the following SIL, the
ownership verifier would immediately catch our error since we would be
using the interior pointer outside of the borrowed from object's scope:

```
sil [ossa] @func : $@convention(thin) () -> () {
bb0:
  ...
  %0 = apply %getKlass() : $@convention(thin) () -> @owned Klass
  %1 = begin_borrow %0 : $Klass
  %2 = ref_element_addr %1 : $Klass, #Klass.optionalField
  %opaqueUser = function_ref @opaqueUser : $@convention(thin) (@in_guaranteed Optional<Klass>) -> ()
  end_borrow %1 : $Klass
  // Error: Escaped interior pointer!
  apply %opaqueUser(%2) : $@convention(thin) (@in_guaranteed Optional<Klass>) -> ()
  destroy_value %0 : $Klass
  // Error: Escaped interior pointer!
  apply %opaqueUser(%2) : $@convention(thin) (@in_guaranteed Optional<Klass>) -> ()
  ...
}
```

This allows for a language designer to know that any code emitted in
this way can not be mis-optimized by the optimizer (since the ossa
verifier would catch this error) and thus perform more aggressive
optimizations/have more aggressive language features.

## Safe Interior Pointers and Coroutines

While this is all nice and good, notice how in our discussion before,
we relied on `Klass`'s internal representation being fragile and known
to the compiler. This is a huge limitation to this approach and if we
did not have any additional functionality, would result in "Safe
Interior Pointers" being less interesting. Luckily, SIL also supports
the notion of yield once coroutines that combines with this feature to
save us.

A yield once coroutine is a coroutine that can only be yielded and
resumed exactly once. For the purposes of our discussion, consider the
following _read accessor on a class:

```
protocol Prot {
    func printMsg()
}

final class Klass {
    let _innerState : Prot? = nil
    var state: Prot {
        @inline(never)
        _read { yield _innerState! }
    }
}

func useKlass(_ k: Klass) {
    k.state.printMsg()
}
```

Lets look at how this could be represented in SIL:

```
sil hidden @$s4main8useKlassyyAA0C0CF : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : $Klass):
  %2 = function_ref @$s4main5KlassC5stateAA4Prot_pvr : $@yield_once @convention(method) \
                 (@guaranteed Klass) -> @yields @in_guaranteed Prot
  (%3, %4) = begin_apply %2(%0) : $@yield_once @convention(method) \
                 (@guaranteed Klass) -> @yields @in_guaranteed Prot
  %8 = open_existential_addr immutable_access %4 : $*Prot to $*@opened("...") Prot
  %9 = witness_method $@opened("...") Prot, #Prot.printMsg!1, %8 : $*@opened("...") Prot
  %10 = apply %9<@opened("...") Prot>(%8) : $@convention(witness_method: Prot) \
                 <τ_0_0 where τ_0_0 : Prot> (@in_guaranteed τ_0_0) -> ()
  end_apply %4
  %13 = tuple ()
  return %13 : $()
} // end sil function '$s4main8useKlassyyAA0C0CF'
```

**NOTE** The reason why I said above could be represented is that
Swift's current implementation of _read accessors does not extend
lifetimes far enough for us today to get the above codegen. That being
said, SIL supports this and the limitation is just with frontend
codegen emitter rather than with SIL itself.

The key thing to see here is notice how we pass the `%0` to as a
guaranteed parameter to the coroutine. The semantics of an
`@guaranteed` parameter states that the value must be alive over the
entire invocation region of a function. In the case of a coroutine,
this generalizes naturally to the statement that an `@guaranteed`
parameter for a coroutine must have a lifetime that is guaranteed to
not end before the coroutine finishes evaluating. In terms of our
example above, this means that without knowing anything about `%0`
that `%0`'s lifetime must extend at least until the end_apply of the
coroutine, regardless of the location/knowledge of the implementation
of the coroutine.

```
sil hidden @$s4main8useKlassyyAA0C0CF : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : $Klass):
  %2 = function_ref @$s4main5KlassC5stateAA6Klass2Cvr : $@yield_once @convention(method) \
                (@guaranteed Klass) -> @yields @guaranteed Klass2
  // %0 must be live from the begin_apply here.
  (%3, %4) = begin_apply %2(%0) : $@yield_once @convention(method) (@guaranteed Klass) -> \
                @yields @guaranteed Klass2
  %7 = function_ref @$s4main6Klass2C8printMsgyyF : $@convention(method) (@guaranteed Klass2) -> ()
  %8 = apply %7(%3) : $@convention(method) (@guaranteed Klass2) -> ()
  // To the end of the coroutine here, end_apply.
  end_apply %4
  %10 = tuple ()
  return %10 : $()
} // end sil function '$s4main8useKlassyyAA0C0CF'
```

Again considering the semantics of coroutines, any yielded
in_guaranteed addresses have a lifetime that must be constrained by
the coroutine, since the coroutine only guarantees the value is around
until it finished evaluating. This means that we can only use `%3`
(the address) until the end_apply.

```
sil hidden @$s4main8useKlassyyAA0C0CF : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : $Klass):
  %2 = function_ref @$s4main5KlassC5stateAA6Klass2Cvr : $@yield_once @convention(method) \
                (@guaranteed Klass) -> @yields @guaranteed Klass2
  // %3 is only valid from the begin_apply
  (%3, %4) = begin_apply %2(%0) : $@yield_once @convention(method) (@guaranteed Klass) -> \
                @yields @guaranteed Klass2
  %7 = function_ref @$s4main6Klass2C8printMsgyyF : $@convention(method) (@guaranteed Klass2) -> ()
  %8 = apply %7(%3) : $@convention(method) (@guaranteed Klass2) -> ()
  // To the end of the coroutine here, end_apply.
  end_apply %4
  %10 = tuple ()
  return %10 : $()
} // end sil function '$s4main8useKlassyyAA0C0CF'
```

Now look at both of the regions in the SIL where both `%0` and `%3`
must be live. Notice how in both cases, the regions begin at the
begin_apply and end at the end_apply. This then allows us to conclude
that statically at compile time the two value's must not be destroyed
while the coroutine is live, regardless of the definition of
printMsg. This then allows us to conclude that we can prove at compile
time via Ownership SSA and coroutines that we can opaquely project out
interior opaquely project out the interior pointers from `%0` safely
across dylib boundaries. QED.