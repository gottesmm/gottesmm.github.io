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

# Why add these operations
