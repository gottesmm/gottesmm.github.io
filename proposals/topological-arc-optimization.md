---
layout: default
title: Topological ARC Optimization (Mathy Version)
categories: proposals
---

# Summary

This document defines a new form of ARC optimization based on topology. We use
topology to define single entry multiple exit regions. For those who are
mathmatically inclined, see
the [Program Path Topology](proposals/program-path-topology.html) document.

This document is intended for compiler engineers. The goal is to reduce ARC
optimizations to set operations on regions of instructions. We show each
individual operation with examples and then show how we will implement it using
a single dataflow pass.

# ARC Optimization Examples

There are 3 basic forms of ARC optimization. We first list them below and then
display how each can be reduced to a set operation.

1. Lifetime Extension Optimization.
2. Guaranteed Lifetime Optimization.
3. Consumed Optimization.

**NOTE** In the following we are only considering instructions in basic
blocks. We can extend this analysis to CFGs later.

## Lifetime Extension Optimization

The lifetime extension pattern looks as follows:

    %x2 = copy_value %x1     (Begin Ownership Region 1
    ... (code block 1)
    destroy_value %x2        (End Ownership Region 1
    ... (code block 2)
    %x3 = copy_value %x1     (Begin Ownership Region 2
    ... (code block 3)
    destroy_value %x3        (End Ownership Region 2

Notice how we have two ownership regions. Naturally if we consider the sets of
instructions in each of the ownership regions, we can consider the ownership
region defined by the **union** of code block 1/2 and code block 3. Such a
lifetime region would ensure that the object's lifetime is still maintained over
both regions implying we can merge the two regions into a third ownership
region:

    %x2 = copy_value %x1     (Begin Ownership Region 3
    ... (code block 1)
    ... (code block 2)
    ... (code block 3)
    destroy_value %x2        (End Ownership Region 3

**NOTE** Notice that there are no requirements on ``%x1's`` ownership.

## Guaranteed Lifetime Optimization

The guaranteed lifetime optimization is defined by the following initial
condition:

    %x2 = copy_value %x1    (Begin Ownership Region 1)
    ... (code block 1)
    %x3 = copy_value %x2    (Begin Ownership Region 2)
    ... (code block 2)
    destroy_value %x3       (End Ownership Region 2)
    ... (code block 3)
    destroy_value %x2       (End Ownership Region 1)

It is natural to see that ``%x1``'s lifetime will be maintained by Ownership
Region 1 even when we are in Ownership Region 2. Thus we can safely eliminate
the inner copy_value, destroy_value pair. In terms of ownership regions, this is
equivalent to performing a union of the two regions. Since Ownership Region 1 is
completely subsumes Ownership Region 2, the end result is just Ownership Region
1, i.e.:

    %x2 = copy_value %x1    (Begin Ownership Region 1)
    ... (code block 1)
    ... (code block 2)
    ... (code block 3)
    destroy_value %x2       (End Ownership Region 1)

## Consumed Optimization

The consumed optimization is defined by the following initial conditions:

    %x2 = copy_value %x1     (Begin Ownership Region 1)
    ... (code block 1)
    %x3 = copy_value %x2     (Begin Ownership Region 2)
    ... (code block 2)
    consuming_apply(%x3)     (End Ownership Region 2)
    ... (code block 3)
    destroy_value %x2        (End Ownership Region 1)

Notice how if there are no uses of `%x2` in code block 3 or we can prove that
the destroy_value of `%x2` is not the last release of `%x2`, we can safely
perform a lifetime intersection operation on the inverse-program paths defined
by the two regions and a union operation on the program paths defined by the two
regions. The inverse-program path of Ownership Region 1 consists of all
instructions before the destroy_value. The inverse-program path of Ownership
Region 2 consists of all instructions before the consuming apply. This means
that the final inverse-program path for the new region will start at the
beginning of ownership region 1 and go to the end of the program. The program
paths of ownership region 1 starts at the first copy_value and for ownership
region 2 starts at the second copy value. Thus their union path is the first
copy_value. Thus the final code region will start at Begin Ownership Region 1
and will end at End Ownership Region 2. Thus we will have:

    %x2 = copy_value %x1     (Begin Ownership Region 1)
    ... (code block 1)
    ... (code block 2)
    consuming_apply(%x3)     (End Ownership Region 2)
    ... (code block 3)

# Implementation

Loop over the whole CFG gathering up producers/consumers. This is known
trivially. Using the APIs from semantic ARC given just 1 producer or consumer,
we can get the entire rest of the producer/consumer set. I imagine we would have
a set of visited instructions to ensure that we visit these sets once. Using the
use-def lists we are then able to construct heirarchical relationships among
these joint dominating sets of producers/consumers. This essentially creates
hierarchical regions. Given these hierarchical regions, there are certain
"legal" moves that we can perform:

1. A set A contained in a set B for which B completely consists of copy_values
   and A completely consists of copy_values implies that as a "lifetime region",
   A can always be collapsed into B. I call this the "union" optimization
   (i.e. the union of a lifetime with another lifetime contained within the
   first is just the first).
   
2. Given a set A contained completely within a set B such that A is a "take"
   set (i.e. no destroy_value), we can intersect B into A. This is the
   "intersection" optimization.
      
3. A set A consisting of producers and destroy_values and a set B
   consisting of copy_values and consumers can be merged into one
   composite region that consists of A and B. This is the "cartesian
   property" optimization.
         
This will be much cheaper than any iterative dataflow that we may perform and
requires /no/ side-effects analysis meaning it will be really fast. It
additionally can allow us to perform other operations by determining
profitability based off of the hotness of code. In fact, we could also make a
pbqp like optimizer upon this since this is very similar to register allocation.
