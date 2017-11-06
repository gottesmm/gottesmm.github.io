---
layout: page
title: The Perfect Shuffle Optimization
categories: drafts
---

$$
\newcommand{\set}[1]{\left\{##1\right\}}
\newcommand{\mb}[1]{\mathbf{##1}}
\newcommand{\P}{\mb P}
\newcommand{\successor}{\mb{\text{succ}}}
\newcommand{\magnitude}[1]{\left|##1\right|}
\newcommand{\branch}{\mb{\text{Branch}}}
\newcommand{\Bool}{\mb{\text{Bool}}}
\newcommand{\BB}{\mb{\text{BB}}}
\newcommand{\N}{\mb{\text{N}}}
\newcommand{\and}{\text{ and }}
$$

## LLVM and the Perfect Shuffle

A shuffle is a specific type of instruction on many CPU architectures that
re-arranges elements in short vectors. These are not permutations since we allow
for the mappings to not be injective. Each one of these instructions has a
specific latency cost. Determining the shortest max latency chain of
microarchitecture specific shuffle for a specific "abstract shuffle" is the
"Perfect Shuffle Problem". Today, there is not a general solution to this
problem, in fact, the only place in LLVM where perfect shuffles are attempted
are in the ARM backend. In this article, I propose a solution to the perfect
shuffle problem.

## Representing Shuffles

In this document, we represent shuffles as NxN matrices with the condition that:

1. Any row must contain exactly one element set to 1 and all other elements must
   be 0.
2. Columns are allowed to have as many elements that are 1.

So for example the matrix:

$$
   \begin{bmatrix}
    0 & 1 & 0 & 0 \\
    0 & 1 & 0 & 0 \\
    0 & 0 & 1 & 0 \\
    0 & 0 & 0 & 1
   \end{bmatrix}
$$

is a valid shuffle, as is:

$$
   \begin{bmatrix}
    1 & 0 & 0 & 0\\
    0 & 1 & 0 & 0\\
    0 & 0 & 0 & 1\\
    0 & 0 & 0 & 1
   \end{bmatrix}
$$

## Bounding Shuffle

## Representing Costs in the Shuffle
