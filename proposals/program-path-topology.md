---
layout: proposal
title: The Program Path Topology
categories: proposals
---

$$
\newcommand{\set}[1]{\left\{#1\right\}}
\newcommand{\mb}[1]{\mathbf{#1}}
\newcommand{\P}{\mb P}
\newcommand{\successor}{\mb{\text{succ}}}
\newcommand{\magnitude}[1]{\left|#1\right|}
\newcommand{\branch}{\mb{\text{Branch}}}
\newcommand{\Bool}{\mb{\text{Bool}}}
\newcommand{\BB}{\mb{\text{BB}}}
\newcommand{\N}{\mb{\text{N}}}
\newcommand{\and}{\text{ and }}
$$

# Summary

This document defines the program path topology and program post-path
topology. These two topologies subsume the concepts of dominance and
post-dominance by including potential single-block backedges that imply one
needs to consider control dependence. Thus these topologies form a better way to
think about optimizing programs in a CFG.

**NOTE** In our model, we explicitly disallow any implicit CFG edges.

# Preliminary Definitions

For a given program, imagine the infinite set of instructions of the program
completely unrolled. Then define the program set $$\P$$ as the countably
infinite ordered set of such instructions. Naturally there is a classification
function $$\successor \colon \P \to \P^*$$ which maps an instruction $$ i \in
\P$$ to its set of successor instructions. Then define the set of branch
instructions $$\branch$$ as:

$$
   \branch = \set{p \in \P \colon \magnitude{\successor(i)} \neq 1}
$$

The define a control flow free path as a set $$\phi \subset \P^*$$ such
that,

$$
   \phi = \set{i \in \P \colon i \notin \branch \and \exists i' \in \phi \owns i \in \successor(i')}
$$

Naturally $$\successor$$ defines an ordering on $$\phi$$ and there must exist at
most one instruction $$i \in \phi$$ such that $$b \in \branch$$ and
$$\successor(i) = \set{b}$$. Naturally we must have that $$b \notin \phi$$ since
$$b$$ does not have one successor.

Define $$\Pi$$ as the set of all $$\phi \subset \P^*$$. **TODO** Fill in easy
argument how we come up with maximal such paths $$\hat\phi$$. Now that we have
our maximal paths, define the set of basic blocks $$\BB$$ in $$\P$$ as:

$$
   \BB = \set{\successor(\hat\phi) + \hat\phi : \hat\phi \in \Pi}
$$

In words this means that the set of basic blocks consists of maximal control
flow free paths unioned with the set consisting of that sets branch
instruction. We naturally can then extend the $$\successor$$ function on our
paths then to our basic blocks. **TODO** Include this definition.

# Program Path Topology

Now we define the program path topology. Consider an instruction $$i \in
\P$$. Naturally there is a set of paths $$p \subset \P^*$$ such that all
instructions $$i' \in p$$ are later than $$i$$. These sets of paths naturally
form a topology since if we form the intersection of any such paths, we are
still in the same set and if we form a union of any such paths, we also stay in
the same set. We can do the same thing considering inverse paths.

Naturally combining together
