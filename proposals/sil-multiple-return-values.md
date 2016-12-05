---
layout: proposal
title: SIL Multiple Return Values
categories: proposals
---


## Multiple Return Value Implementation

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
