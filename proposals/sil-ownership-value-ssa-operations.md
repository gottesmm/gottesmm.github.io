---
layout: proposal
title: SIL Ownership SSA Value Operations
categories: proposals
---

# {{ page.title }}

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Summary](#summary)
- [Definitions](#definitions)
    - [store_borrow](#storeborrow)
    - [begin_borrow](#beginborrow)
    - [tuple_destructure, struct_destructure, destructure_result](#tupledestructure-structdestructure-destructureresult)

<!-- markdown-toc end -->

# Summary

This document proposes the addition of the following new SIL instructions:

1. `store_borrow`
2. `begin_borrow`
3. `tuple_destructure`, `struct_destructure`, `destructure_result`

These are necessary to express the following operations in SILGen:

1. Passing an `@guaranteed` value to an `@in_guaranteed` argument without
   copy. (`store_borrow`)
<!-- 2. Passing an `@owned` value as an `@inout` argument without
   copy. (store_borrow [inout], end_inout_borrow) -->
3. Copying a field from an `@owned` aggregate without consuming the
   aggregate. (`begin_borrow`)
4. Passing an `@owned` value as an `@guaranteed` argument (`begin_borrow`)
5. Performing a destructure take operation when emitting multiply operations
   in a statement. (`tuple_destructure`, `struct_destructure`, `destructure_result`).

# Definitions

## store_borrow

Define `store_borrow` as:

    %borrowed_y = store_borrow %x to %y : $*T
    ...
    end_borrow %borrowed_y from %x : $*T, $T

      =>

    store %x to %y
    ...
    end_borrow %x from %y

`store_borrow` is needed to convert `@guaranteed` values to `@in_guaranteed`
arguments. Without a `store_borrow`, this can only be expressed via an
inefficient `copy_value` + `store` + `load` + `destroy_value` sequence:

    sil @g : $@convention(thin) (@in_guaranteed Foo) -> ()
    sil @f : $@convention(thin) (@guaranteed Foo) -> () {
    bb0(%0 : $Foo):
      %1 = function_ref @g : $@convention(thin) (@in_guaranteed Foo) -> ()
      %2 = alloc_stack $Foo
      %3 = copy_value %0 : $Foo
      store %3 to [init] %2 : $Foo
      apply %1(%2) : $@convention(thin) (@in_guaranteed Foo) -> ()
      %4 = load [take] %2 : $*Foo
      destroy_value %4 : $Foo
      dealloc_stack %2 : $Foo
      ...
    }

`store_borrow` allows us to express this in a more efficient and expressive SIL:

    sil @g : $@convention(thin) (@in_guaranteed Foo) -> ()
    sil @f : $@convention(thin) (@guaranteed Foo) -> () {
    bb0(%0 : $Foo):
      %1 = function_ref @g : $@convention(thin) (@in_guaranteed Foo) -> ()
      %2 = alloc_stack $Foo
      %3 = store_borrow %0 to %2 : $*T
      apply %1(%3) : $@convention(thin) (@in_guaranteed Foo) -> ()
      end_borrow %3 from %0 : $*T, $T
      dealloc_stack %2 : $Foo
      ...
    }

## begin_borrow

Define a `begin_borrow` instruction as:

    %borrowed_x = begin_borrow %x : $T
    ...
    end_borrow %borrowed_x from %x : $T, $T
    
      =>
    
    %xhat = fake_identity_inst %x : $T
    ...
    fix_lifetime(%xhat)
    fix_lifetime(%x)

A `begin_borrow` instruction explicitly converts an `@owned` value to a
`@guaranteed` value. The result of the `begin_borrow` is paired with an
`end_borrow` instruction that explicitly represents the end scope of the
`begin_borrow`.

Without this instruction, we can not copy out sub-projections from an aggregate
without storing to memory or performing extra, `copy_value`,
`destroy_values`. Consider the following SIL:

    struct Foo {
      var x: Builtin.NativeObject
      var y: Builtin.NativeObject
    }

    sil @g : $@convention(thin) (@owned Builtin.NativeObject) -> ()
    sil @f : $@convention(thin) (@owned Foo) -> Builtin.NativeObject {
    bb0(%0 : $Foo):
        %1 = struct_extract %0 : $Foo, #Foo.x
        %2 = copy_value %1 : $Builtin.NativeObject
        %3 = function_ref @g : $@convention(thin) (@owned Foo) -> ()
        %4 = apply %3(%0) : $@convention(thin) (@owned Foo) -> ()
        return %2 : $Builtin.NativeObject
    }

In Semantic SIL this is illegal since `%0` is consumed by `%1` and `%4`. We
could express this via a destructure take operation (see below), but we would
need to introduce to copy and destroy `#Foo.y` unnecessarily:

    sil @g : $@convention(thin) (@owned Builtin.NativeObject) -> ()
    sil @f : $@convention(thin) (@owned Foo) -> Builtin.NativeObject {
    bb0(%0 : $Foo):
        %1 = copy_value %0 : $Foo
        %2 = struct_destructure $1 : $Foo
        %3 = destructure_result %2 : #Foo.x
        %4 = destructure_result %2 : #Foo.y
        %5 = function_ref @g : $@convention(thin) (@owned Foo) -> ()
        apply %5(%0) : $@convention(thin) (@owned Foo) -> ()
        destroy_value %4 : $Builtin.NativeObject
        return %3 : $Builtin.NativeObject
    }

But using `begin_borrow`, this operation can be expressed as:

    sil @g : $@convention(thin) (@owned Builtin.NativeObject) -> ()
    sil @f : $@convention(thin) (@owned Foo) -> Builtin.NativeObject {
    bb0(%0 : $Foo):
        %1 = begin_borrow %0 : $Foo
        %2 = struct_extract %1 : $Foo, #Foo.x
        %3 = copy_value %2 : $Builtin.NativeObject
        end_borrow %1 from %0 : $Foo, $Foo
        %4 = function_ref @g : $@convention(thin) (@owned Foo) -> ()
        apply %4(%0) : $@convention(thin) (@owned Foo) -> ()
        return %3 : $Builtin.NativeObject
    }

## tuple_destructure, struct_destructure, destructure_result

In SILGen, take operations are used to forward values in between sub-expressions
of a statement. Consider the following psuedo-Swift:

    struct S {
        var x: Builtin.NativeObject
        var y: Builtin.NativeObject
    }

    func bar1(s: S) -> S { return s }
    func bar2(x: Builtin.NativeObject) -> () { ... }
    func foo(s: S) {
        bar2(bar1(s).x)
    }

This will approximately lower to the following SIL today:

    sil @bar1 : $@convention(thin) <T> (@in T) -> @out T {
    bb0(%0 : $*T, %1 : $*T):
        copy_addr [take] %1 to [init] %0 : $*T
        return
    }

    sil @bar2 : $@convention(thin) (@owned Builtin.NativeObject) -> () { ... }

    sil @foo : $@convention(thin) (@owned S) -> () {
    bb0(%0 : $S):
        %1 = function_ref @bar1 : $@convention(thin) (@owned S) -> @owned S
        %2 = copy_value %0 : $S
        %3 = apply %1(%2) : $@convention(thin) (@owned S) -> @owned S
        %4 = struct_extract %3 : $S, #S.x
        %5 = struct_extract %3 : $S, #S.y
        %6 = function_ref @bar2 : $@convention(thin) (@owned Builtin.NativeObject) -> ()
        apply %6(%4) : $@convention(thin) (@owned Builtin.NativeObject) -> ()
        destroy_value %5 : $Builtin.NativeObject
        dealloc_stack %1 : $*S
        destroy_value %0 : $S
    }

This is invalid semantic SIL since %3 is consumed twice, once by `%4` and a
second time by `%5`. We could use the `begin_borrow` trick from above to copy
only `%4`, but we would still in such a case, have to destroy all of `%3`,
implying an unneeded `destroy_value` operation:

    sil @foo : $@convention(thin) (@owned S) -> () {
    bb0(%0 : $S):
        %1 = function_ref @bar1 : $@convention(thin) (@owned S) -> @owned S
        %2 = copy_value %0 : $S
        %3 = apply %1(%2) : $@convention(thin) (@owned S) -> @owned S
        %4 = begin_borrow %3 : $S
        %5 = struct_extract %4 : $S, #S.x
        %6 = copy_value %5 : $Builtin.NativeObject
        end_borrow %4 from %3 : $S, $S
        %7 = function_ref @bar2 : $@convention(thin) (@owned Builtin.NativeObject) -> ()
        apply %6(%5) : $@convention(thin) (@owned Builtin.NativeObject) -> ()
        destroy_value %3 : $S
        dealloc_stack %1 : $*S
        destroy_value %0 : $S
    }

To represent this code sequence optimally, we instead use a take destructuring
operation:

    sil @foo : $@convention(thin) (@owned S) -> () {
    bb0(%0 : $S):
        %1 = function_ref @bar1 : $@convention(thin) (@owned S) -> @owned S
        %2 = copy_value %0 : $S
        %3 = apply %1(%2) : $@convention(thin) (@owned S) -> @owned S
        %4 = struct_destructure $3 : $Foo
        %5 = destructure_result %4 : #Foo.x
        %6 = destructure_result %4 : #Foo.y
        %7 = function_ref @bar2 : $@convention(thin) (@owned Builtin.NativeObject) -> ()
        apply %7(%5) : $@convention(thin) (@owned Builtin.NativeObject) -> ()
        destroy_value %6 : $Builtin.NativeObject
        dealloc_stack %1 : $*S
        destroy_value %0 : $S
    }

yielding optimal SILGen code that can be optimized down to 2 `destroy_value`,
one on each field of `%0`.
