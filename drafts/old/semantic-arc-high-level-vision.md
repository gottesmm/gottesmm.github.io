---
layout: page
title: Semantic ARC High Level Proposal
categories: proposals old
---

## Preface

For Swift 4, we plan to introduce "Semantic ARC", a major overhaul of ownership
in SIL. This document outlines the problems solved by "Semantic ARC", how it
works, and an engineering plan for implementing it.

## The Problem and the Goal

As defined today, SIL represents retain/release operations as independent
operations, without surfacing the relationships that would the most aggressive
optimizations and fast static verification. Semantic ARC enhances SIL to
preserve crucial information, allowing us to:

1. Quickly determine the source of any ARC errors from the optimizer and SILGen.
2. Implement a new, simpler ARC optimizer that will be faster and more
   maintainable.
3. Eliminate retain/release traffic more aggressively using semantic pairing
   information.
4. Lay the groundwork for the representation of other forms of ownership, such
   as move semantics, in SIL.

## The Engineering Plan

The engineering plan involves the following steps:

1. Replace Low Level Dataflow ARC Operations with High Level SSA ARC operations.
2. Add ownership conventions to block arguments and terminators.
3. Implement a verifier that ensures all paired ownership operations have
   compatible conventions.
4. Implement an algorithm that verifies that all ownership operations are paired
   exactly once along paths through the program.
5. Represent Address Only Types using SSA values instead of Memory Locations.

Each of these steps will be described in detail by focused mini-proposals sent
to swift-dev.
