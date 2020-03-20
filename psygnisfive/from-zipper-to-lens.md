# From Zipper To Lens

Before we start, some prefacing junk:

```haskell
{-# LANGUAGE RankNTypes #-}

import Control.Applicative
```

How can we pull out an element from a structure to focus on it without losing our sense of where it is in the structure? One approach is zippers (if you don't know what a zipper is, read up!). Let's make a zipper for pairs:

```haskell
data PairContext a b = InFst b | InSnd a

data PairZipper a b c = PairZipper c (PairContext a b)
```

A ``PairZipper a b c`` is a pair in ``(a,b)`` focused on something of type ``c``. We can look into the components of a pair (and wrap zippers back up into pairs) like so:

```haskell
lookFstPZ :: (a,b) -> PairZipper a b a
lookFstPZ (a,b) = PairZipper a (InFst b)

unzipFstPZ :: PairZipper a b a -> (a,b)
unzipFstPZ (PairZipper a (InFst b)) = (a,b)

lookSndPZ :: (a,b) -> PairZipper a b b
lookSndPZ (a,b) = PairZipper b (InSnd a)

unzipSndPZ :: PairZipper a b b -> (a,b)
unzipSndPZ (PairZipper b (InSnd a)) = (a,b)
```

And we can look at or modify a focused element:

```haskell
viewPZ :: PairZipper a b c -> c
viewPZ (PairZipper c l) = c

overPZ :: (c -> c) -> PairZipper a b c -> PairZipper a b c
overPZ f (PairZipper c l) = PairZipper (f c) l
```

Question: how can we compose these so that we can ``lookFstPZ``, then ``lookSndPZ``, and have this be another kind of Zipper-like thing?

Answer: too hard, forget it.

How about a zipper for lists?

```haskell
data ListContext a = ListCtx [a] [a]

data ListZipper a = ListZipper a (ListContext a)

lookHeadLZ :: [a] -> ListZipper a
lookHeadLZ (a:as) = ListZipper a (ListCtx [] as)

unzipLZ :: ListZipper a -> [a]
unzipLZ (ListZipper a (ListCtx bfr aft)) = bfr ++ [a] ++ aft

viewLZ :: ListZipper a -> a
viewLZ (ListZipper a l) = a

overLZ :: (a -> a) -> ListZipper a -> ListZipper a
overLZ f (ListZipper a l) = ListZipper (f a) l
```

Question: how can we compose these? Again, too hard. Forget it!

```haskell
data Tree a = Leaf a | Branch a (Tree a) (Tree a)

data TreeCtx a = Here
               | InLeft a (TreeCtx a) (Tree a)
               | InRight a (Tree a) (TreeCtx a)

data TreeZipper a = TreeZipper (Tree a) (TreeCtx a)

lookRootTZ :: Tree a -> TreeZipper a
lookRootTZ t = TreeZipper t Here

unzipTZ :: TreeZipper a -> Tree a
unzipTZ (TreeZipper t Here) = t
unzipTZ (TreeZipper l (InLeft a c r)) = unzipTZ (TreeZipper (Branch a l r) c)
unzipTZ (TreeZipper r (InRight a l c)) = unzipTZ (TreeZipper (Branch a l r) c)

viewTZ :: TreeZipper a -> a
viewTZ (TreeZipper (Leaf a) c) = a
viewTZ (TreeZipper (Branch a l r) c) = a

overTZ :: (a -> a) -> TreeZipper a -> TreeZipper a
overTZ f (TreeZipper (Leaf a) c) = TreeZipper (Leaf (f a)) c
overTZ f (TreeZipper (Branch a l r) c) = TreeZipper (Branch (f a) l r) c
```

Question: ... well you get the idea.

Why not just use...

```haskell
data Focused s a = Focused { foc :: a, loc :: a -> s }
```

We can now be pretty generic:

```haskell
unfocusF :: Focused s a -> s
unfocusF (Focused foc loc) = loc foc

type Focuser s a = s -> Focused s a

viewF :: Focuser s a -> s -> a
viewF l s = foc (l s)

overF :: Focuser s a -> (a -> a) -> s -> s
overF l f s = let Focused foc loc = l s
              in loc (f foc)
```

Let's also introduce a new function that makes life simpler:

```haskell
setF :: Focuser s a -> a -> s -> s
setF l a s = overF l (const a) s
```

Let's look at our various zippers, but this time viewed as ``Focused`` elements. First pairs:

```haskell
lookFstF :: Focuser (a,b) a
lookFstF (a,b) = Focused a (\a' -> (a',b))

lookSndF :: Focuser (a,b) b
lookSndF (a,b) = Focused b (\b' -> (a,b'))
```

And lists:

```haskell
lookHeadF :: Focuser [a] a
lookHeadF (a:as) = Focused a (\a' -> a':as)
```

And trees:

```haskell
lookRootF :: Focuser (Tree a) a
lookRootF (Leaf a) = Focused a (\a' -> Leaf a')
lookRootF (Branch a l r) = Focused a (\a' -> Branch a' l r)
```

We can now answer our question about how to compose:

```haskell
(>-) :: Focuser a b -> Focuser b c -> Focuser a c
(l >- l') a = let Focused foc loc = l a
                  Focused foc' loc' = l' foc
              in Focused foc' (loc.loc')
```

This does exactly what you'd hope: ``lookFstF >- lookSndF :: Focuser ((a,b),c) b``  is exactly
the gadget that looks at the first element of the outer pair, then at the second of that
same for, say, lists: ``lookHeadF >- lookFstF :: Focuser [(a,b)] a`` looks at the first
component of the head of a list of pairs. and so on generically.

But over has an unusual restriction: the action on the focused element can't change the
elements type. Sometimes that seems reasonable, like for a list which have to be homogeneously
typed, but other times it's an unnecessary constraint, such as for pairs. Why can't we, for
instance, focus on the first component of an ``(a,b)`` and then somehow act on it to produce
an ``(a',b)``? We need to generalize focusing for this to work:

```haskell
data Focused' t a b = Focused' { foc' :: a, loc' :: b -> t }

type Focuser' s t a b = s -> Focused' t a b
```

Fortunately, that's _all_ we have to change. The rest is identical, other than types:

```haskell
unfocusF' :: Focused' s a a -> s
unfocusF' (Focused' foc loc) = loc foc

viewF' :: Focuser' s t a b -> s -> a
viewF' l s = foc' (l s)

overF' :: Focuser' s t a b -> (a -> b) -> s -> t
overF' l f s = let Focused' foc loc = l s
               in loc (f foc)

setF' :: Focuser' s t a b -> b -> s -> t
setF' l b s = overF' l (const b) s

lookFstF' :: Focuser' (a,b) (a',b) a a'
lookFstF' (a,b) = Focused' a (\a' -> (a',b))

lookSndF' :: Focuser' (a,b) (a,b') b b'
lookSndF' (a,b) = Focused' b (\b' -> (a,b'))

lookHeadF' :: Focuser' [a] [a] a a
lookHeadF' (a:as) = Focused' a (\a' -> a':as)

lookRootF' :: Focuser' (Tree a) (Tree a) a a
lookRootF' (Leaf a) = Focused' a (\a' -> Leaf a')
lookRootF' (Branch a l r) = Focused' a (\a' -> Branch a' l r)
```

We can again compose:

```haskell
(>--) :: Focuser' s t a b -> Focuser' a b u v -> Focuser' s t u v
(l >-- l') a = let Focused' foc loc = l a
                   Focused' foc' loc' = l' foc
               in Focused' foc' (loc.loc')
```

So we have some nice examples now, for instance: ``setF' lookFstF' 3 ("a","b") == (3,"b")``

The modern lens type, however, aims to fuse this functionality together, by finding a single type that can somehow provide both ``view`` behavior and ``over`` behavior.

If we inspect the types of these two functions

```haskell
viewF' :: Focuser' s t a b -> s -> a
overF' :: Focuser' s t a b -> (a -> b) -> s -> t
```

we see that there is some similarity: both look roughly like ``Focuser' s t a b -> ... -> s -> ...``. Let's line things up nicer:

```haskell
viewF' :: Focuser' s t a b ->                 s -> a
overF' :: Focuser' s t a b ->   (a -> b) ->   s -> t
```

If we could somehow have a type that, in one moment can be equivalent to ``a`` and in the next ``t``, we would be closer to our goal. Similarly, if we had a type that was equivalent to ``a`` one moment and ``b`` the next, we could also supply the identity function in the former situations and get our ``viewF'`` type out. That is to say, abstractly we need a type ``b'`` that is equivalent to ``a`` or ``b`` depending, and similarly ``t'``. We ought to factor out the "depending", so instead let's use a type operator, to say that ``f b`` is equivalent to ``a`` or ``b`` depending, and similarly ``f t``. Depending on what? Well, the choice of ``f``!

Abstractly then what we need is

```haskell
Focuser' s t a b -> (a -> f b) -> s -> f t
```

This gives us our general type:

```haskell
type Lens s t a b = forall f. Functor f => (a -> f b) -> s -> f t
```

Choosing functors, not just type operators, makes using lenses much more convenient, and guarantees certain properties about how values of this type are defined.

If we pick ``f`` appropriately, ``a -> f b`` is equivalent to ``a -> a`` and ``s -> f t`` is equivalent to ``s -> a``, letting us define ``view`` functionality. The easiest way to do this is by choosing ``f = Const a``, where ``Const`` is defined as

```haskell
data Const a b = Const { getConst :: a }
```

with the functor instance

```haskell
instance Functor (Const a) where
  fmap f (Const a) = Const a
```

Instantiating ``f = Const a``, we get

```haskell
(a -> Const a b) -> s -> Const a t
```

Now all we need to do is somehow turn such a function into ``s -> Const a t`` by
supplying another function ``a -> Const a b`` as it's argument. The constructor ``Const``
is precisely what we need. Thus, given

```haskell
f :: (a -> Const a b) -> s -> Const a t
```

we have that

```haskell
f Const :: s -> Const a t
```

We can now extract the a value using the ``getConst`` function:

```haskell
view :: Lens s t a b -> s -> a
view l s = getConst (l Const s)
```

Similarly, we need to pick ``f`` so that ``a -> f b`` is equivalent to ``a -> b`` and ``s -> f t`` to ``s -> t``. This is somewhat easier, by
choosing ``f = Identity``:

```haskell
data Identity a = Identity { getIdentity :: a }

instance Functor Identity where
  fmap f (Identity a) = Identity (f a)
```

The definition of ``over`` is subtly different but no harder to find:

```haskell
over :: Lens s t a b -> (a -> b) -> s -> t
over l f s = getIdentity (l (Identity . f) s)
```

We can of course convert back and forth between ``Focuser'``s and ``Lens``es.

```haskell
lens :: Focuser' s t a b -> Lens s t a b
lens l f s = let Focused' foc loc = l s
             in fmap loc (f foc)

focuser :: Lens s t a b -> Focuser' s t a b
focuser l s = Focused' (view l s) (\b -> over l (const b) s)
```

That these are mutual inverses is relatively easy to show, and this
demonstrates that ``Lens s t a b`` is equivalent to ``Focuser' s t a b``.

A convenience type can be defined:

```haskell
type SimpleLens s a = Lens s s a a
```

Here are our ``Focuser'``s again, this time as ``Lens``es:

```haskell
_1 :: Lens (a,b) (a',b) a a'
_1 f (a,b) = (\a' -> (a',b)) <$> f a

_2 :: Lens (a,b) (a,b') b b'
_2 f (a,b) = (\b' -> (a,b')) <$> f b

_head :: SimpleLens [a] a
_head f (a:as) = (\a' -> a':as) <$> f a

_root :: SimpleLens (Tree a) a
_root f (Leaf a) = (\a' -> Leaf a') <$> f a
_root f (Branch a l r) = (\a' -> Branch a' l r) <$> f a
```

Notice that this is more or less exactly how things would behave if we focused
with whichever focuser, used an action on the focus, and then unfocused. Except here,
the unfocus function application is not the normal application but is instead
functorial, using ``fmap`` in its infix form ``(<$>)``.

That means that we can compose lenses as functions with ``(.)`` instead of defining a custom
composition, and we get something rather sensible looking:

```haskell
_head._1 :: SimpleLens [(a,b)] a
  = \f ((a,b):ps) -> (\p' -> p':ps) <$> ((\a' -> (a',b)) <$> f a)
```

Sometimes this is described as composing "backwards", but it really shouldn't be
seen like that. Instead, you just need to think of lenses as things which turn
actions on parts into actions on wholes, so ``_1`` doesn't "retrieve" the first component
of a pair, rather, it turns an action on the first component into an action on the whole
pair. Same with ``_head``: it turns an action on the head element into an action on the
whole list. That means that if you compose two lenses, you're making an action on the whole
that runs an action on a part that runs an action on an even smaller part. That's just what
fmap does: it pushes actions down into structure. Lenses just happen to do that in a
very specific way, to a focused place, rather than in the usual "apply everywhere" way
that normal functor intuitions provide.

This schema also lets us produce lenses really easily. The general picture is

```haskell
_l f m = (\a' -> [a'/a]m) <$> f a
```

where ``m`` is some pattern with variable ``a``, and ``[a'/a]m`` is the pattern with all occurrences
of ``a`` replaced by ``a'``, just so we don't get confused with the new bound variable. This provides a general schema for making lenses for your own types. It even lets us make funky lenses that focus on "non-elements", such as the first two elements of a triple:

```haskell
_12 :: SimpleLens (a,b,c) (a,b)
_12 f (a,b,c) = (\(a',b') -> (a',b',c)) <$> f (a,b)
```

or the heads of both elements of a pair of lists:

```haskell
_1head2head :: SimpleLens ([a],[b]) (a,b)
_1head2head f (a:as,b:bs) = (\(a',b') -> (a':as,b':bs)) <$> f (a,b)
```