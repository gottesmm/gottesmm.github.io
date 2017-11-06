---
layout: page
title: SIL Ownership Model
categories: proposals old
---

## {{ page.title }}

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [Summary](##summary)
- [Cleanly separating Value and Def APIs in SIL](##cleanly-separating-value-and-def-apis-in-sil)
- [Values and ValueOwnershipKinds](##values-and-valueownershipkinds)
- [ValueDefs and Ownership Constraints](##valuedefs-and-ownership-constraints)
- [Verification of Ownership Semantics](##verification-of-ownership-semantics)
    - [Use Verification](##use-verification)
    - [Dataflow Verification](##dataflow-verification)
- [Appendix](##appendix)
    - [Changes to SILValue API for SIL Ownership](##changes-to-silvalue-api-for-sil-ownership)
    - [Full Dataflow Verification Algorithm](##full-dataflow-verification-algorithm)

<!-- markdown-toc end -->

## Summary

This document defines a SIL ownership model and a compile time static verifier
for the model. This will allow for static compile time verification that a SIL
program satisfies all ownership model constraints.

The proposed SIL ownership model embeds ownership into SIL's SSA def-use
edges. This is accomplished by:

1. Formalizing into SIL API's the distinction in between defs
   (e.g. `SILInstruction` and `SILArgument`) and the values produced by defs
   (e.g. `SILValue`). This is needed to model values as having ownership and
   defs as not having ownership. The main implication here is that the two can
   no longer be implicitly convertible and the API must enforce this.
2. Specifying a set of ownership kinds and specifying a method for mapping a
   `SILValue` to an ownership kind.
3. Specifying ownership constraints on all `SILInstruction`s and `SILArgument`s
   that constrain what ownership kinds their operands and incoming values
   respectively can possess.
4. Implementing a verifier to ensure that all `SILInstruction` and `SILArgument`
   are compatible with the ownership kind propagated by their operand
   `SILValue`s and that pseudo-linear dataflow constraints are maintained.

## Cleanly separating Value and Def APIs in SIL

All values in SIL are defined via an assignment statement of the form: `<foo> = <bar>`.
In English, we say `foo` is a value that is defined by the def
`bar`. Originally, these two concepts were distinctly represented by the classes
`SILValue` and `ValueBase`. All `ValueBase` defined a list of `SILValue` that
were related, but not equivalent to the defining `ValueBase`. With the decision
to represent multiple return values as instruction projections instead of as a
list of `SILValue`, this distinction in between a def and the values was lost,
resulting in `SILValue` being used interchangeably with `ValueBase` throughout
the swift codebase. This exposes a modeling issue when one attempts to add
ownership to SIL, namely that values have ownership, while the defs that define
the values do not. This implies that defs and values *should not* be
interchangeable.

In order to model that values, not defs, have ownership, we separate the
`SILValue` and `ValueBase` APIs. This is done by:

1. Renaming `ValueBase` to `ValueDef`. This makes it clear from a naming
perspective that a `ValueDef` is not a value, but may define values.
2. Eliminate all operator methods on `SILValue` that allow one to work with the
`ValueDef` API via a `SILValue` in favor of an explicit method for getting the
internal `ValueDef` of a `SILValue`, i.e.:

        class SILValue {
          ...
          ValueDef *operator->() const; // deleted
          ValueDef &operator*() const; // deleted
          operator ValueDef *() const; // deleted

          bool operator==(ValueDef *RHS) const; // deleted
          bool operator!=(ValueDef *RHS) const; // deleted
          ...
          ValueDef *SILValue::getDef() const; // new
          ...
        };

3. Use private access control and friend classes to only allow for `SILValue`s
   to be constructed and vended by their defining `ValueDef` via a new method on
   `ValueDef`:

        class SILValue {
          friend class SILUndef; // new
          friend class SILArgument; // new
          friend class SILInstruction; // new
        public:
          SILValue(ValueDef *); // deleted
        private:
          SILValue(ValueDef *); // new
        };

        class ValueDef {
          ...
          SILValue getValue() const; // new
          ...
        };

To see how specific common code patterns change in light of these changes,
please see the [appendix](##changes-to-silvalue-api-for-sil-ownership).

## Values and ValueOwnershipKinds

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

## ValueDefs and Ownership Constraints

In order to determine if a `SILInstruction`'s operands are SILValue that have
compatible ownership with a `SILInstruction`, we introduce a new API on
`SILInstruction` that returns the ownership constraint of the `i`th operand of
the `SILInstruction`:

    Optional<ValueOwnershipKind> SILInstruction::getOwnershipConstraint(unsigned i) const;

Specifically, it returns `.Some(OwnershipKind)` if there is a constraint or
`.None` if the `SILInstruction` will accept any `ValueOwnershipKind`
(e.g. `copy_value`). This API is sufficient since there are no `SILInstructions`
that accept only a subset of `ValueOwnershipKind`, all either accept a singular
ownership kind or all ownership kinds.

For `SILArgument`, we determine compatible ownership by introducing the concept
of ownership conventions. The reason to introduce these ownership conventions is
that it ensures that we do not need to perform any form of dataflow to determine
the convention of a `SILArgument` in a loop. Additionally, it allows for greater
readability in the IR and prevents the optimizer from performing implicit
conversions of ownership semantics on branch instructions, for instance,
converting a `switch_enum` from consuming a value at +1 to using a borrowed +0
parameter. In terms of API, we introduce a new method on `SILArgument`:

    ValueOwnershipKind SILArgument::getOwnershipConstraint() const;

which returns the required ownership constraint. In terms of representing these
ownership constraint conventions in textual SIL, we print out the ownership
constraint next to the specific argument in the block and the specific argument
in the given terminator. For details of how this will look with various
terminators, see the [appendix](##sil-argument-terminator-convention-examples).

## Verification of Ownership Semantics

Since our ownership model is based around SSA form, all verification of
ownership only need consider an individual value (`SILValue`) and the uses of
the def (`SILInstruction`, `SILArgument`). Thus for each def/use-set, we:

1. **Verify Use Semantics**: All uses must be compatible with the def's
   `ValueOwnershipKind`.
2. **Dataflow Verification**: Given any path P in the program and a `ValueDef`
   `V` along that path:
   a. There exists only one <a href="##footnote-0-lifetime-ending-use">"lifetime ending"</a> use of `V` along that path.
   b. After the lifetime ending use of `V`, there are no non-lifetime ending
      uses of `V` along `P`.

Since the dataflow verification requires knowledge of a subset of the uses, we
perform the use semantic verification first and then use the found uses for the
dataflow verification.

### Use Verification

Just like with `SILValue`, individual `SILInstruction` kind's have well-defined
ownership semantics implying that we can use a `SILVisitor` approach here as
well. Thus we define a `SILVisitor` called
`OwnershipCompatibilityUseChecker`. This checker works by taking in a `SILValue`
and visiting all of the `SILInstruction` users of the `SILValue`. Each visitor
method returns a pair of booleans, the first stating whether or not the
ownership values were compatible and the second stating whether or not this
specific use should be considered a "lifetime ending" use. The checker then
stores the lifetime ending uses and non-lifetime ending uses in two separate
arrays for processing by the dataflow verifier.

### Dataflow Verification

The dataflow verifier takes in as inputs the `SILValue` and lists of
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

The full code is in the [appendix](##full-dataflow-verification-algorithm).

<a id="footnote-0-lifetime-ending-use">[0]</a>: A use after which a def is no
longer allowed to be used in any way, e.g. `destroy_value`, `end_borrow`.

## Appendix

### Changes to SILValue API for SIL Ownership

The common code pattern eliminated by removing implicit uses of `ValueDef`'s API
via a `SILValue` are as follows:

    SILValue V;
    ValueDef *Def;

    if (V != Def) { ... }
    if (V->getParentBlock()) { ... }

Our change, causes these code patterns to be rewritten in favor of the explicit:

    SILValue V;
    ValueDef *Def;

    if (V.getDef() != Def) { ... }
    if (V.getDef()->getParentBlock()) { ... }

In cases like the above, we want the to increase clarity through the usage of
verbosity. In other cases, this verboseness makes convenient APIs more difficult
to use. The main example here are the `isa` and `dyn_cast` APIs. We introduce
two helper functions that give the same convenience as before but added clarity by
making it clear that we are not performing an isa query on the value, but
instead the underlying def of the value:

    template <typename ParentTy>
    bool def_isa(SILValue V) { return isa<ParentTy>(V.getDef()); }
    template <typename ParentTy>
    ParentTy *def_dyn_cast(SILValue V) { return dyn_cast<ParentTy>(V.getDef()); }

Consider the following code using the old API,

    SILValue V;

    if (isa<ApplyInst>(V)) { ... }
    if (auto *PAI = dyn_cast<PartialApplyInst>(V)) { ... }

Notice how it seems like one is casting the SILValue as if it is a
ValueBase. This is due to the misleading usage of the implicit conversion from a
`SILValue` to the `SILValue`'s internal `ValueBase`. In comparison the new API
makes it absolutely clear that one is reasoning about the ValueKind of the
underlying def of the `SILValue`:

    SILValue V;

    if (def_isa<ApplyInst>(V)) { ... }
    if (auto *PAI = def_dyn_cast<PartialApplyInst>(V)) { ... }

Thus the convenience of the old API is maintained, clarity is improved, and the
conceptual API boundary is enforced.

The final change that we propose is eliminating the ability to construct a
`SILValue` from a `ValueDef` externally to the `ValueDef` itself. Allowing this
to occur violates the modeling notion that a `SILValue` is defined and thus is
dependent on the `ValueDef`. To implement this, we propose changing the
constructor `SILValue::SILValue(ValueDef *)` to have private instead of public
access control and declaring `ValueDef` subclasses as friends of
`SILValue`. This then allows the `ValueDef` to vend opaquely constructed
`SILValue`, but disallows external users of the API from directly creating
`SILValue` from `ValueDef`, enforcing the value/def distinction in our model.

### SILArgument/Terminator OwnershipConvention Examples

We present here several examples of how ownership conventions look on
SILArguments and terminators.

First we present a simple branch and return example.

    class C { ... }

    sil @simple_branching : $@convention(thin) : @convention(thin) (@owned Builtin.NativeObject, @guaranteed C) -> @owned C {
    bb0(%0 : @owned $Builtin.NativeObject, %1 : @guaranteed $C):
      br bb1(%0 : @owned $Builtin.NativeObject)

    bb1(%1 : @owned $Builtin.NativeObject):
      strong_release %1 : $Builtin.NativeObject
      %2 = copy_value %0 : $C
      return %2 : @owned $C
    }

Now consider this loop example:

    sil @loop : $@convention(thin) : $@convention(thin) (@owned Optional<Builtin.NativeObject>) -> () {
    bb0(%0 : @owned $Optional<Builtin.NativeObject>):
      br bb1(%0 : @owned $Builtin.NativeObject)

    bb1(%1 : @owned $Builtin.NativeObject):
      %2 = alloc_object $C
      strong_release %1 : $Builtin.NativeObject
      %3 = unchecked_ref_cast %2 : $C to $Builtin.NativeObject
      cond_br %cond, bb1(%3 : @owned Builtin.NativeObject), bb2

    bb2:
      strong_release %3 : $Builtin.NativeObject
      %result = tuple()
      return %result : @trivial $()
    }

Note how it is absolutely clear what convention is being used when passing a
value to the phi node. No dataflow reasoning is required implying we can do a
simple pass over the CFG to prove correctness.

Now consider two examples of switches:

    sil @owned_switch_enum : $@convention(thin) : $@convention(thin) (@owned Optional<Builtin.NativeObject>) -> () {
    bb0(%0 : @owned $Optional<Builtin.NativeObject>):
      switch_enum %0 : @owned $Builtin.NativeObject, ##Optional.none.enumelt: bb1, #Optional.some.enumelt.1: bb2

    bb1:
      br bb3

    bb2(%1 : @owned $Builtin.NativeObject):
      strong_release %1 : $Builtin.NativeObject
      br bb3

    bb3:
      %result = tuple()
      return %result : @trivial $()
    }

    sil @guaranted_converted_switch_enum : $@convention(thin) : $@convention(thin) (@owned Optional<Builtin.NativeObject>) -> () {
    bb0(%0 : @owned $Optional<Builtin.NativeObject>):
      %1 = begin_borrow %0 : $Optional<Builtin.NativeObject>
      switch_enum %1 : @guaranteed $Builtin.NativeObject, ##Optional.none.enumelt: bb1, #Optional.some.enumelt.1: bb2

    bb1:
      br bb3

    bb2(%2 : @guaranteed $Builtin.NativeObject):
      %3 = function_ref @g_user : $@convention(thin) (@guaranteed Builtin.NativeObject) -> ()
      apply %3(%2) : $@convention(thin) (@guaranteed Builtin.NativeObject) -> ()
      br bb3

    bb3:
      end_borrow %1 from %0 : $Optional<Builtin.NativeObject>, $Optional<Builtin.NativeObject>
      %result = tuple()
      return %result : @trivial $()
    }

Notice how the lifetime is completely explicit in both cases, so the optimizer
can not treat the conversion of switch_enum from +1 to +0 implicitly.

### Full Dataflow Verification Algorithm

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

Then for each non-lifetime ending use (found by the
`OwnershipCompatibilityUseChecker`), we add the block and its instruction to the
`BlocksWithNonLifetimeEndingUses`. There is a possibility of having multiple
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

Since every time we insert objects into the worklist, we first try to insert it
into the `VisitedBlock` list, we do not need to check if `BB` has been visited
yet, since elements can not be added to the worklist twice.

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
for processing. We use insert here to ensure that we only ever add a
VisitedBlock once to the Worklist:

    for (const SILBasicBlock *PredBlock : BB->getPredecessorBlocks()) {
      if (!VisitedBlocks.insert(PredBlock).second)
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

The full code:

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

    // Until the worklist is empty.
    while (!Worklist.empty()) {
      // Grab the next block to visit.
      const SILBasicBlock *BB = Worklist.pop_back_val();

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
        // Try to insert the PredBlock into the VisitedBlocks list to ensure
        // that we only ever add a block once to the worklist.
        if (!VisitedBlocks.insert(PredBlock).second)
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
