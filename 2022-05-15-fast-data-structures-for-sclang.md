---
title: Fast Data Structures for SCLang
---

I've long held the opinion that any programming language worth its salt should provide two fundamental data structures:
an integer-indexable collection (think `Array` in sclang) and an associative *key-value* dictionary-style container
supporting constant-time access. I think I can trace this back to working on a large Python code base, comparatively
early in my career. Coming from C, I found Python's lack of pointer arithmetic to be limiting at first, but eventually
became comfortable with building data structures from arrays and dictionarys.

When I started learning SCLang I looked for those two primitives, and found `Array` and `Dictionary` seemed to
mostly map the APIs and use cases that I was familiar with. As I've been working on Hadron, however, I've come to read a
great deal of library code and understand that my Python-like expectations of `Dictionary` are not entirely accurate.

Python dictionaries support constant-time access, so I have always assumed that `Dictionary`, or at least its faster
derived class `IdentityDictionary` would do the same. Let's take a look at how `IdentityDictionary` looks up values
given a key:

{{< highlight plain "linenos=table,lineNoStart=427" >}}
	at { arg key;
		_IdentDict_At
		^this.primitiveFailed
		/*^array.at(this.scanFor(key) + 1)*/
	}
{{< /highlight >}}

Well, that's promising. The `_IdentDict_At` token identifies a *primitive* in sclang, meaning an instruction to the
sclang interpreter to call a C++ function connected to that keyword and compiled into the interpreter binary. Primitives
are a common idiom in sclang library code, particularly for core data structures like `IdentityDictionary`. They are
often used when manipulating language internals, or for connecting to other C/C++ libraries (like for MIDI), or when
the language implementor wants to leverage the additional speed compiled C++ code offers over sclang.

To find the associated C++ code I usually just search the `lang/` subdirectory of the SuperCollider repository for that
exact primitive token:

{{< highlight shell >}}
lang % grep -Rn _IdentDict_At *
LangPrimSource/PyrListPrim.cpp:785:    definePrimitive(base, index++, "_IdentDict_At", prIdentDict_At, 2, 0);
{{< /highlight >}}

Once you search for a few of these you'll find that they all generally reside in the `LangPrimSource` subdirectory and
are associated with a `definePrimitive()` call just like this one. The argument following the primitive is the
associated function name, `prIdentDict_At`, and grepping for that string reveals the function definition in the same
file, `PyrListPrim.cpp`:

{{< highlight cpp "linenos=table,lineNoStart=416" >}}
int prIdentDict_At(struct VMGlobals* g, int numArgsPushed) {
    PyrSlot* a = g->sp - 1; // dict
    PyrSlot* key = g->sp; // key
    PyrObject* dict = slotRawObject(a);

    bool knows = IsTrue(dict->slots + ivxIdentDict_know);
    if (knows && IsSym(key)) {
        if (slotRawSymbol(key) == s_parent) {
            slotCopy(a, &dict->slots[ivxIdentDict_parent]);
            return errNone;
        }
        if (slotRawSymbol(key) == s_proto) {
            slotCopy(a, &dict->slots[ivxIdentDict_proto]);
            return errNone;
        }
    }

    identDict_lookup(dict, key, calcHash(key), a);
    return errNone;
}
{{< /highlight >}}

We start the function by extracting the arguments from the sclang stack inside of the `VMGlobals` structure `g->sp` (or
*stack pointer*). The code is expecting an IdentityDictionary at `sp - 1` and the key to lookup at `sp`. SuperCollider
stacks grow upward, meaning the stack pointer is incremented on a push, so this corresponds to a `this` pointer to the
dictionary pushed, followed by pushing the key.

You can ignore the bit in the middle from lines 421-431, it's dealing with some additional logic `IdentityDictionary`
offers around the `knows` member. The interesting call for our research is the `identDict_lookup` call on line 433.
Note that we are computing the hash of the key, and providing the pointer back to `a` to return the result.

Here's the text for `identDict_lookup`, also excerpted from `PyrListPrim.cpp`:

{{< highlight cpp "linenos=table,lineNoStart=343" >}}
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
{{< /highlight >}}

I've omitted further `knows` hijinks from the bottom half of the function, the key here is the call to
 `arrayAtIdentityHashInPairsWithHash(..)`, which is going to return an `index` to the element matching `hash` in the
 `array` member of `IdentityDictionary`, which it inherited from `Set`. The `SlotEq` check is to see if the key actually
 matches the one supplied, and if it does it copies the next element in the array into the result buffer and returns a
 flag which is ignored by the calling code in our example.

 Now we're getting somewhere. The name `arrayAtIdentityHashInPairsWithHash`, and the access pattern `index + 1`, implies
 that `IdentityDictionary` organizes its data as key/value pairs in an internal array. Let's complete the journey and
 look at the actual lookup code, still in the same `PyrListPrim.cpp`:

{{< highlight cpp "linenos=table,lineNoStart=185" >}}
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
{{< /highlight >}}

This is cryptic and uncommented. The strategy it follows is to use `hash` to identify a position in the array to start
searching from. If the element associated with `key` was the only element in the dictionary, then it would be stored at
the position calculated for `start` on line 190. Hash is a 32-bit unsigned number, addressing roughly 4 billion unique
values, so it can't be used directly as the index for `key`. So the code computes the *modulus* of hash based on the
size of the current array, using only the least significant bits of hash needed to compute the index of `key`. The
shifting right and then left by a single bit is a clever way of quickly dividing and multiplying by two, due to the fact
that the array stores key and value pairs, so the number of elements in the array `maxHash` is actually *half* the size
of the array, but the index will be doubled again to account for the inline pairs.

Taking the modulus of the hash is a classic technique of hash tables that allows indexing tractably-sized arrays. Using
part of the key can result in *collisions*, meaning two different hashes can map to the same `start` value. As a result
the code *probes* the array, checking each value in order from `start` until the end of the array. If either an empty
slot or a match is encountered the function returns that index immediately.

When inserting keys the we perform the same index computation, but if there's a colliding element already in place in
the hash table at that index, the inserting code tries to insert into the *next open element* in the array, incrementing
until it reaches the end of the array, then starting the second for loop at the first element in the array and
continuing until it encounters the `start` again.

So it makes sense that the searching code here would follow the same pattern. If it encounters an empty element in the
array that means the key is not in the array, because it would have been inserted in that empty element by the insertion
code if it was.

As an aside, I spent a few minutes trying to deduce the meaning of the `-2` return value on line 204. I think it's
entirely possible it would cause an interpreter crash if ever executed. The calling code uses that returned index with
the assumption that it is a valid index into the array, so calling `SlotEq` on a pointer 32 bytes (or 2 16-byte slots)
*before* the start of the array elements seems like a recipe for a potential segmentation fault, for any call of this
function on a *completely full* dictionary (so, no null entries) and for a key not in the dictionary. My suspicion is
that it never gets executed, however, because of the resizing logic applied within its mirror function, `identDictPut`:

{{< highlight cpp "linenos=table,lineNoStart=232" >}}
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
{{< /highlight >}}

I've cut out the top part of the function, the salient bits here are that it looks for an index in the current array,
copies the value into place in the slot following the key, then checks if it should also copy the key. The check for
`IsNil(slot)` on line 236 is trying to determine if this `put` operation is overwriting an existing value previously
associated with the key, or storing a new value. If the slot is `nil` this is a *new* value, so the following code
copies the key and then increases the `size` element by one. The `if` check on line 241 is interesting. The size of the
array is always going to be at most *twice* the size of the IdentityDictionary, because the array stores two elements
(key and value) for each single element in the dictionary. But this code resizes the array if it doesn't have
capacity for *triple* the size. This strategy ensures that there will always be some empty slots within the dictionary,
leaving that -2 crash bug to sleep in dead code for another day.

Note the re-hash happening on lines 247-254. A bigger array means a different index for every key in the hash table, so
as the entires are copied from the old array to the newer one they are each rehashed and copied into that new index.
It's painful from a performance perspective but a necessary step of any hash table resize.

## Runtime Performance

Another way to implement hash tables is to append colliding keys onto a list, so each element in the hash table is
itself a table of the colliding keys. For some reason it's easier for me to reason about the asymptotic runtime of the
table-of-colliding keys form than the sclang form, but I suspect they both have similar asymptotic runtime
characteristics. For hash tables that aren't full, and with "good" hashing functions, I would expect access times to be
roughly constant in hash table size.

In practice, I suspect storing collisions in subsequent entries greatly increases the potential for collisions,
resulting in a performance degredation when compared to storing the collisions in a per-bucket list. There's also some
interesting trade-offs here in terms of memory consumption. Chaining hash tables can be more tolerant of a full hash
table since that typically only means a few entries must be searched in the colliding bucket. To be fair, the chaining
may occur some additional overhead in storage of the individual buckets, and may not enjoy the same cache coherence as
the "all-in-one" strategy.

Storing collisions in subsequent buckets introduces an interesting order dependency to the sclang hash table. Some input
sequences will result in elements being stored at a significant distance from their computed indices, particularly for
dictionaries near that 2/3 capacity limit with few open slots.


## Dead Code

Curiously, there's a HashTable data structure in the lang/LangSource directory, but it's not used at all in the language
source. The scsynth code references it a few times.