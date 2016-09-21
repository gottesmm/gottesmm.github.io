
This year a major piece of work at the SIL level is Semantic ARC. This email
contains a high level description (i.e. the vision) that describes the problems
that this model will solve and provide a broad overview of the engineering plan
that will be needed to achieve this.

The goal of the semantic ARC effort is the creation of a SIL Ownership model
that is enforced via invariants created through verifiers. This means that for
all SIL SSA values, the following is true:

1. All "producers" and "consumers" of a given value are able to be found
   trivially via an API. This will have a verifier to ensure that this property
   holds true.
2. All "producers" and "consumers" of a SILValue will have ownership attributes
   that are compatible with their respective matching "consumer" and "producer".
3. All "producer" ("consumer") will have exactly one matching
   "consumer" ("producer") along any path to the end (from the start) of the
   program. This will be verified.

This will cause the IR to be statically verifiably correct from an ownership
perspective. This will enable us to do the following:

1. The verifiers will allow for quick determination and triage of ARC errors in
   the SIL IR.
2. The optimizer will have access to information such as semantic pairings and
   will become more robust due to the verification.
3. The stage will be set for other forms of ownership to be added to the SIL IR
   such as move semantics.

The engineering plan for this involves implementing several transformations on
the IR that can occur inter-dependently. Each of these steps will have a
mini-proposal (the first of which is attached to this email) sent out that
describes in more detail the problem, the solution, and the relevant engineering
plan. The specific transformations are:

1. Replace Low Level dataflow ARC Operations with High Level SSA ARC operations.
2. Add Ownership Conventions to Block Arguments and terminators.
3. Implement a simple algorithm based on use-def traversal for determining for
   any non-trivial SSA values consumers/producers of the value. Use this
   algorithm to determine that all consumers/producers have compatible
   conventions.
4. Implement an algorithm that verifies that for any specific producer/consumer,
   along any path through the program, a producer/consumer is only matched with
   one other consumer/producer.
5. Represent Address Only Types via SSA values instead of Memory Locations to
   enable Address Only Types to participate in this system.
