# I love profunctors. They're so easy.

<!-- ∷ ←↑→↓↔ ↦　⇒ ∙∘ m— n− nbsp' '-->

“_I love profunctors. They're so easy._” —beaky on [#haskell](irc://irc.freenode.net/haskell)

[![Reddit](http://reddit.com/static/spreddit2.gif)](http://www.reddit.com/r/haskell/comments/1bg21c/so_i_accidentally_a_profunctors_tutorial/ "Reddit comments") [![G+](http://www.gstatic.com/images/icons/gplus-16.png)](https://plus.google.com/100604691747126657305/posts/Q5Wyc6WaVBa "G+ thread")
[`[日本語版]`](http://d.hatena.ne.jp/its_out_of_tune/20130407/1365350952)


## Covariant Functors

I hope we can all agree that functors are useful, and that we are all familiar with them. Just for the record, the `Functor` typeclass can be thought of as follows:

``` haskell
class Functor f where
    fmap ∷ (a → b) → f a → f b
```

Given a value `x ∷ f a` and a function `g ∷ a → b`, then `fmap g x ∷ f b`. [Simples](http://www.youtube.com/watch?v=M0mXUC0cUPg)!

However, Haskell's `Functor` is only one of many _functors_ in the mathematical sense. It is in fact a _covariant_ functor, meaning that `fmap` preserves the direction of the arrows:

``` haskell
     g ∷   a →   b
fmap g ∷ f a → f b
```

See? Both arrows point to the right.


## Contravariant Functors

Let's start with a motivational example. We call a function returning a `Bool` and taking one argument a [`Predicate`][], indicating the [truthiness](http://en.wikipedia.org/wiki/Truthiness) of its argument:

``` haskell
type Predicate a = a → Bool
```

Is [`Predicate`][] a `Functor`?
@@@
![Grumpy Cat says: NO.](https://www.fpcomplete.com/media/d06e3d62-09c1-4261-ab42-c472b9b247b3.jpeg)
@@@

Is [`Predicate`][] a _functor_?
@@@
Let's ask lambdabot:
```
<liyang> @faq Is Predicate a functor?
<lambdabot> The answer is: Yes! Haskell can do that.
```
@@@

From [`Data.Functor.Contravariant`][]:
``` haskell
class Contravariant f where
    contramap ∷ (b → a) → f a → f b
```

This characterises _contravariant_ functors. Note that `contramap` swaps the direction of the arrow, in _contrast_ to `fmap`:

``` haskell
          g ∷   a ←   b
contramap g ∷ f a → f b
```

Let's make a [`Contravariant`][] [`Predicate`][]:

``` active haskell
{-# LANGUAGE UnicodeSyntax #-}
import Control.Applicative
import Data.Functor.Contravariant (Contravariant (..))
-- show
newtype Predicate a = Predicate { getPredicate ∷ a → Bool }

instance Contravariant Predicate where
    contramap g (Predicate p) = Predicate (p . g)

veryOdd ∷ Predicate Integer
veryOdd = contramap (`div` 2) (Predicate odd)

main ∷ IO ()
main = print $ getPredicate veryOdd <$> [0 .. 11]
-- /show
 
```

[Can you tell what the output is yet?](http://www.youtube.com/watch?v=EUERRPKA2_w)


### Examples of Contravariant Functors

``` active haskell
{-# LANGUAGE UnicodeSyntax #-}
import Data.Function
import Data.Functor.Contravariant (Contravariant (..))
-- show
newtype Const a b = Const a
instance Contravariant (Const a) where
    contramap _ (Const a) = Const a

newtype Comparison a = Comparison (a → a → Ordering) -- e.g. compare
instance Contravariant Comparison where
    contramap g (Comparison comp) = Comparison (comp `on` g)

newtype Op b a = Op (a → b)
instance Contravariant (Op b) where
    contramap g (Op f) = Op (f . g)
-- /show
main = return ()
```

The above (and more) are already provided by [`Data.Functor.Contravariant`][].


## Bifunctors

A bifunctor in the mathematical sense is a functor of two arguments; three arguments would make trifunctors…
@@@
followed by quadri-, quinque-, sexa-, and septi-functor. A multifunctor is fine too.
@@@

In Haskell this means a parametric type of kind `* → * → *`. Familiar bifunctors include `Either`, `(,)` or even `(→)`…

However, the [`Bifunctor`][] typeclass correspond only to bifunctors that are _covariant in both arguments_:

``` haskell
class Bifunctor f where
    bimap ∷ (a → c) → (b → d) → f a b → f c d
```

``` haskell
      g   ∷   a    →   c
        h ∷     b  →     d
bimap g h ∷ f a b  → f c d
```

Both `Either` and `(,)` are [`Bifunctor`][]s. There are also [`Biapplicative`][], [`Bifoldable`][] and [`Bitraversable`][] classes, if you feel inclined to investigate. [Watch out](http://www.youtube.com/watch?v=OMAIsqvTh7g) for [`Clown`][]s and [`Joker`][]s [popling](http://strictlypositive.org/CJ.pdf) out around the corner though.


### Exercise: `instance Bifunctor Either`

@@@
``` active haskell
{-# LANGUAGE UnicodeSyntax #-}
class Bifunctor f where
    bimap ∷ (a → c) → (b → d) → f a b → f c d
-- show
instance Bifunctor Either where
    bimap g h = either (Left . g) (Right . h)
-- /show
main = return ()
```
@@@


### Exercise: `instance Bifunctor (,)`

@@@
``` active haskell
{-# LANGUAGE UnicodeSyntax #-}
import Control.Arrow
class Bifunctor f where
    bimap ∷ (a → c) → (b → d) → f a b → f c d
-- show
instance Bifunctor (,) where
    bimap = (***)
-- /show
main = return ()
```
@@@


## Profunctors

A [`Profunctor`][] is [just](http://james-iry.blogspot.jp/2009/05/brief-incomplete-and-mostly-wrong.html) a bifunctor that is contravariant in the first argument and covariant in the second. [What's the problem](http://www.quickmeme.com/meme/3r455v/)?

``` haskell
class Profunctor f where
    dimap ∷ (c → a) → (b → d) → f a b → f c d
```

``` haskell
      g   ∷   a    ←   c
        h ∷     b  →     d
dimap g h ∷ f a b  → f c d
```

If we only want to map over one of the two type arguments, there are:
``` active haskell
{-# LANGUAGE UnicodeSyntax #-}
import Data.Profunctor (Profunctor (dimap))
-- show
lmap ∷ Profunctor f ⇒ (c → a) → f a b → f c b
lmap = (`dimap` id)

rmap ∷ Profunctor f ⇒ (b → d) → f a b → f a d
rmap = (id `dimap`)
-- /show
main = return ()
```

The simplest and most common [`Profunctor`][] is `(→)`. The specialised type of `dimap` would be:

``` haskell
dimap :: (c → a) → (b → d) → (a → b) → (c → d)
```


### Exercise: `instance Profunctors (→)`

@@@
``` active haskell
{-# LANGUAGE UnicodeSyntax #-}
class Profunctor f where
    dimap ∷ (c → a) → (b → d) → f a b → f c d
-- show
instance Profunctor (→) where dimap g h f = h . f . g
-- /show
main = return ()
```
@@@


### My First Profunctor

[![If you really hate someone, teach them to recognise Profunctors.](https://www.fpcomplete.com/media/69c95475-5e55-4573-b213-c1e78fb81219.png)](http://xkcd.com/1015/)

This is where I recognised my first `Profunctor`:

``` haskell
data Limits a = Limits
    { step ∷ a → (a, a)
    , check ∷ a → a → Bool } 
```

This was part of the user-facing code we used in production that lets the user adjust various parameters: she can either click an up/down button—or supply a new value directly. The `check` function then validates the new input with respect to the old.

If we generalise over the positive and negative argument positions,

``` active haskell
{-# LANGUAGE RecordWildCards #-}
{-# LANGUAGE UnicodeSyntax #-}
import Control.Arrow
import Data.Function
import Data.Profunctor
-- show
type Limits a = Limits' a a
data Limits' a b = Limits
    { step ∷ a → (b, b)
    , check ∷ a → a → Bool } 

instance Profunctor Limits' where
    dimap g h Limits {..} = Limits
        { step = (h *** h) . step . g
        , check = check `on` g }

maybeLimit ∷ a → Limits a → Limits (Maybe a)
maybeLimit d = dimap (maybe d id) Just

millionsLimit ∷ Limits Double → Limits Double
millionsLimit = dimap (1.0e6 *) (/ 1.0e6)
-- /show
main = return ()
```


### Example: Containers with Keys

Consider the [plethora of \*WithKey functions](https://www.fpcomplete.com/hoogle?q=WithKey) one comes across when working with various containers, for example:

``` haskell
Map.map        ∷     (a → b) → Map i a → Map i b
Map.mapWithKey ∷ (i → a → b) → Map i a → Map i b
```

Can we unify the two functions above?

```
<lambdabot> The answer is: Yes! Profunctors can do that.
```

The [`Control.Lens.Indexed`][] module provides the [`Indexed`][] [`Profunctor`][]:

``` haskell
newtype Indexed i a b = Indexed { runIndexed ∷ i → a → b }
```


#### Exercise: `instance Profunctor (Indexed i)`

@@@
``` active haskell
{-# LANGUAGE UnicodeSyntax #-}
import Data.Profunctor

newtype Indexed i a b = Indexed { runIndexed ∷ i → a → b }
-- show
instance Profunctor (Indexed i) where
    dimap g h (Indexed f) = Indexed (dimap g h . f)
-- /show
main = return ()
```
@@@

Together with the [`Indexable`] class,

``` active haskell
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE UnicodeSyntax #-}

newtype Indexed i a b = Indexed { runIndexed ∷ i → a → b }
-- show
class Indexable i p where
    indexed ∷ p a b → i → a → b

instance Indexable i (Indexed i) where indexed = runIndexed
instance Indexable i (→)         where indexed = const
-- /show
main = return ()
```

we can now give a unified story for `Map.map` and `Map.mapWithKey`:

``` haskell
mapIndexable ∷ Indexable i p ⇒ p a b → Map i a → Map i b
mapIndexable ∷         Indexed i a b → Map i a → Map i b
mapIndexable ∷               (a → b) → Map i a → Map i b
```


#### Exercise: `mapIndexable`

@@@
``` active haskell
{-# LANGUAGE UnicodeSyntax #-}
import Control.Lens.Indexed
import Data.Map as Map

mapIndexable ∷ Indexable i p ⇒ p a b → Map i a → Map i b
-- show
mapIndexable = Map.mapWithKey . indexed
-- /show
main = return ()
```
@@@


## Conclusion

![Profunctors, Profunctors everywhere.](https://www.fpcomplete.com/media/46542da6-1707-43a9-a0f1-a283ab7f592b.jpeg)

The [`UpStar`][] and [`DownStar`][] [`Profunctor`][]s are also worth investigating, as are the concepts of [`Strong`][] and [`Choice`][]. Homework for the reader… or let me know if you want me to extend this tutorial. :) Other comments are welcome too &c. &c. 
[![Reddit](http://reddit.com/static/spreddit2.gif)](http://www.reddit.com/r/haskell/comments/1bg21c/so_i_accidentally_a_profunctors_tutorial/ "Reddit comments") [![G+](http://www.gstatic.com/images/icons/gplus-16.png)](https://plus.google.com/100604691747126657305/posts/Q5Wyc6WaVBa "G+ thread")


## Further Reading

* [How to abstract over a “back and forth” transformation?](http://stackoverflow.com/a/15235409)
* [Profunctors in Haskell](http://blog.sigfpe.com/2011/07/profunctors-in-haskell.html)
  (Warning: [abstract nonsense](http://en.wikipedia.org/wiki/Abstract_nonsense))
* [Functor](http://en.wikipedia.org/wiki/Functor) on Wikipedia with sections on [bifunctors](http://en.wikipedia.org/wiki/Functor#Bifunctors_and_multifunctors) and [covariance versus contravariance](http://en.wikipedia.org/wiki/Functor#Covariance_and_contravariance "In contrast to the OO-normative article labelled ‘computer science’.")
* [Profunctor](http://en.wikipedia.org/wiki/Profunctor) on Wikipedia


[`Data.Functor.Contravariant`]: http://hackage.haskell.org/packages/archive/contravariant/latest/doc/html/Data-Functor-Contravariant.html
[`Contravariant`]: http://hackage.haskell.org/packages/archive/contravariant/latest/doc/html/Data-Functor-Contravariant.html#t:Contravariant
[`Predicate`]: http://hackage.haskell.org/packages/archive/contravariant/latest/doc/html/Data-Functor-Contravariant.html#t:Predicate

[`Bifunctor`]: http://hackage.haskell.org/packages/archive/bifunctors/3.2.0.1/doc/html/Data-Bifunctor.html#t:Bifunctor
[`Biapplicative`]: http://hackage.haskell.org/packages/archive/bifunctors/3.2.0.1/doc/html/Data-Biapplicative.html#t:Biapplicative
[`Bifoldable`]: http://hackage.haskell.org/packages/archive/bifunctors/3.2.0.1/doc/html/Data-Bifoldable.html#t:Bifoldable
[`Bitraversable`]: http://hackage.haskell.org/packages/archive/bifunctors/3.2.0.1/doc/html/Data-Bitraversable.html#t:Bitraversable
[`Clown`]: http://hackage.haskell.org/packages/archive/bifunctors/3.2.0.1/doc/html/Data-Bifunctor-Clown.html#t:Clown
[`Joker`]: http://hackage.haskell.org/packages/archive/bifunctors/3.2.0.1/doc/html/Data-Bifunctor-Joker.html#t:Joker

[`Profunctor`]: http://hackage.haskell.org/packages/archive/profunctors/3.3.0.1/doc/html/Data-Profunctor.html#t:Profunctor
[`Strong`]: http://hackage.haskell.org/packages/archive/profunctors/3.3.0.1/doc/html/Data-Profunctor.html#t:Strong
[`Choice`]: http://hackage.haskell.org/packages/archive/profunctors/3.3.0.1/doc/html/Data-Profunctor.html#t:Choice
[`UpStar`]: http://hackage.haskell.org/packages/archive/profunctors/3.3.0.1/doc/html/Data-Profunctor.html#t:UpStar
[`DownStar`]: http://hackage.haskell.org/packages/archive/profunctors/3.3.0.1/doc/html/Data-Profunctor.html#t:DownStar


[`Control.Lens.Indexed`]: http://hackage.haskell.org/packages/archive/lens/latest/doc/html/Control-Lens-Indexed.html
[`Indexable`]: http://hackage.haskell.org/packages/archive/lens/latest/doc/html/Control-Lens-Indexed.html#t:Indexable
[`Indexed`]: http://hackage.haskell.org/packages/archive/lens/latest/doc/html/Control-Lens-Indexed.html#t:Indexed
