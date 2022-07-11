---
title: "Fast Data Structures for SCLang"
date: "2022-07-09"
author: "Luke Nihlen"
description: "A discussion of the implementation of key data structures in the SuperCollider programming language."
---

I've long held that any programming language worth its salt should provide two fundamental data structures:

- an integer-indexable collection (think `Array` in sclang)
- an associative *key-value* dictionary-style container supporting amortized constant-time access.

I think I can trace this back to working on a large Python code base comparatively early in my career. Coming from C, I
initially found Python's lack of pointer arithmetic limiting. Eventually, Python taught me one could make any
arbitrary data structure with those two fundamental elements.

When learning any new programming language, I look for those two primitives. So, when I started to learn SuperCollider,
I found `Array` and `IdentityDictionary`. As I've been working on implementing Hadron, I've built all of the
sclang-accessible data structures from those two components. Recently, I implemented Hadron's version of the
`IdentityDictionary`. Because of [Hyrum's Law](https://www.hyrumslaw.com/), I want to ensure the Hadron version behaves
similarly to the existing sclang implementation. So, we will first study that.

## SCLang IdentityDictionary Implementation

Let's take a look at how `IdentityDictionary` looks up values given a key:

{{< code language=sclang title="SCClassLibrary/Common/Collections/Dictionary.sc excerpt">}}
	at { arg key;
		_IdentDict_At
		^this.primitiveFailed
		/*^array.at(this.scanFor(key) + 1)*/
	}
{{< /code >}}

Well, that's promising. The `_IdentDict_At` token identifies a *primitive* in sclang, meaning an instruction to the
sclang interpreter to call a C++ function connected to that keyword and compiled into the interpreter binary. Primitives
are a common idiom in sclang library code, particularly for methods supporting core data structures like
`IdentityDictionary`. I've seen them used for a variety of reasons in sclang:

- when manipulating language internals or changing values that aren't accessible from sclang
- connecting to other C/C++ libraries that don't have a sclang interface (e.g., MIDI, HID)
- when the language implementor wants to leverage the additional speed compiled C++ code offers over sclang

We want to read the C++ code associated with the `_IdentDict_At` primitive. I usually search the `lang/` subdirectory of
the SuperCollider repository for the primitive token:

```shell
lang % grep -Rn _IdentDict_At *
LangPrimSource/PyrListPrim.cpp:785:    definePrimitive(base, index++, "_IdentDict_At", prIdentDict_At, 2, 0);
```

Once you search for a few of these you'll find that they usally reside in the `LangPrimSource` subdirectory and are
associated with a `definePrimitive()` call just like this one. The argument following the primitive name is the
associated function name, `prIdentDict_At`, and grepping for that function name reveals the function definition in the
same file, `PyrListPrim.cpp`:

{{< code language=cpp title="lang/LangPrimSource/PyrListPrim.cpp excerpt" >}}
int prIdentDict_At(struct VMGlobals* g, int numArgsPushed) {
    PyrSlot* a = g->sp - 1; // dict
    PyrSlot* key = g->sp; // key
    PyrObject* dict = slotRawObject(a);

    /* SNIP */

    identDict_lookup(dict, key, calcHash(key), a);
    return errNone;
}
{{< /code >}}

We start the function by extracting the arguments from the sclang stack inside the `VMGlobals` structure `g->sp` (or
*stack pointer*). The code expects an `IdentityDictionary` at `sp - 1` and the key to lookup at `sp`. SuperCollider
stacks grow upward, meaning the stack pointer is incremented on a push, so this corresponds to the caller pushing the
dictionary pointer, followed by the key.

I've omitted some code dealing with additional logic `IdentityDictionary` offers around the `knows` member. The
relevant code for our research is the `identDict_lookup` call.

Here's the code for the first part of `identDict_lookup`, also excerpted from `PyrListPrim.cpp`:

{{< code language=cpp title="lang/LangPrimSource/PyrListPrim.cpp excerpt" >}}
bool identDict_lookupNonNil(PyrObject* dict, PyrSlot* key, int hash, PyrSlot* result) {
again:
    PyrSlot* dictslots = dict->slots;
    PyrSlot* arraySlot = dictslots + ivxIdentDict_array;

    if (isKindOfSlot(arraySlot, class_array)) {
        PyrObject* array = slotRawObject(arraySlot);

        int index = arrayAtIdentityHashInPairsWithHash(array, key, hash);
        if (SlotEq(key, array->slots + index)) {
            slotCopy(result, &array->slots[index + 1]);
            return true;
        }
    }

    /* SNIP */
}
{{< /code >}}

I've omitted further `knows` hijinks from the bottom half of the function. We follow the code to the call to
 `arrayAtIdentityHashInPairsWithHash(..)`, which returns an `index` to the element matching `hash` in the `array` member
 of `IdentityDictionary`, which it inherited from `Set`. The `SlotEq` check is to see if the key matches the one
 supplied in the argument. If the keys match, it copies the next element in the array into the result buffer and
 returns.

 Now we're getting somewhere. The name `arrayAtIdentityHashInPairsWithHash`, and the access pattern `index + 1`, implies
 that `IdentityDictionary` organizes its data as key/value pairs in adjacent elements in an internal array. Let's
 complete the journey and look at the actual lookup code, still in the same `PyrListPrim.cpp`:

{{< code language=cpp title="lang/LangPrimSource/PyrListPrim.cpp excerpt" >}}
int arrayAtIdentityHashInPairsWithHash(PyrObject* array, PyrSlot* key, int hash) {
    PyrSlot *slots, *test;
    unsigned int i, start, end, maxHash;

    maxHash = array->size >> 1;
    start = (hash % maxHash) << 1;
    end = array->size;
    slots = array->slots;
    for (i = start; i < end; i += 2) {
        test = slots + i;
        if (IsNil(test) || SlotEq(test, key))
            return i;
    }
    end = start - 2;
    for (i = 0; i <= end; i += 2) {
        test = slots + i;
        if (IsNil(test) || SlotEq(test, key))
            return i;
    }
    return -2;
}
{{< /code >}}

This code is terse and uncommented, requiring some careful reading. The algorithm uses `hash` to identify a position in
the array to start its search. If the element associated with `key` is the only element in the dictionary, then it would
be stored at the position calculated for `start`. The hash is a 32-bit signed number, addressing roughly 2 billion
positive values, so we can't use it directly as the index for `key`. So the code computes the *modulus* of `hash` on the
size of the current array, using only the least significant bits of hash needed to derive the index of `key`. The
shifting right and left with `>>` and `<<` divides and then multiplies the values by two, accounting for the
elements in the arrays in key/value *pairs*.

Taking the modulus of the hash is a classic technique of hash tables that allows indexing tractably-sized arrays. Using
part of the key can result in *collisions*, meaning two different hashes can map to the same `start` value. The code *probes* the array, checking each value in order from the `start`. If it encounters an empty slot or match, the function returns that index immediately.

To insert a key, we perform the identical index computation and probing behavior, inserting the new element in the
first open position the algorithm finds. So, it makes sense that the searching code here would follow the same pattern.
The key is not in the dictionary if it encounters an empty element while probing.

I spent a few minutes trying to deduce the meaning of the `-2` return value on the final line. I think it would cause
the interpreter to crash if ever executed. The calling code uses that returned index with the assumption that it is a
valid index in the array, so calling `SlotEq` on a pointer 32 bytes (or two 16-byte slots) *before* the start of the
array elements seems like a buffer underrun and possible segmentation fault. The crash would occur for any call of this
function on a *full* dictionary (so, no null entries) and a key not in the dictionary. I suspect that it never gets
executed because of the resizing logic applied within its mirror function, `identDictPut`:

{{< code language=cpp title="lang/LangPrimSource/PyrListPrim.cpp excerpt" >}}
int identDictPut(struct VMGlobals* g, PyrObject* dict, PyrSlot* key, PyrSlot* value) {

    /* SNIP */

    index = arrayAtIdentityHashInPairs(array, key);
    slot = array->slots + index;
    slotCopy(&slot[1], value);
    g->gc->GCWrite(array, value);
    if (IsNil(slot)) {
        slotCopy(slot, key);
        g->gc->GCWrite(array, key);
        size = slotRawInt(&dict->slots[ivxIdentDict_size]) + 1;
        SetRaw(&dict->slots[ivxIdentDict_size], size);
        if (array->size < size * 3) {
            PyrObject* newarray;
            newarray = newPyrArray(g->gc, size * 3, 0, false);
            newarray->size = ARRAYMAXINDEXSIZE(newarray);
            nilSlots(newarray->slots, newarray->size);
            slot = array->slots;
            for (i = 0; i < array->size; i += 2, slot += 2) {
                if (NotNil(slot)) {
                    index = arrayAtIdentityHashInPairs(newarray, slot);
                    newslot = newarray->slots + index;
                    slotCopy(&newslot[0], &slot[0]);
                    slotCopy(&newslot[1], &slot[1]);
                }
            }
            SetRaw(&dict->slots[ivxIdentDict_array], newarray);
            g->gc->GCWriteNew(dict, newarray); // we know newarray is white so we can use GCWriteNew
        }
    }
    return errNone;
}
{{< /code >}}

I've cut out the top part of the function. The salient bits here are that it looks for an index in the current array,
copies the value into place in the slot following the key, then checks if it should also copy the key. The check for
`IsNil(slot)` determines if this `put` operation is overwriting an existing value previously associated with the key. If
the slot is `nil` this is a *new* value, so the following code copies the key and then increases the `size` element by
one.

The following `if` size comparison is interesting. The array stores two elements (key and value) for each Dictionary
entry. This code resizes the array if it doesn't have room for *triple* the size. This strategy ensures that there will
always be empty slots within the dictionary, leaving that -2 crash bug to sleep in dead code for another day.

The hash table strategy used here is classically called [Open Addressing with Linear
Probing](https://en.wikipedia.org/wiki/Open_addressing). This style of hash table favors cache access at the expense of
a possible "clustering" problem; when entries clump up in the table next to each other due to the linear probing
behavior of inserting colliding elements into the *next available slot* in the storage array. Clustering worsens as the
hash table fills up, which explains the code in SCLang to limit the `IdentityDictionary` to two-thirds full.

Note the re-hash happening after the resize. Resizing the array changes the index for every key in the hash table, so
each element is re-hashed during insertion.

## Next Steps

I took what I learned from this study and implemented Hadron's `IdentityDictionary` primitives for insertion and key
lookup. Long term, I'm not confident that the specific trade-offs made in the current implementation work well for
Hadron. While I haven't completed the design yet, I anticipate that Hadron's garbage collector will be very different
from the one inside SCLang, resulting in new options in terms of data structure design and performance characteristics.

I've tried to avoid premature optimization while working on Hadron; outside of some fundamental architecture choices
(e.g., JIT) I'd say I've been mostly successful. I'm still in pursuit of *correctness* first. While implementing major
subsystems, I've had to change them enough that I'm confident I would have wasted a lot of early optimization work had I
decided to pause on the correctness effort and go and optimize. So, mostly this has felt like the right call.

However, in implementing these data structures, I must admit that I struggled mightily against the temptation to
redesign them. I will eventually implement a `HadronHashTable`, and likely will migrate all of the Hadron language data
structures that currently use `IdentityDictionary` to that. I want to think I'll wait until I have overall performance
data that suggests a data structure redesign would be the most impactful optimizations I could make on Hadron, but I
might not be *that* disciplined.

Getting to correctness seems like a ton of work, making this optimization work feel even further away. My intuition
suggests that dictionary optimizations would be impactful, particularly considering that the `Event` class, central to
many SuperCollider pattern-based workflows, derives from `IdentityDictionary`. Time will tell.