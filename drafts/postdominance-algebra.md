---
layout: page
title: The Post Dominance Algebra
categories: draft
---

{% raw %}
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
\newcommand{\Z}{\mb{\text{Z}}}
\newcommand{\and}{\text{ and }}
$$
{% endraw %}

## Summary

This document is based off the Program Path Topology document. It is very
preliminary and I am just joting ideas down at this point.

## Preliminary Definitions

Let $$\BB$$ be the set of all blocks in a function.

Given an acyclic program, define a path in that program as a function,

$$
   f \colon \N -> \BB
$$

Consider the set $$\P$$ of all paths through a specific acyclic program. Given
$$A \in \BB$$, one can define the pointed set of paths to the end of the program
through $$A$$ by defining a function

$$
   \phi_A(f) \colon \P -> \set{\emptyset} + \Z
$$

As a quick formalization, we ignore any non-reachable blocks by ignoring the
kernel of $$\phi_A$$ and work on equivalence classes. We are going to gloss over
this detail for now to ease our notation. Thus this function when precomposed
with any $f \in \P$ results in:

$$
  f(n + \phi_A(f))(0) = A
$$

This is just a translation, i.e. nothing crazy. Thus define the pointed path set
at $$A$$ as,

$$
   \P_A = \set{ f \circ \phi_A(f) \colon \Z \to \BB }
$$

--------------------------------------------------------------------------------

Just to back up the big thing that I want to talk about here is that: Given any
block B reachable from A, one can extend the block B to a set of blocks
JointPostDom(B) that together joint post-dominate A and are not reachable from
each other. The extra blocks are essentially extending the space of paths until
it completes the pointed path space.

All of this really reminds me of Linear Algebra (and Differential Geometry at a
higher level):

1. The program is the manifold.
2. The set of paths is the Lie Group.
3. The pointed path sets are the Lie Algebras.
4. The operation is something related to reachability.

The last point is the most interesting to me since the reachability matrix of an
adajacency matrix is the exponential of A, i.e.:

$$
  R(A) = \sum_{i=1}^n A^i
$$

where $$n$$ is the number of nodes in the graph. This relationship flows from
the adjacency matrix being the set of one paths in a graph. One could even
imagine a countably infinite such exponential.

So one could imagine going from the lie algebra of pointed path sets (described
by an adjacency matrix) going through an exponential to compute reachability on
the manifold. Or something like that.

Could one define a "connection" ala differential geometry here? Not sure.
