---
layout: page
title: An Exploration of Floating Point Manifolds
categories: draft
---

{% raw %}
$$
\newcommand{\set}[1]{\left\{#1\right\}}
\newcommand{\mb}[1]{\mathbf{#1}}
\newcommand{\mbb}[1]{\mathbb{#1}}
\newcommand{\P}{\mb P}
\newcommand{\C}{\mb C}
\newcommand{\B}{\mb B}
\newcommand{\FP}{\mbb F_{32}}
\newcommand{\successor}{\mb{\text{succ}}}
\newcommand{\magnitude}[1]{\left|#1\right|}
\newcommand{\branch}{\mb{\text{Branch}}}
\newcommand{\Bool}{\mb{\text{Bool}}}
\newcommand{\BB}{\mb{\text{BB}}}
\newcommand{\N}{\mb{\text{N}}}
\newcommand{\and}{\text{ and }}
$$
{% endraw %}

## Abstract

Floating point has been a point of facination to me for a long
time. It has amazed me that no one has tried to study it in a more
formal way. I think that the reason why this is true is that floating
point is sort of mysterious and hard to understand. Lets start by
demistifying what a floating point number is...

1. Insert description of floating point numbers including
formula. Talk about how it is a logarithmic lattice and show how the
lattice width grows as we hit powers of 2. Then show how we define it
as a projective space and then talk about how real floating point is
actually done on equivalence classes due to NaN, Infinity.

2. Show how that causes associativity to break down, but preserves
commutativity.

3. Talk about how all of these properties are interesting because
associativity is normally saved before commutativity.

Now that we know what floating point numbers are define the set of
IEEE754 single precision floats as the set $$\FP$$. Naturally, we can
then define $$\B$$ as the subsets of $$\FP$$ that are equidistance
from their nextf, prevf neighbors and thus the set of cut points as,

$$
  \C = \FP \bigcap \overline{\cup \B}
$$

In words, these cutpoints $\C$ are the places where the binaid
distance changes.
