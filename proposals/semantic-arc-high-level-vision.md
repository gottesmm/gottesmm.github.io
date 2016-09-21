
# Preface

This year a major piece of work at the SIL level is Semantic ARC. This email
contains a high level description (i.e. the vision) that describes the problems
that this model will solve and provide a broad overview of the engineering plan
that will be needed to achieve this.

# The Problem and the Goal

A problem in SIL today is that SIL as an IR is defined at too low of level for
the definition of a statically verifiable SIL ownership model.  The lack of such
a model precludes the ability to aggressivly optimize ARC or write a fast
SIL ownership verifier.

Semantic ARC solves this general problem by making changes to SIL that allow for
such a model to be defined and creating verifiers that will enable the ownership
model to be statically verified at compile time. This will result in:

1. The ability to quickly triage and determine the source of ARC errors from the
   optimizer and SILGen.
2. More aggressive ARC optimization using additional semantic information such
   as "semantic pairings".
3. Creating the necessary information for other forms of ownership to be added
   to the SIL IR such as move semantics.

# The Engineering Plan

The engineering plan for this involves implementing several transformations on
the IR that can occur inter-dependently. Each of these steps will have a
mini-proposal (the first of which will be sent out shortly) that describes each
specific step in more detail. The specified steps are:

1. Replace Low Level dataflow ARC Operations with High Level SSA ARC operations.
2. Add Ownership Conventions to block arguments and terminators.
3. Implement a simple algorithm based on use-def traversal for determining
   consumers/producers for any SSA value and use this to verify that
   consumers/producers of SSA values have matching semantics.
4. Implement an algorithm that verifies that for any specific producer/consumer,
   along any path through the program, a producer/consumer is only matched with
   one other consumer/producer.
5. Represent Address Only Types via SSA values instead of Memory Locations to
   enable Address Only Types to participate in this system.

