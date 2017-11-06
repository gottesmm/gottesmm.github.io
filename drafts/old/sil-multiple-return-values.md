---
layout: page
title: SIL Multiple Return Values
categories: proposals old
---

## Introduction

This document is a proposal to re-add multiple return values to SIL. The reason
why this is being proposed is to enable for a destructure operation to be
implemented in SIL. The structure of this document is as follows:

1. Background and Justification.
2. Design

## Background and Justification

SIL originally had multiple return values. These were removed in the past year
since the APIs that were used to manipulate return values were difficult to use
and a small compile time benefit was realized. The difficulty of these APIs
being yielded were rooted in the APIs for manipulating single value instructions
and multiple value instructions were not separated at the API level. This led to
groups of bugs where a compiler writer was trying to only handle single result
instructions but did not realize that he/she needed to also handle multiple
value instructions. These bugs resulted in various proposals for removing the
ability to make this mistake by changing the relevant APIs. At the time there
was no program in Swift that could not be expressed efficienctly in terms of a
SIL program that only used single return values. Thus multiple return values
were removed. Semantic SIL has changed this computation.

Today, SIL is being changed to 

We currently have two kinds of def: instructions and arguments.  Arguments
always define a single value, and I don't see anything in your proposal that
changes that.  And the idea that an instruction produces exactly one value is
already problematic, because many don't produce a meaningful value at all.
All that changes in your proposal is that certain instructions need to be able
to produce multiple values.

Moreover, the word "def" clearly suggests that it refers to the definition of a
value that can be used, and that's how the term is employed basically everywhere.

So allow me to suggest that a much clearer way of saying what you're trying to
say is that we need to distinguish between defs and instructions.  An instruction
may have an arbitrary number of defs, possibly zero, and each def is a value
that can be used.  (But the number of defs per instruction is known statically
for most instructions, which is something we can use to make working with
defs much less annoying.)

Also this stuff you're saying about values having ownership and defs not having
ownership is, let's say, misleading; it only works if you're using a meaninglessly
broad definition of ownership.  It would be better to simply say that the verification
we want to do for ownership strongly encourages us to allow instructions to 
introduce multiple defs whose properties can be verified independently.

2. Specifying a set of ownership kinds and specifying a method for mapping a
   `SILValue` to an ownership kind.
3. Specifying ownership constraints on all `SILInstruction`s and `SILArgument`s
   that constrain what ownership kinds their operands and incoming values
   respectively can possess.
4. Implementing a verifier to ensure that all `SILInstruction` and `SILArgument`
   are compatible with the ownership kind propagated by their operand
   `SILValue`s and that pseudo-linear dataflow constraints are maintained.

I'll tackle these other sections in another email.  Let's go one at a time.

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

Again, I do not understand what you're trying to say about ownership here.
Just drop it.

In order to model that values, not defs, have ownership, we separate the
`SILValue` and `ValueBase` APIs. This is done by:

Almost all of this is based on what I consider to be a really unfortunate use
of the term "def".

Allow me to suggest this:

1. You do not need to rename ValueBase.  Let's stick with the term "Value"
instead of "Def" in the source code.

2. There are only three subclasses of ValueBase for now:
  - SILArgument
  - SingleInstructionResult
  - MultiInstructionResult
This means that the Kind can probably be packed into the use-list pointer.

3. SingleInstructionResult will only be used for instructions that are known to
produce exactly one value.  All such instructions can derive from a single
class, of which SingleInstructionResult can be a superclass.  This will make
it possible to construct a SILValue directly from such instructions, as well as
making it easy to support dyn_casting back to such instructions.

4. MultiInstructionResult will have to store the result index as well as provide
some way to get back to the instruction.  I think the most efficient representation
here is to store an array of MultiInstructionResults immediately *before* the
address point of the instruction, in reverse order, so that (this + ResultIndex + 1)
gets you back to the instruction.  This also makes it possible to efficiently index
into the results without loading the number of results (except for the
bounds-checking assert, of course).

It will not be possible to construct a SILValue directly from one of these instructions;
you'll have to ask the instruction for its Nth result.

It's not clear to me what dyn_casting a SILValue back to a multi-result instruction
should do.  I guess it succeeds and then you just ask the instruction which
result you had.

5. SILInstruction has an accessor to return what's basically an ArrayRef of
its instruction results.  It's not quite an ArrayRef because that would require
type-punning, but you can make it work as efficiently as one.

---

<!--

### Multiple Return Value Implementation

Now that we have separated the APIs that relate `SILValue` and `ValueDef`
cleanly. (TODO Maybe say match the model?), we need to consider how to represent
multiple return values. We do this by introducing a new class called a
`ValueBundle`.

A `ValueBundle` is conceptually a “bundle of values” that is represented by an
SSA value. Its sub-elements can only be extracted by an as yet to be implemented
`bundle_extract` instruction. In the trivial case (i.e. a unary result), we do
not represent the `ValueBundle` and `bundle_extract` explicitly. This ensures
that the current IR we print today is invariant. But in the case of a
`ValueBundle` with multiple elements, we require that:

The `ValueBundle` SSA value is only used by `bundle_extract` instructions.  Each
“sub-element” of the `ValueBundle` is taken by a `bundle_extract` instruction
exactly once along all paths through the program.  This enables `ValueBundles`
to be used with `bundle_extracts` to represent multiple return values.

Given that a `ValueBundle` is what a `ValueDef` defines, we make `ValueBundle` a
field on `ValueDef`:

    class ValueDef {
      ...
      ValueBundle Bundle;
    
    public:
      ValueBundle *getValues() const;
      ...
    }

Given that ValueBundle is now on ValueDef and it contains the bundle of values
defined by ValueDef, we remove the SILType and use-list field in ValueDef and
move those into ValueBundle. Thus we define the ValueBundle API as follows:

    class ValueBundleImpl {
      SILType *typeBegin;
      SILType *typeEnd;
    
    public:
      ArrayRef<SILType> getTypes() const {
        return {typeBegin, typeEnd};
      }
    
      using UseListTy =
        ArrayRefView<std::function<iterator_range<use_iterator> (Operand *)>>;
      UseListTy getUseLists() const {
        return make_arrayrefview(ArrayRef<Operand>(getOperandBegin(), getNumElements()),
                                 [](Operand *Op) -> iterator_range<use_iterator> {
                                   return {use_iterator(Op), use_iterator(nullptr)};
                                 });
        return {opBegin, getNumElements()};
      }
    
      unsigned getNumElements() const { return typeEnd - typeBegin; }
    
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
      Operand **getOperandBegin() const {
        return reinterpret_cast<Operand **>(this + 1);
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

Notice how we explicitly differentiate in the class hierarchy in between the
case of having a TrivialValueBundle and a NonTrivialValueBundle. We make it easy
to go from ValueBundle to TrivialValueBundle by using the familiar getAs pattern
to make it easy to test. Thus when before one would perform:

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

If we know that our ValueBase has a single result (i.e. it is a `SILArgument`,
`SILUndef`, or a `SILInstruction` with 1 result), then we can expose the trivial
value bundle APIs such as getType(), getUses(), etc.

This does require us to change SILValue. Previously when we had multiple return
values, a SILValue contained a pointer to the ValueBase and an integer
representing the index in the multiple return value array of the
ValueBase. Since we have eliminated multiple return values, SILValue’s have been
used often times in data structures that expect a pointer sized value. This
creates some representation issues for us to work around. We do this by
introducing a new data structure called EmbeddedPointerListInt. This is a class
that embeds an integer in the spare bits of a list of pointers. Assume that we 4
byte align our Operands implying our operand pointers provide us with 4 bits to
work with. We wish to store in these spare bits the operand offset from specific
leader operands to the first operand in the operand list. We first assume that a
user will never define a struct with 32768 fields. This means we need to find 15
bits of extra space. Then we use the following for a given SILValue to determine
its ValueBundle:

Define an EmbeddedPointerListInt pointer list as a group of up to 5 pointers
where we store an integer in the spare bits of the pointers. We use a variadic
integer scheme by saying that the leader pointer of the EmbeddedPointerListInt
is the only pointer that does not have its bottom tagged pointer bit set. This
enables one to iterate back through the pointers to determine the beginning of
an EmbeddedPointerListInt. Then we introduce an additional constraint where we
say that if we have a leader pointer where the 2nd to the bottom bit is set,
then we know we are at the beginning of a list of EmbeddedPointerListInts. This
enables us to determine the difference in between the first leader pointer in
the operand list and the rest of the leader pointers.

Assume that we store a pointer to the Operand *(i.e. ValueBundle use list)
associated with a given SILValue. We begin by checking the 1st bit. If none of
the lower bits are set, then we know that we have the first element of the
Operand list and know the location of the ValueBundle structure. Assume that the
1st bit is set to 1. Then we check if the second bit is set. If the second bit
is not set, then we know that we have reached the beginning of the operand list
of a NonTrivialValueBundle. If the second bit is set, then we know we have the
leader pointer of an intermediate ValueBundle. Since we do not know if we have a
full EmbeddedPointerListInt, we go back through the Operand list to the previous
EmbeddedPointerListInt. We know via contiguousness that this must be a full
EmbeddedPointerListInt, then we extract the distance to the beginning of the
Operand list and recover the location of the ValueBundle. If the 1st bit is not
set, then we know that we do not have a leader pointer in an
EmbeddedPointerListInt. Then we traverse backwards through the Operand list
until we find a leader. If the leader pointer is a beginning of Operand list
pointer, we know the address of our ValueBundle. Otherwise, if we visit 5
elements before we find a leader pointer, then we know that this operand was
part of a complete EmbeddedPointerListInt and can retrieve the relative offset
to the ValueBundle containing the SILValue. If we do not visit 5 elements before
we find a leader pointer, we jump back 5 elements to ensure that we can find a
complete EmbeddedPointerListInt and retrieve the relative offset from the leader
operand of that EmbeddedPointerListInt to the ValueBundle. The key thing to
notice is that the case of a TrivialValueBundle is extremely fast and simple
since that will be the majority of the cases that are visited, while the more
complicated case of having a non-trivial ValueBundle still has a constant amount
of work and does not involve performing a scalability hurting linear search or
bisection along the operand list.
-->
