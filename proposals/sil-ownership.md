---
layout: page
title: SIL Ownership Model (Newer Full Document)
categories: proposals
---

# Preliminary Information

Here we quickly define some terms and give some information about Ownership at the
SIL level that we will be referring to below.

## Producers/Consumers of SSA values

**TODO** Describe how there are non-side-effect having instructions and other
side-effect having forwarding instructions.

**TODO** Show how guaranteed also has this property, it is just the consumer is
implicit at the end of a function so we ignore it. This opens up the ability to
create regions of guaranteed parameters and sets up the ability to talk about
load_strong [guaranteed] in the load_strong document (which creates a guaranteed
region).

# Summary

In this document, we propose the creation of a SIL Ownership Model. We assume
that the reader is already familiar with SIL and has read the Definitions
section above.

By implementing this model, we will be able to statically ensure that for any
SIL SSA value, the following is true:

1. All "producers" and "consumers" of a given value are able to be found
   trivially via an API. This will have a verifier to ensure that this property
   holds true.
2. All "producers" and "consumers" of a SILValue will have ownership attributes
   that are compatible with their respective matching "consumer" and "producer".
3. All "producer" ("consumer") will have exactly one matching
   "consumer" ("producer") along any path to the end (from the start) of the
   program. This will be verified.

We do this by performing the following transformations:

1. Replace Low Level ARC Operations that require dataflow for analysis with High
   Level ARC operations that can be analyzed using use-def lists.
2. Add Ownership Conventions to Block Arguments and terminators and Add the
   Guaranteed Region end convention.
2. Define the "Producer/Consumer" discovery algorithm and enforce via a verifier
   that the algorithm succeeds for any leaf type of any SILValue.
3. Write a verifier that ensures that all Producer/Consumers are matched with
   producers/consumers at the same level of the SCC tree in each function's
   CFG. This essentially comes down to ensuring that there are no semantic
   pairings in and outside of SCCs that are paired with each other. If there is
   a pairing, it must be mediated via SILArguments.
4. Define the "Ownership Compatibility Verifier" that ensures that the
   producers/consumers that we have found have compatible ownership
   semantics. Enforce this at the SIL level.
5. Define the "Singular Matching Algorithm" for "Producer/Consumer" that are
   able to verify that along any path through the program, a producer/consumer
   is only matched with one other consumer/producer.
6. Represent Address Only Types via SSA values instead of Memory Locations to
   enable Address Only Types to participate in this system.

Once these five points have been implemented, Ownership semantics at the SIL
level will be statically verified. We go through each one of the points below in
an abridged form to give a general "gist" of the implementation strategy and how
everything fits together. Each will be set out in mini-proposals that provide in
depth implementation strategies and that will be sent out at a later point in
time. The first mini-proposal (load_strong, store_strong) is sent out in as a
separate attachment alone side this proposal.

Once this has been done, the optimizer will gain the following benefits:

1. ARC Code Motion will be able to be much more aggressive due to load_strong,
   store_strong.
2. Since producers/consumers can be found at the leaf type level, the compiler
   pipeline can be changed such that splitting of aggregates is no longer
   required for ARC optimization to be effective.
3. The ARC optimizer will be able to reason about semantic pairings ensuring
   that miscompiles will not occur due to eliminating non-semantic
   pairings. This will enable us to be very aggressive when optimizing
   guaranteed values.
4. Since Address Only Types will be values, they will be able to participate in
   ownership verification and a simple copy propagation algorithm can be
   implemented that does not involve dataflow.
5. Since block arguments and terminators will have conventions, we will be able
   to perform a version of function signature optimization at the function level
   where we try to change @owned terminators/block arguments of regions into
   @guaranteed regions. This will then allow us to in a natural way extend our
   aggressive guaranteed optimizations to subregions of the CFG.

# Replace Low Level Dataflow ARC Operations with High Level Use-Def ARC Operations

Most of the main ARC optimization primitives today can not be analyzed except
via dataflow. This makes static verification of properties such as pairing,
compatibility of ownership requirements more difficult and require
dataflow. By changing these operations to use high level operations that pass
requirements along use-def edges, we eliminate these problems.

The specific changes that we propose are:

1. Implement the store_strong, load_strong operations.
2. Replace strong_retain, retain_value with a copy_value operation.
3. Replace strong_release, release_value with a destroy_value operation.

## Add strong_store/load_strong

The reason to perform this change is that combining those operations into single
SIL instructions allows the optimizer to avoid miscompiles that can occur due to
releases being moved into the partial initialization region in between a
load/retain, load/release. As a result of this, we are able to be more
aggressive when performing code motion of ARC operations since we no longer have
to worry about a release triggering a deinit before ownership of an object is
fully taken.

## strong_retain/retain_value => copy_value

The main reason to perform this change is that it enables the ownership effect
of the reference count increment to be modeled as the copy_value returning an
@owned value. This allows one to then require that along all paths through the
program after the retain, there is some consuming operation in the use-def web
from the retain's result that consumes the retain without performing
dataflow. This will enable us to reduce compile time and to have a way of
speaking at the SIL level of semantic pairings.

As an additional bonus, by doing this we remote an unnecessary distinction in
between non-trivial types and reference types provided by release_value,
strong_release. IRGen is more than capable of understanding which SILTypes
require just a strong_release and which require more in depth handling ala
release_value.

## strong_release/release_value => destroy_value

The simple reason to perform this change is to require these primitives to match
in name copy_value. We also eliminate the similarly unnecessary non-trivial
type/reference semantic type as by transforming strong_retain/retain_value =>
copy_value.

# The "Producer/Consumer Discovery" Algorithm and Verifier

Once all data-flow only ARC operations at the SIL level have been eliminated, we
then can begin to implement the "Producer/Consumer Discovery" algorithm. The
high level purpose of this algorithm is that for any SSA value, we want it to be
a property of the IR that we can for any SSA value look through any instructions
that forward ownership (for instance side-effect free values such as
aggregate/projection/casting operations) in the def tree of the value until we
find a def that we can not forward and through all uses through the use tree of
the value until we can find a value that does not forward and consumes the
ownership.

The way that this will be implemented is that we will first construct a
SILVisitor that for a given result says how the result's ownership propagates to
the instruction's users and vis-a-versa. If an instruction consumes/produces,
the visitor categorises it as so. If the instruction just forwards, again the
visitor says so. If the instruction has not yet been categorized by the visitor,
the visitor returns an 'unknown' result. This will be placed behind a method on
ValueBase that will cause the ValueBase instance to construct the visitor and
pass itself into the visitor. As an alternative to this implementation, we could
also use a pure virtual method on ValueBase as well. *NOTE* At the beginning the
visitor will return unknown for all instructions.

Then we create a verifier that visits each SILValue in the program in the
SILVerifier and makes sure that for each SILValue, we are able to look through
the transitive def list of the user and never find an instruction that is
unlabeled. We do the same for consumers with the transitive use list of the
value. Of course, at the beginning (as mentioned above) there will be able to
express how their results are forwarded. To work around this issue, we maintain
a white-list in the verifier of instructions which the verifier if it finds an
unknown value for should assert.

Thus our implementation will work by teaching the SILVisitor how to handle each
ValueBase one by one and as we add support and feel confidant that the verifier
will not trip, we add ValueBase kinds to the whitelist which will cause the
verifier to ensure that we never lose the ability to reason about said
instruction or if we missed any specific cases during implementation.

**NOTE** We potentially could just verify each instruction individually and make
sure that each can be understood. I have not thought about if this would work
(am in a bit of a rush), so I am suggesting the full solution for now.

# Ensure all retain/release are at the same level of SCCs (i.e. no semantic pairings over loop boundaries)

**TODO** Might fold this into the previous section maybe? I use this invariant
in the "Singular Pairing Algorithm".

# The "Ownership Compatibility Verifier"

The next step is that we need to ensure that all consumers and producers have
compatible ownership conventions. This means that:

1. @owned producer defs are matched with @owned consumer uses.
2. @guaranteed producer defs are matched with @guaranteed consumer uses.

Since we know that for any SSA value in SIL, we can find consumers/producers,
this reduces to writing a verifier that loops through a SILFunction and comes up
with a list of producer/consuming use/defs. This via the previous section is
able to be known statically at compile time. Then for each producer (consumer),
we find its set of consumers (producers) and we verify that the producer's
(consumer's) convention matches with all of the producer (consumer) set's
values. The reason why we have to visit both producers/consumers is that often
time in SIL, ownership convention pairings can occur in between sets of
producers/consumers that joint dominate each other. By visiting each
individually, we handle this case without needing to (at this point) determine
joint domination sets.

*NOTE* This can be implemented in parallel with the "Producer/Consumer"
discovery algorithm by ignoring instructions for which the "Producer/Consumer"
discovery algorithm returns unknown and are not in the white list for the
"Producer/Consumer" discovery algorithm.

# The "Singular Matching Algorithm"

After all of this work, we know that:

1. All of the producer/consumer pairings in our program can be found.
2. All producer and consumer pairings have compatible ownership conventions.
3. All producer/consumer pairings are at the same SCC level of the CFG.

But, we still do not know if along all paths through the program each producer
is matched to just one consumer and each consumer is matched with just one
producer.

In order to determine this, for a specific producer, we take its consumer set
and then determine if the consumer set is a minimal joint-postdominating set and
for each consumer if its producer set if a minimal-dominating set. In terms of
implementation this implies proving the following for a producers consumer set
(ignoring differences in between dominance, post-dominance):

1. No consumer is reachable from any other consumer. Since all consumers are at
   the same SCC level, we do not need to consider any back-edges implying that
   we can just traverse from each consumer's block to the producer block and
   make sure that we do not find any other consumers. Using a cache this could
   be made faster and we could use more efficient techniques as well if we need
   to.

2. The least common ancestor of the consumers in the post-dominance tree does
   not have any children that are not reachable from an element of the consumer
   set.

# Represent Address Only Types via SSA values instead of Memory Locations

In terms of attempting to represent address only types as SSA values.
