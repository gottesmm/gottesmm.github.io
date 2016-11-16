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

  a. store_borrow
  b. begin_borrow
  c. tuple_destructure, struct_destructure, destructure_result

These are necessary to express the following operations:

1. Passing an `@guaranteed` value to an `@in_guaranteed` argument without
   copy. (store_borrow)
<!-- 2. Passing an `@owned` value as an `@inout` argument without
   copy. (store_borrow [inout], end_inout_borrow) -->
3. Copying a field from an `@owned` aggregate without consuming the
   aggregate. (begin_borrow)
4. Passing an `@owned` value as an `@guaranteed` argument without
   copy. (begin_borrow)
5. Performing a destructure [take] operation when emitting multiply operations
   in a statement. (tuple_destructure, struct_destructure, destructure_result).

# Definitions

## store_borrow

Define a `store_borrow` instruction as:

    %borrowed_y = store_borrow %x to %y : $*T
    ...
    end_borrow %borrowed_y from %x : $*T, $T

      =>

    store %x to %y
    ...
    end_borrow %x from %y

`store_borrow` is used to forward `@guaranteed` values as `@in_guaranteed`
arguments. Without a `store_borrow`, this can only be expressed via an
inefficient `copy_value` + `store` sequence:

    sil @g : $@convention(thin) (@in_guaranteed Foo) -> ()
    sil @f : $@convention(thin) (@guaranteed Foo) -> () {
    bb0(%0 : $Foo):
      %1 = function_ref @g : $@convention(thin) (@guaranteed Foo) -> ()
      %2 = alloc_stack $Foo
      %3 = copy_value %0 : $Foo
      store %3 to [init] %2 : $Foo
      apply %1(%2) : $@convention(thin) (@in_guaranteed Foo) -> ()
      %4 = load [take] %2 : $*Foo
      destroy_value %4 : $Foo
      dealloc_stack %2 : $Foo
      ...
    }

But using `store_borrow` yields the following SIL.

    sil @g : $@convention(thin) (@in_guaranteed Foo) -> ()
    sil @f : $@convention(thin) (@guaranteed Foo) -> () {
    bb0(%0 : $Foo):
      %1 = function_ref @g : $@convention(thin) (@guaranteed Foo) -> ()
      %2 = alloc_stack $Foo
      %3 = store_borrow %0 to %2
      apply %1(%3) : $@convention(thin) (@in_guaranteed Foo) -> ()
      end_borrow %3 from %0
      dealloc_stack %2 : $Foo
      ...
    }

<!--
## store_borrow [inout] and end_inout_borrow

Define `store_borrow [inout]` and `end_inout_borrow` as follows:

    %2 = store_borrow %1 to [inout] %0 : $*T
    apply %f(%2) : $@convention(thin) (@inout T) -> ()
    %3 = end_inout_borrow %2 from %1 : $*T, $T

    =>

    store %1 to %0 : $*T
    apply %f(%0) : $@convention(thin) (@inout T) -> ()

A `store_borrow [inout]` instruction is used to convert `@owned` values to
`@inout` parameters without performing a copy. The `end_inout_borrow` represents
the end of the mutable borrow scope and lowers to a written back value. Today this can only be expressed
using a `copy_value`+`store`+`load`+`destroy_value` sequence:

    sil @g : $@convention(thin) (@inout Foo) -> ()
    sil @f : $@convention(thin) (@owned Foo) -> () {
    bb0(%0 : $Foo):
      %1 = function_ref @g : $@convention(thin) (@guaranteed Foo) -> ()
      %2 = alloc_stack $Foo
      %3 = copy_value %0 : $Foo
      store %3 to [init] %2 : $*Foo
      apply %1(%2) : $@convention(thin) (@inout Foo) -> ()
      %4 = load [take] %2 : $*Foo
      destroy_value %4 : $Foo
      dealloc_stack %2 : $Foo
      destroy_value %1
      ...
    }

Using `store_borrow [inout]` and `end_inout_borrow` (see below), we can express
the same SIL without a copy as:

    sil @g : $@convention(thin) (@inout Foo) -> ()
    sil @f : $@convention(thin) (@owned Foo) -> () {
    bb0(%0 : $Foo):
      %1 = function_ref @g : $@convention(thin) (@guaranteed Foo) -> ()
      %2 = alloc_stack $Foo
      %3 = store_borrow %0 to [inout] %2 : $*Foo
      apply %1(%3) : $@convention(thin) (@inout Foo) -> ()
      %4 = end_inout_borrow %3 from %0 : $*Foo, $Foo
      dealloc_stack %2 : $Foo
      destroy_value %1
      ...
    }
-->

## begin_borrow

Define a `begin_borrow` instruction as:

    %borrowed_x = begin_borrow %x : $T
    ...
    end_borrow %borrowed_x from %x : $T, $T

or for an address:

    %borrowed_x = begin_borrow %x : $*T
    ...
    end_borrow %borrowed_x from %x : $*T, $*T

A `begin_borrow` instruction represents the beginning of a new +0 scope for a
specific value and implies that a value will remain live until a paired
end_borrow instruction is visited. This instruction is necessary for verifying
ownership in semantic SIL since:

1. It allows for `@owned` values to be converted to `@guaranteed` values to be
   passed to `@guaranteed` arguments.
2. It allows for the projection of copies of subfields from an object without
   the need for a complicated destructuring operation.

## tuple_destructure, struct_destructure, destructure_result
