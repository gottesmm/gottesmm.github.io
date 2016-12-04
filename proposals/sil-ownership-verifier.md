---
layout: proposal
title: SIL Ownership Model
categories: proposals
---

# {{ page.title }}

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Summary](#summary)
- [Clarifying Values and Defs](#clarifying-values-and-defs)
    - [SILNode and ValueDef](#silnode-and-valuedef)
    - [ValueBundle](#valuebundle)
- [ValueOwnershipKind](#valueownershipkind)
- [Verification of Ownership Semantics](#verification-of-ownership-semantics)
    - [Use Verification](#use-verification)
    - [Dataflow Verification](#dataflow-verification)
- [Appendix](#appendix)
    - [Full Code For Initialization of Worklist](#full-code-for-initialization-of-worklist)
    - [Full Code For Worklist Algorithm](#full-code-for-worklist-algorithm)
- [Implementation](#implementation)
    - [`ValueDef::getOwnershipKind()`](#valuedefgetownershipkind)
- [Mapping ValueDef to ValueOwnershipKind](#mapping-valuedef-to-valueownershipkind)
- [Proving Def-Use Convention Correctness](#proving-def-use-convention-correctness)
- [Identifying Ownership Dataflow Errors](#identifying-ownership-dataflow-errors)

<!-- markdown-toc end -->

# Summary

This document defines a SIL ownership model and a compile time static verifier
for the model. This will allow for static compile time verification that a SIL
program satisfies all ownership model constraints.

The proposed SIL ownership model embeds ownership into SIL's SSA def-use
edges. This is accomplished by:

1. Eliminating ownership representation issues in SIL. This implies:
   
   a. Formalizing into SIL API's the difference in between defs
   (e.g. `SILInstruction`) and the values produced by defs
   (e.g. `SILValue`). This is needed to model values as having ownership and
   defs as not having ownership. The main implication here is that the two can
   no longer be implicitly convertible and the API must enforce this.

   b. Introducing a new value called a `ValueBundle` that enables multiple
   return values to be implemented using projection operations.

2. Specifying a set of ownership kinds and specifying a method for mapping a
   `SILValue` to an ownership kind.

3. Specifying constraints on all `SILInstruction`s that constrain what ownership
   kinds their operands can have.

4. Implementing a verifier to ensure that all `SILInstructions` are compatible
   with the ownership kind propagated by the `ValueDef` and that pseudo-linear
   dataflow constraints are maintained.

# Eliminating Ownership Representation Issues in SIL

All values in SIL are defined via an assignment statement of the form: `<foo> = <bar>`.
In English, we say `foo` is a value that is defined by the def
`bar`. Originally, these two concepts were distinct concepts represented by the
classes `SILValue` and `ValueBase`. All `ValueBase` defined a list of `SILValue`
that were related, but not equivalent to the defining `ValueBase`. With the
decision to represent multiple return values as instruction projections instead
of as a list of `SILValue`, this distinction in between a def and the values was
lost, resulting in `SILValue` being used interchangeably with `ValueBase`
throughout the swift codebase. This exposes several representation issues when
one attempts to add ownership to SIL:

1. Values have ownership, while the defs that define the values do not. This
   implies that defs and values *should not* be interchangeable.
2. The plan to use projections for multiple return values was never implemented
   since at the time there was no compelling need. This lack of implementation
   is an issue for representing ownership in SIL since a destructure take
   operation is needed to efficiently forward subparts of aggregates.

We propose below a series of transformation to SIL that resolves these
issues. **NOTE** A condition of this proposed transformation is that today's
textual SIL (ignoring additive changes below) should stay completely
unchanged. This is important to ensure that we do not need to update already
existing test cases.

## Defs and Values

In order to model that values, not defs, have ownership, we separate the
`SILValue` and `ValueBase` APIs.

First we rename `ValueBase` to `ValueDef`. This makes it clear from a naming
perspective that a `ValueDef` is not a value, but the def of a value.

Then we eliminate eliminate all operator methods on `SILValue` that allow one to
work with a `SILValue` as a `ValueDef` directly,

    class SILValue {
      ...
      ValueDef *operator->() const;
      ValueDef &operator*() const;
      operator ValueDef *() const;
    
      bool operator==(ValueDef *RHS) const;
      bool operator!=(ValueDef *RHS) const;
      ...
    };

Instead, we provide an explicit named method:

    ValueDef *SILValue::getDef() const

This will eliminate code like the following where a SILValue is implicitly used
as a def,

    SILValue V;
    ValueDef *Def;
    
    if (V != Def) { ... }
    if (V->getParentBlock()) { ... }

In favor of the clearer:

    SILValue V;
    ValueDef *Def;
    
    if (V.getDef() != Def) { ... }
    if (V.getDef()->getParentBlock()) { ... }

Notice how now it is explicit in the code that we are not working with the
properties of V, but rather with V's def.

In cases like the above, we want the verbosity to yield clarity. In other cases,
this makes certain very convenient APIs more difficult to use. The main example
here are the `isa` and `dyn_cast` APIs. We introduce helper functions that give
the same convenience as before but added clarity by making it clear that we are
not performing an isa query on the value, but instead the underlying def of the
value. Consider the following code using the old API,

    SILValue V;
    
    if (isa<ApplyInst>(V)) { ... }
    if (auto *PAI = dyn_cast<PartialApplyInst>(V)) { ... }

Notice how it seems like one is casting the SILValue as if it is a
ValueBase. This is due to the implicit conversion from a SILValue to its
internal ValueBase. In comparison the new API makes it absolutely clear that one
is casting the underlying def of the SILValue:

    SILValue V;
    
    if (def_isa<ApplyInst>(V)) { ... }
    if (auto *PAI = def_dyn_cast<PartialApplyInst>(V)) { ... }

Thus the convenience of the old API is maintained, clarity is improved, and the
conceptual API boundary is enforced.

The final change that we make is that we eliminate 
2. Make the constructor `SILValue(ValueDef *)` private and via declaring
   `ValueDef` subclasses as friend classes, introduce a new API
   `ValueDef::getValue()` that constructs a `SILValue` from a specific
   `ValueDef`. This makes conceptual sense since a `SILValue` is defined by a
   `ValueDef` implying a `ValueDef` should also vend those `SILValue`s.

## Multiple Return Values

Now that we have separated the APIs that relate `SILValue` and `ValueDef`
cleanly. (TODO Maybe say match the model?), we need to consider how to represent
multiple return values. Since a key 

introduce the `ValueBundle`. A
`ValueBundle` is conceptually a "bundle of values" that is represented by an SSA
value. Its sub-elements can only be extracted by an as yet to be implemented
`bundle_extract` instruction. In the trivial case (i.e. a unary result), we do
not represent the `ValueBundle` and `bundle_extract` explicitly. This ensures
that the current IR we print today is invariant. But in the case of a
`ValueBundle` with multiple elements, we require that:

1. The `ValueBundle` SSA value is only used by `bundle_extract` instructions.
2. Each "sub-element" of the `ValueBundle` is taken by a `bundle_extract`
instruction exactly once along all paths through the program.

This enables `ValueBundles` to be used with `bundle_extract`s to represent
multiple return values.

Given that a `ValueBundle` is what a `ValueDef` defines, we make `ValueBundle` a
field on `ValueDef`:

    class ValueDef {
      ...
      ValueBundle Bundle;
    
    public:
      ValueBundle *getValues() const;
      ...
    }

Given that `ValueBundle` is now on `ValueDef` and it contains the bundle of
values defined by `ValueDef`, we remove the `SILType` and use-list field in
`ValueDef` and move those into `ValueBundle`. Thus we define the `ValueBundle`
API as follows:

    class ValueBundleImpl {
      Operand *opBegin;
      Operand *opEnd;

    public:
      ArrayRef<SILType> getTypes() const {
        return {getTypeStart(), getNumElements()};
      }

      using UseListTy =
        ArrayRefView<std::function<iterator_range<use_iterator> (Operand *)>>;
      UseListTy getUseLists() const {
        return make_arrayrefview(ArrayRef<Operand>(opBegin, opEnd),
            [](Operand *Op) -> iterator_range<use_iterator> {
              return {use_iterator(Op), use_iterator(nullptr)};
            });
        return {opBegin, getNumElements()};
      }

      unsigned getNumElements() const { return opEnd - opBegin; }

      TrivialValueBundle *getAsTrivial() const {
        if (getNumElements() != 1)
          return nullptr;
        return reinterpret_cast<TrivialValueBundle *>(this);
      }

      /// Return a range that acts as a flatmap of other ranges.
      flatmap_range<UseListTy> getAllUses() const {
        return make_flatmap_range(getUseLists());
      }

    private:
      SILType *getTypeStart() const {
          return reinterpret_cast<SILType *>(this + 1);
      }
    };

    template <unsigned NumElts>
    class ValueBundle : public ValueBundleImpl,
        protected llvm::TrailingObjects<SILType, Operand *> {
    public:
        ValueBundle() : NumElements(NumElts) {}
    };

    class TrivialValueBundle : ValueBundle<1> {
      SILType getType() const {
          return getTypeStart()[0];
      }

      use_iterator use_begin() const {
        return use_iterator(opBegin);
      }
      
      use_iterator use_end() const {
        return use_iterator(nullptr);
      }
      
      iterator_range<use_iterator> getUses() const {
          return {use_begin(), use_end()};
      }
    };

A few things to notice:

1. We explicitly differentiate in the class hierarchy in between the case of
having a TrivialValueBundle and a NonTrivialValueBundle. We make it easy to go
from ValueBundle to TrivialValueBundle by using the familiar getAs pattern to
make it easy to test. Thus when before one would perform:

    for (auto *Use : V->getUses()) {
      ...
    }

Instead one performs:

    if (auto *TrivialVB = V->getValues()->getAsTrivial()) {
      for (auto *Use : TrivialVB->getUses()) {
        ...
      }
    }

If one does not care about whether or not one has a trivial value bundle, there
is a convenience API for retrieving all uses from all operand lists:

    for (auto *Use : V->getAllUses()) {
      ...
    }

Now our SIL hierarchy looks as follows:

    INSERT GRAPHIC HERE

The final change that needs to be made is to separate the notion of Def from
SILInstruction. We do this by changing SILInstruction to no longer inherit from
`ValueDef`. Thus `SILInstruction` in of itself . We then introduce a subclass of `SILInstruction` called
`DefSILInstruction` that is defined as:

    class DefSILInstruction : public SILInstruction, ValueDef {
      
    };

Every ValueBundle is at least trivial, so we always have one FirstTy,
FirstUseList stored inline. The rest of the types/uselists are allocated in a
trailing objects array.

1. Clarifying that a `ValueBase` is not a value (and thus can not have
   ownership) and renaming `ValueBase` to `ValueDef`. For the rest of the
   document we use `ValueDef` instead of `ValueBase`.
2. Introducing the `ValueBundle` as the group of values that are defined by a
   `ValueDef`. A `ValueBundle` is just a group of values and is not a value
   itself.
3. Explicitly defining a `SILValue` as a sub-element of a `ValueBundle` with an
   ownership kind.
4. Introducing a new subclass 


# ValueOwnershipKind

Define `ValueOwnershipKind` as the enum with the following cases:

    enum class ValueOwnershipKind {
      Trivial,
      Unowned,
      Owned,
      Guaranteed,
    }

Our approach to mapping a `SILValue` to a `ValueOwnershipKind` is to use a
`SILVisitor` called `ValueOwnershipKindVisitor`. This works well since if one
holds `ValueKind` constant, `SILValue` have well defined ownership
constraints. Thus we can handle each case individually via the visitor. We use
SILNodes.def to ensure that all `ValueKind` have a defined visitor method. This
will ensure that when a new `ValueKind` is added, the compiler will emit a
warning that the visitor must be updated, ensuring correctness.

The visitor will be hidden in a *.cpp file and will expose its output via a new
API on `SILValue` :

    ValueOwnershipKind SILValue::getOwnershipKind() const;

Since the implementation of `SILValue::getOwnershipKind()` will be out of line,
none of the visitor code will be exposed to the rest of the compiler.

In order to determine if a `SILInstruction`'s operands are SILValue that have
compatible ownership with a `SILInstruction`, we introduce a new API on
`SILInstruction` that returns the ownership constraint of the `i`th operand of
the `SILInstruction`:

    Optional<ValueOwnershipKind> SILInstruction::getOwnershipConstraint(unsigned i) const;

Specifically, it returns `.Some(OwnershipKind)` if there is a constraint or
`.None` if the `SILInstruction` will accept any ValueOwnershipKind
(e.g. `copy_value`). This API is sufficient since there are no `SILInstructions`
that accept only a subset of `ValueOwnershipKind`, all either accept a singular
ownership kind or all ownership kinds.

# Verification of Ownership Semantics

Since our ownership model is based around SSA form, all verification of
ownership only need consider an individual value (`SILValue`) and the uses of the
def (`SILInstruction`). Thus for each def/use-set, we:

1. **Verify Use Semantics**: All uses must be compatible with the def's
   `ValueOwnershipKind`.
2. **Dataflow Verification**: Given any path P in the program and a `ValueDef`
   `V` along that path:
   a. There exists only one <a href="#footnote-0-lifetime-ending-use">"lifetime ending"</a> use of `V` along that path.
   b. After the lifetime ending use of `V`, there are no non-lifetime ending
      uses of `V` along `P`.

Since the dataflow verification requires knowledge of a subset of the uses, we
perform the use semantic verification first and then use the found uses for the
dataflow verification.

## Use Verification

Just like with `ValueDef`, individual `SILInstruction` kind's have well-defined
ownership semantics implying that we can use a `SILVisitor` approach here as
well. Thus we define a `SILVisitor` called
`OwnershipCompatibilityUseChecker`. This checker works by taking in a
`ValueDef` and visiting all of the `SILInstruction` users of the
ValueDef. Each visitor method returns a pair of booleans, the first stating
whether or not the ownership values were compatible and the second stating
whether or not this specific use should be considered a "lifetime ending"
use. The checker then stores the lifetime ending uses and non-lifetime ending
uses in two separate arrays for processing by the dataflow verifier.

## Dataflow Verification

The dataflow verifier takes in as inputs the `ValueDef` (i.e. def) and lists of
lifetime-ending and non-lifetime ending uses. Since we are using SSA form, we
already know that our def must dominate all of our uses implying that a use can
never overconsume due to a def not being along a path. On the other hand, we
still must consider issues related to joint post-dominance namely:

1. Double-Consume due to 2+ lifetime ending uses along a path. This implies
   proving that no lifetime ending use is reachable from another lifetime ending
   use.
2. Use-After-Free due to a non-lifetime ending use not being joint-post
   dominated by the lifetime ending uses. This implies proving that all
   non-lifetime ending uses are joint post-dominated by the lifetime ending use
   set.
3. Lifetime-Leaks due to the non-lifetime ending uses not joint-post dominating
   the lifetime ending uses.

Note how we must consider joint post-dominance in two different places. Thus we
must be able to prove joint post-dominance quickly and cheaply. To do so, we
take advantage of dominance implying that all uses are reachable from the def to
just needing to prove that when we walk the CFG up to prove reachability, that
any block on the reachability path does not have any successors that have not
been visited when we finish walking and visit the def.

Define the following book keeping data structures.

    // The worklist that we will use for our iterative reachability query.
    llvm::SmallVector<SILBasicBlock *, 32> Worklist;
 
    // The set of blocks with lifetime ending uses.
    llvm::SmallPtrSet<SILBasicBlock *, 8> BlocksWithLifetimeEndingUses;
 
    // The set of blocks with non-lifetime ending uses and the associated
    // non-lifetime ending use SILInstruction.
    llvm::SmallDenseMap<SILBasicBlock *, SILInstruction *, 8> BlocksWithNonLifetimeEndingUses;

    // The blocks that we have already visited.
    llvm::SmallPtrSet<SILBasicBlock *, 32> VisitedBlocks;
 
    // A list of successor blocks that we must visit by the time the algorithm terminates.
    llvm::SmallPtrSet<SILBasicBlock *, 8> SuccessorBlocksThatMustBeVisited;

Then for each non-lifetime ending use, we add the block and its instruction to
the `BlocksWithNonLifetimeEndingUses`. There is a possibility of having multiple
non-lifetime ending uses in the same block, in such a case, we always take the
later value since we are performing a liveness-esque dataflow:

    for (SILInstruction *User : getNonLifetimeEndingUses()) {
      // First try to associate User with User->getParent().
      auto Result = BlocksWithNonLifetimeEndingUses.insert({User->getParent(), User});

      // If the insertion succeeds, then we know that there is no more work to
      // be done, so process the next use.
      if (Result.second)
        continue;

      // If the insertion fails, then we have at least two non-lifetime ending uses
      // in the same block. Since we are performing a liveness type of dataflow,
      // we only need the last non-lifetime ending use to show that all lifetime
      // ending uses post dominate both. Thus, see if Use is after
      // Result.first->second in the use list. If Use is not later, then we wish
      // to keep the already mapped value, not use, so continue.
      if (std::find(Result.first->second->getIterator(), Block->end(), User) ==
            Block->end()) {
        continue;
      }

      // At this point, we know that Use is later in the Block than
      // Result.first->second, so store Use instead.
      Result.first->second = User;
    }

Then we visit each each lifetime ending use and its parent block:

    for (SILInstruction *User : getLifetimeEndingUses()) {
      SILBasicBlock *UserParentBlock = User->getParent();
      ...
    }

We begin by adding `UserParentBlock` to the `BlocksWithLifetimeEndingUses`
set. If a block is already in the set (i.e. insert fails), then we know that (1)
has been violated and we error. If we never hit that condition, then we know
that all lifetime ending uses are in different blocks.

    if (!BlocksWithLifetimeEndingUses.insert(UserParentBlock).second) {
      double_consume_error(); // ERROR!
    }

Then we check if we previously saw any non-lifetime ending uses in
`UserParentBlock` by checking the map `BlocksWithNonLifetimeEndingUses`. If we do
find any such uses, we check if the lifetime ending use is earlier in the block
that the non-lifetime ending use. If so then (2) is violated and we
error. Once we know that (2) has not been violated, we remove 

    auto Iter = BlocksWithNonLifetimeEndUses.find(UserParentBlock);
    if (Iter != BlocksWithNonLifetimeEndUses.end()) {
      // Make sure that the non-lifetime ending use is before the lifetime
      // ending use. Otherwise, we have a use after free.
      if (std::find(User->getIterator(), UserParentBlock->end(), Iter->second) !=
            UserParentBlock->end()) {
        use_after_free_error(); // ERROR!
      }

      // Erase the use since we know that it is properly joint post-dominated.
      BlocksWithNonLifetimeEndingUses.erase(Iter);
    }

Then we mark `UserParentBlock` as visited to ensure that we do not consider
UserParentBlock to be a successor we must visit and prime the worklist with the
predecessor blocks of `UserParentBlock`.

    // Then mark UserParentBlock as visited add all predecessors of
    // UserParentBlock to the worklist.
    VisitedBlocks.insert(UserParentBlock);
    copy(UserParentBlock->getPreds(), std::back_inserter(Worklist));

The entire block of code is provided in the appendix. Now we know that:

1. All lifetime ending uses are in different basic blocks.
2. All non-lifetime ending uses are either in different basic blocks.

We begin our worklist algorithm by popping off the next element in the worklist.

    // Until the worklist is empty.
    while (!Worklist.empty()) {
      // Grab the next block to visit.
      const SILBasicBlock *BB = Worklist.pop_back_val();
      ...
    }

We only visit blocks at most once, so we first check if this is a block that we
have already visited. If we have, continue there is no further work to do.

    if (!VisitedBlocks.insert(BB).second) {
      continue;
    }

Then make sure that BB is not in `BlocksWithLifetimeEndingUses`. If BB is in
`BlocksWithLifetimeEndingUses`, then we know that there is a path going through
the def where the def is consumed twice, an error!

    if (BlocksWithLifetimeEndingUses.count(BB)) {
      double_consume_error(); // ERROR!
    }

Now that we know we are not double consuming, we need to update our data
structures. First if BB is contained in SuccessorBlocksThatMustBeVisited, we
remove it. This ensures when the algorithm terminates, we know that the path
through the successor was visited as apart of our walk.

    SuccessorBlocksThatMustBeVisited.erase(BB);

Then if BB is in BlocksWithNonLifetimeEndingUses, we remove it. This ensures
that when the algorithm terminates, we know that the non-lifetime ending uses
were properly joint post-dominated by the lifetime ending uses.

    BlocksWithNonLifetimeEndingUses.erase(BB);

Then add all successors of BB that we have not visited yet to the
`SuccessorBlocksThatMustBeVisited` set. This ensures that if we do not visit the
successor as apart of our CFG walk, at the end of the algorithm we will assert
that there is a leak.

    for (const SILBasicBlock *SuccBlock : BB->getSuccessorBlocks()) {
      // If we already visited the successor, there is nothing to do since
      // we already visited the successor.
      if (VisitedBlocks.count(SuccBlock))
        continue;
          
      // Otherwise, add the successor to our MustVisitBlocks set to ensure that
      // we assert if we do not visit it by the end of the algorithm.
      SuccessorBlocksThatMustBeVisited.insert(SuccBlock);
    }

Finally add all predecessors of BB that we have not visited yet to the Worklist
for processing:

    for (const SILBasicBlock *PredBlock : BB->getPredecessorBlocks()) {
      if (VisitedBlocks.count(PredBlock))
        continue;
      Worklist.push_back(PredBlock);
    }

This continues until we have exhausted the worklist. Once the worklist is
exhausted, we know that:

1. If `SuccessorBlocksThatMustBeVisited` is non-empty, then the Blocks in the
   set are not joint post-dominated by the set of lifetime ending users implying
   a leak.
2. If `BlockSwithNonLifetimeEndingUses` is non-empty, then there was a
   non-lifetime ending use that was not joint post-dominated by the lifetime
   ending use set. This implies a use-after free.

Thus we assert that both sets are empty and error accordingly.

   if (!SuccessorBlocksThatMustBeVisited.empty()) {
     leak_error(); // ERROR!
   }
   if (!BlocksWithNonLifetimeEndingUses.empty()) {
     use_after_free_error(); // ERROR!
   }

The full code is in the appendix.

<a id="footnote-0-lifetime-ending-use">[0]</a>: A use after which a def is no
longer allowed to be used in any way, e.g. `destroy_value`, `end_borrow`.

# Appendix

## Full Code For Initialization of Worklist

    for (SILInstruction *User : getLifetimeEndingUses()) {
      SILBasicBlock *UserParentBlock = User->getParent();
      if (!BlocksWithLifetimeEndingUses.insert(UserParentBlock).second) {
        double_consume_error(); // ERROR!
      }

      auto Iter = BlocksWithNonLifetimeEndUses.find(UserParentBlock);
      if (Iter != BlocksWithNonLifetimeEndUses.end()) {
        // TODO: Could this be cached.
        if (std::find(User->getIterator(), UserParentBlock->end(), Iter->second) !=
              UserParentBlock->end()) {
          use_after_free_error(); // ERROR!
        }
      }

      // Then add all predecessors of the user block to the worklist.
      VisitedBlocks.insert(UserParentBlock);
      copy(UserParentBlock->getPreds(), std::back_inserter(Worklist));
    }

## Full Code For Worklist Algorithm

    // Until the worklist is empty.
    while (!Worklist.empty()) {
      // Grab the next block to visit.
      const SILBasicBlock *BB = Worklist.pop_back_val();

      // If we already visited the block, then there is no further work to do.
      if (!VisitedBlocks.insert(BB).second) {
        continue;
      }

      // Then check that BB is not a lifetime-ending use block. If it is error,
      // since we have an overconsume.
      if (BlocksWithLifetimeEndingUses.count(BB)) {
        double_consume_error(); // ERROR!
      }

      // Ok, now we know that we are not double-consuming. Update our data
      // structures.

      // First remove BB from the SuccessorBlocksThatMustBeVisited list. This
      // ensures that when the algorithm terminates, we know that BB was not the
      // beginning of a non-covered path to the exit.
      SuccessorBlocksThatMustBeVisited.erase(BB);

      // Then remove BB from BlocksWithNonLifetimeEndingUses so we know that
      // this block was properly joint post-dominated by our lifetime ending
      // users.
      BlocksWithNonLifetimeEndingUses.erase(BB);

      // Then add all successors of BB that we have not visited yet to the
      // SuccessorBlocksThatMustBeVisited set.
      for (const SILBasicBlock *SuccBlock : BB->getSuccessorBlocks()) {
        // If we already visited the successor, there is nothing to do since
        // we already visited the successor.
        if (VisitedBlocks.count(SuccBlock))
          continue;
            
        // Otherwise, add the successor to our MustVisitBlocks set to ensure that
        // we assert if we do not visit it by the end of the algorithm.
        SuccessorBlocksThatMustBeVisited.insert(SuccBlock);
      }

      // Finally add all predecessors of BB that have not been visited yet to
      // the worklist.
      for (const SILBasicBlock *PredBlock : BB->getPredecessorBlocks()) {
        if (VisitedBlocks.count(PredBlock))
          continue;
        Worklist.push_back(PredBlock);
      }
    }

    if (!SuccessorBlocksThatMustBeVisited.empty()) {
      leak_error(); // ERROR!
    }
    if (!BlocksWithNonLifetimeEndingUses.empty()) {
      use_after_free_error(); // ERROR!
    }

<!--

# Implementation

## `ValueDef::getOwnershipKind()`

Now that we have defined intersection, we categorize all ValueDef into sets
depending on the ValueDef's result ownership properties (if a result
exists). These categories are:

1. **No Result**. A ValueDef without a result.
2. **Constant Ownership**. A ValueDef with a result that always produces the same
   ownership kind.
3. **Forwarding Ownership**. A ValueDef that is an Instruction whose result is
   always equivalent to the same ownership kind as one of the instruction's operands.
4. **Special Ownership**. An instruction with special rules for propagating
   ownership. This includes ValueDef such as ApplyInst, SILArgument, and
   TryApply.

Using these categories, we then define:

`ValueOwnershipKind ValueDef::getOwnershipKind() const`

This is done by 
Using these categories, we implement a method on `ValueDef` called
`ValueDef::getOwnershipKind()`. This will be implemented using a visitor to
ensure that warnings are provided when engineers add new `ValueDef` so that
keeping the ownership code up to date will be easy.

     /// The ownership semantics of a def-use edge.
     enum class ValueOwnershipKind {

       /// Represents unaddited ownership.
       ///
       /// Represents the ownership of a value that has not been audited or is
       /// actually undefined. A def that produces a value with unknown
       /// ownership can not be paired with any use that does not have Invalid
       /// or Any ownership.
       Undefined,

       /// Represents the ownership of a trivially typed SSA value. Can only be
       /// paired with uses with Any or Trivial ownership.
       Trivial,

       /// Represents the ownership of a non-trivial value that must be
       /// immediately retained before use. This is used generally in
       /// objective-c conventions.
       Unowned,

       /// Represents the ownership of a non-trivial value that is being passed
       /// at +1.
       Owned,

       /// Represents the ownership of an immutably borrowed value being passed
       /// at +0.
       Guaranteed,

       /// Represents the ownership of a mutably borrowed value being passed at
       /// +0.
       InOut,

       /// Top. Represents any ownership.
       Any,
     };


1. Defining an enum called `ValueOwnershipKind` that specifies possible
ownership along a def-use edge.
2. Implementing the API `ValueOwnershipKind ValueDef::getOwnershipKind() const`
to vend these values.
3. Implementing the API `void SILInstruction::verifyOperandOwnership() const`
that verifies that a `SILInstruction`'s operands have ownership that is
compatible with the `SILInstruction`'s ownership.

These 3 points will enable for all def-use edges in SIL to be statically
verified as obeying ownership semantics.

Define `ValueOwnershipKind` as follows:




# Mapping ValueDef to ValueOwnershipKind

# Proving Def-Use Convention Correctness

# Identifying Ownership Dataflow Errors
-->
