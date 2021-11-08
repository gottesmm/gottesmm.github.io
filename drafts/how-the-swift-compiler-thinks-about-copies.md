---
layout: page
title: How the Swift Compiler thinks about Copy Insertion
categories: draft
---

In Swift, all non-trivial values are able to have their uses partitioned into a
set of lifetime ending (sometimes called consuming) and non-lifetime ending
uses. An example of a non-lifetime ending use could be a function that takes in
a class at +0 and adds an integer instance variable in the class to a running
total. An example of a lifetime ending use could be a move like operation, being
passed as an argument to an initializer or setter, or being passed as an
argument to a function with an explicitly marked __owned parameter:

```
func myFunc(_ k: Klass) -> () // k is passed at +0. Non lifetime ending.
x.myK = k // k is passed at +1 into the setter for myK. Lifetime Ending.
func myOwnedFunc(_ k: __owned Klass) -> () // k is forced to be passed at +1.
```

At the lifetime ending points the compiler must be able to guarantee that the
value is live and we can pass off ownership of it to a different piece of code
(passing at +1). This causes the compiler to need to insert copies before any
consuming use that a 2nd consuming use is reachable from. Along paths where
there no uses are lifetime ending, the compiler will insert destroys right after
the last non-lifetime ending use along that path.

Lets spend a second going through an example of where the swift compiler needs
to insert copies. Consider the following Swift code:

```
func doSomething() -> Bool { ... }
func useX(_ k: Klass) { ... }
func secondUseX(_ k: Klass) { ... }

let x = Klass()
if doSomething() {
  useX(x)
} else {
  secondUseX(x)
}
doSomethingElse()
```

To the compiler this actually looks like the following two directed graph:

[image](how-the-swift-compiler-thinks-about-copies-img1.png)
[image](how-the-swift-compiler-thinks-about-copies-img2.png)

The left hand side graph is what the compiler sees before it would insert any
destroys. Looking at that graph, the compiler would look at the uses of x and
see that none of its uses (useX(X) and secondUseX(X)) consumed x implying that
no extra copies are required for x to remain alive at useX, secondUseX. Thus the
compiler will insert destroys right after useX and secondUseX resulting in the
RHS graph.

What happens if we introduce a lifetime ending function into the mix:

```
func doSomething() -> Bool { ... }
func useX(_ k: Klass) -> () { ... }
func secondUseX(_ k: Klass) -> () { ... }
func consume(_ k: __owned Klass) -> () { ... }

let x = Klass()
if doSomething() {
  useX(x)
} else {
  secondUseX(x)
  consume(x)
}
doSomethingElse()
```

In this case, rather than having a destroy of x after secondUse, we will instead
pass off x to consume at +1 yielding the following graph on the LHS:

[image](how-the-swift-compiler-thinks-about-copies-img3.png)
[image](how-the-swift-compiler-thinks-about-copies-img4.png)

the compiler is able to know that it should do this since it knows from the
signature of consume that its first parameter is __owned meaning it is at +1.

So lets say, we add... (wait for it)... one more consume! In that case, we get
the graph on the RHS above. In this case, notice how since we have to consume a
value twice along the right path of the diamond, *we have to insert a copy*! Our
implementation of move only variables is informed by that realization... that we
can use the compiler’s already present notion that a copy would be needed in
such a situation to instead emit a diagnostic error and thus implement move only
semantics.
