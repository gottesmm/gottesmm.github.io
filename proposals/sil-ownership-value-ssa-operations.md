---
layout: proposal
title: SIL Ownership SSA Value Operations
categories: proposals
---

# {{ page.title }}

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Summary](#summary)

<!-- markdown-toc end -->

# Summary

This document proposes:

1. Adding a new flag [mut] to load_borrow to allow for load_borrow to also refer
   to mutable borrows.
2. The addition of the following new SIL instructions:
   a. store_borrow
   b. begin_borrow
   c. end_mut_borrow

This will allow for clear SSA ownership transfer in between:

1. Converting +1 rvalues to +0 immutable and mutable rvalues.
2. Converting +1 lvalues to +0 immutable and mutable rvalues.
3. Converting rvalues to +0 rvalues.

# Definitions

## store_borrow

Define a `store_borrow` instruction as:

    %borrowed_y = store_borrow %x to %y : $*T
    ...
    end_borrow %borrowed_y from %x : $*T, $T

      =>

    store %x to %y

    fix_lifetime(%x)
    fix_lifetime(%y)

In words, [INSERT_SEMANTICS]

A `store_borrow` instruction is necessary to be able to represent the forwarding
of values with `@guaranteed` and `@owned` ownership as arguments with
`@in_guaranteed` ownership.

## begin_borrow

Define a `begin_borrow` instruction as:

    %borrowed_x = begin_borrow %x : $T
    ...
    end_borrow %borrowed_x from %x : $T, $T

or for an address:

    %borrowed_x = begin_borrow %x : $*T

    end_borrow %borrowed_x from %x : $*T, $*T

A `begin_borrow` instruction represents the beginning of a new +0 scope
for a specific value and implies that a value will remain live until a paired
end_borrow instruction is visited. This instruction is necessary for verifying
ownership in semantic SIL since:

1. It allows for `@owned` values to be converted to `@guaranteed` values to be
   passed to `@guaranteed` arguments.
2. It allows for the projection of copies of subfields from an object without
   the need for a complicated destructuring operation.
