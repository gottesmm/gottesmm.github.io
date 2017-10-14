---
layout: default
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

<!-- markdown-toc end -->

# Summary

This document proposes the addition of the following new SIL instructions:

1. `store_borrow`
2. `begin_borrow`

These enable the expression of the following operations in Semantic SIL:

1. Passing an `@guaranteed` value to an `@in_guaranteed` argument without
   performing a copy. (`store_borrow`)
2. Copying a field from an `@owned` aggregate without consuming or copying the entire
   aggregate. (`begin_borrow`)
3. Passing an `@owned` value as an `@guaranteed` argument parameter.

# Definitions

## store_borrow

Define `store_borrow` as:

    store_borrow %x to %y : $*T
    ...
    end_borrow %y from %x : $*T, $T

      =>

    store %x to %y

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

    sil @f : $@convention(thin) (@guaranteed Foo) -> () {
    bb0(%0 : $Foo):
      %1 = function_ref @g : $@convention(thin) (@in_guaranteed Foo) -> ()
      %2 = alloc_stack $Foo
      store_borrow %0 to %2 : $*T
      apply %1(%2) : $@convention(thin) (@in_guaranteed Foo) -> ()
      end_borrow %2 from %0 : $*T, $T
      dealloc_stack %2 : $Foo
      ...
    }

**NOTE** Once `@in_guaranteed` arguments become passed as values, `store_borrow`
will no longer be necessary.

## begin_borrow

Define a `begin_borrow` instruction as:

    %borrowed_x = begin_borrow %x : $T
    %borrow_x_field = struct_extract %borrowed_x : $T, #T.field
    apply %f(%borrowed_x) : $@convention(thin) (@guaranteed T) -> ()
    end_borrow %borrowed_x from %x : $T, $T

      =>

    %x_field = struct_extract %x : $T, #T.field
    apply %f(%x_field) : $@convention(thin) (@guaranteed T) -> ()
    
A `begin_borrow` instruction explicitly converts an `@owned` value to a
`@guaranteed` value. The result of the `begin_borrow` is paired with an
`end_borrow` instruction that explicitly represents the end scope of the
`begin_borrow`.

`begin_borrow` also allows for the explicit borrowing of an `@owned` value for
the purpose of passing the value off to an `@guaranteed` parameter.

*NOTE* Alternatively, we could make it so that *_extract operations started
borrow scopes, but this would make SIL less explicit from an ownership
perspective since one wouldn't be able to visually identify the first
`struct_extract` in a chain of `struct_extract`. In the case of `begin_borrow`,
there is no question and it is completely explicit.
