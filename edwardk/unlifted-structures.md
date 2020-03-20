# Unlifted Structures

An `ArrayArray#` is just an `Array#` with a modified invariant. It points directly to other unlifted `ArrayArray#`'s or `ByteArray#`'s.

While those live in `#`, they are garbage collected objects, so this all lives on the heap.

They were added to make some of the Data Parallel Haskell stuff fast when it has to deal with nested arrays.

I'm currently abusing them as a placeholder for a better thing.

**Edit:** In the reddit thread for this post [I wound up explaining a bit about what all the fiddly `#`'s mean.](https://www.reddit.com/r/haskell/comments/3im7ha/edward_kmett_unlifted_structures/cuhski4) If you are having trouble following along, you might want to start there.

# The Problem

Consider the scenario where you naively write a classic doubly-linked list (DLL) in Haskell.

```haskell
data DLL = DLL (IORef (Maybe DLL)) (IORef (Maybe DLL))
```

Chasing from one DLL to the next requires following 3 pointers on the heap!

```
DLL ~> IORef (Maybe DLL) ~> MutVar# RealWorld (Maybe DLL) ~> Maybe DLL ~> DLL
```

That is 3 levels of indirection!

We can trim one easily by simply unpacking the IORef with `-funbox-strict-fields` or `UNPACK`.

We can trim another by adding a `Nil` constructor to `DLL` and worsening our representation.

```haskell
data DLL = DLL !(IORef DLL) !(IORef DLL) | Nil
```

but now we're still stuck with one level of indirection 

```
DLL ~> MutVar# RealWorld DLL ~> DLL
```

This means that every operation we perform on this structure will be about half of the speed of an implementation in most other languages assuming we're memory bound on loading things into cache!

# Making Progress

I have been working on a number of data structures where the indirection of going from something in `*` out to an object in `#` which contains the real pointer to my target and coming back effectively doubles my runtime.

We need to go out to the `MutVar#` because we are allowed to put the `MutVar#` onto the mutable list when we dirty it. There is a well defined write-barrier there.

I could change out the representation to use

```haskell
data DLL = DLL (MutableArray# RealWorld DLL) | Nil
```

I can just store two pointers in the `MutableArray#` every time, but this doesn't help _much_ directly. It has reduced the amount of distinct addresses in memory from 3 per object to 2.

I still have to go out to the heap from my `DLL` and get to the array object and then chase it to the next `DLL` and chase that to the next array. I do get my two pointers together in memory though. I'm paying for a card marking table as well, which I don't particularly need with just two pointers, but we can shed that with the `SmallMutableArray#` machinery added back in 7.10, which is just the old array code as a new data type, which can speed things up a bit when you don't have very big arrays:

```haskell
data DLL = DLL (SmallMutableArray# RealWorld DLL) | Nil
```

But what if I wanted my object itself to live in # and have two mutable fields and be able to share the same write barrier?

An `ArrayArray#` points directly to other unlifted array types. What if we have one `# -> *` wrapper on the outside to deal with the impedence mismatch between the imperative world and Haskell, and then just let the `ArrayArray#`'s hold other `ArrayArray#`s?

```haskell
data DLL = DLL (MutableArrayArray# RealWorld)
```

now I need to make up a new `Nil`, which I can just make be a special `MutableArrayArray#` I allocate on program startup. I can even abuse pattern synonyms. Alternately I can exploit the internals further to make this cheaper.

Then I can use the `readMutableArrayArray#` and `writeMutableArrayArray#` calls to directly access the preceding and next entry in the linked list.

So now we have one `DLL` wrapper which just 'bootstraps me' into a strict world, and everything there lives in `#`.

```haskell
next :: DLL -> IO DLL
next (DLL m) = IO $ \s -> case readMutableArrayArray# s of 
   (# s', n #) -> (# s', DLL n #)
```

It turns out GHC is quite happy to optimize all of that code to keep things unboxed. The `DLL` wrappers get removed pretty easily when they are known strict and you chain operations of this sort!

In each of these calls we leap from box to box like [Maru the cat](https://www.youtube.com/watch?v=jgxL-PwmY7s), and it is the compiler's job to make the boxes all go away.

With this I've made a strict little universe and shoved it out on the heap.

But as it stands, we don't have a portal back to the "real world". 

# Unboxed

Now I have one outermost indirection pointing to an array that points directly to other arrays.

I'm stuck paying for a card marking table per object, but I can fix that by duplicating the code for `MutableArrayArray#` and using a `SmallMutableArray#`. I can hack up primops that let me store a mixture of `SmallMutableArray#` fields and normal ones in the data structure. Operationally, I can just `unsafeCoerce#` the existing `SmallMutableArray#` primitives to change the kind of one of the arguments it takes!

This is almost ideal, but not quite. I often have fields that would be best left unboxed.

```haskell
data DLL = DLL !Int !(IORef DLL) !(IORef DLL) | Nil
```

was able to unpack the `Int`, but we lost that. We can currently at best point one of the entries of the `SmallMutableArray#` at a boxed or add a `MutableByteArray#` for all of our misc. data and shove the `Int` in question in there.

If I were to implement a HAMT, like `HashMap` this way I need to store masks and administrivia as I walk down the tree. Having to go off to the side costs me almost the entire win from avoiding the first pointer chase!

The other day I posted about this to the `ghc-devs@` mailing list, and Ryan Yates suggested that he may be able to help.

If we had a heap object we could construct that had n words with unsafe access and m pointers to other heap objects, one that could put itself on the mutable list when any of those pointers changed then I could shed this last factor of two in all circumstances.

# Prototype

Over the last few days I've put together a small prototype implementation with a few non-trivial imperative data structures for things like Tarjan's link-cut trees, the list labeling problem and order-maintenance.

[https://github.com/ekmett/structs](https://github.com/ekmett/structs)

Notable bits:

[Data.Struct.Internal.LinkCut](https://github.com/ekmett/structs/blob/9ff2818f888aff4789b7a41077a674a10d15e6ee/src/Data/Struct/Internal/LinkCut.hs) provides an implementation of link-cut trees in this style.

[Data.Struct.Internal](https://github.com/ekmett/structs/blob/9ff2818f888aff4789b7a41077a674a10d15e6ee/src/Data/Struct/Internal.hs) provides the rather horrifying guts that make it go fast.

Once compiled with -O or -O2, if you look at the core, almost all the references to the `LinkCut` or `Object` data constructor get optimized away, and we're left with beautiful strict code directly mutating our underlying representation.

-[Edward Kmett](mailto:ekmett@gmail.com)

August 27th, 2015