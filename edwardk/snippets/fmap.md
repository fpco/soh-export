# The free theorem for fmap

When we write down the definition of `Functor` we carefully state two laws:

1. `fmap id = id`
2. `fmap f . fmap g = fmap (f . g)`

These are pretty well known in the Haskell community.

What is less well known is that the second actually follows from the first and parametricity, so you only need to sit down and prove _one_ `Functor` law when you go to supply a `Functor`!

This is a “folklore” result, which I've used in conversation many times before, but it [continues to surprise](https://twitter.com/BartoszMilewski/status/566204998015787008) folks, so I decided to write up a slow, step by step proof of this result as it is a fun little exercise in equational reasoning.

To prove this we're going to need the [free theorem](http://ttic.uchicago.edu/~dreyer/course/papers/wadler.pdf) for `fmap` and a few lemmas.

> ###Free Theorem:
>
> The free theorem for `fmap :: (a -> b) -> F a -> F b` is that given functions `f`, `g`, `k`, and `h` such that
>
> ```haskell
> g . h = k . f
> ```
> 
> then
>
> ```
> $map g . fmap h = fmap k . $map f
> ```
>
> where `$map` is the "natural map" for the type constructor `F`. 
>
>
> ###Proof:
>
> This is a free theorem, so it holds for _any_ function with the same type signature as
> `fmap`, regardless of implementation.
>
> You can obtain this theorem employing Philip Wadler's [“Theorems for Free”](http://ttic.uchicago.edu/~dreyer/course/papers/wadler.pdf) laboriously by hand as is done by Bartosz in the comments below, but we can also obtain this result just by asking `lambdabot` to do it for us on IRC.
>
> ```
> <edwardk> @free fmap :: (a -> b) -> (F a -> F b)
> <lambdabot> g . h = k . f => $map_F g . fmap h = fmap k . $map_F f
> ```
>
> Thanks, `lambdabot`!

The natural map has the properties we're looking for, so what we need to do is use the above theorem to prove `fmap f = $map f`, and borrow them.

Note: There are some caveats about precisely when such a natural map exists in the comments below, but in any case where `fmap` can be given with `fmap id = id`, it can also exist with this variance.

To do that we start with

> ###Lemma 1:
> 
> Given `fmap id = id`, then
> 
> ```
> fmap f = $map f
> ```
>
> ###Proof:
>
> ```
> fmap f 
> = {- by $map id = id -}
> $map id . fmap f
> = {- by free theorem, using g = k = id, h = f -} 
> fmap id . $map f
> = {- by fmap id = id -}
> $map f
> ```

Now we know that `fmap f = $map f` pointwise, and if we assume functional extensionality, we can even show `fmap = $map`.

**Lemma 1** is sufficient to show that any two definitions `fmap1` and `fmap2` for `fmap` that each satisfy `fmap id = id`, are equivalent up to functional extensionality, as of course `fmap1 f = $map f = fmap2 f`. 

Therefore the observable behavior of `fmap` is uniquely determined.

Next we'll, need another precondition:

> ###Lemma 2:
>
> ```haskell
> f . g = id . (f . g)
> ```
> 
> ###Proof: 
>
> Naively, `id` is the unit for `(.)`. In reality it results in it eta-expanded.
>

Now we're finally ready to proceed to the real proof:

> ###Theorem:
> 
> Given `fmap id = id`, we can show that
>
> ```
> fmap f . fmap g = fmap (f . g)
> ```
> 
> ###Proof:
> 
> We can read this off of the properties of the free theorem several ways.
> 
> The easiest one which does not use the same shaped property on `$map` is to just play with `$map id = id`
> 
> ```haskell
> fmap f . fmap g 
> = {- by lemma 1, fmap f = $map f -}
> $map f . fmap g
> = {- by the free theorem for fmap using lemma 2 for the precondition -}
> fmap id . $map (f . g)
> = {- by fmap id = id -}
> $map (f . g)
> = {- by fmap _ = $map _ -}
> fmap (f . g)
> ```

(We could have also employed the fact that `$map f . $map g = $map (f . g)` more directly.)

There is definitely room to improve this proof. It would be more satisfying to start with
two definitions of `fmap` such that they both satisfy `fmap_1 id = id` and `fmap_2 id = id`, and show that they must be equivalent up to functional extensionality, rather than appeal to the existence of a "natural map", which is a bit hand wavy.  Russell O'Connor [took a similar approach](http://thread.gmane.org/gmane.comp.lang.haskell.libraries/15382/focus=15384) to concisely show that `fmap` is uniquely determined, if it exists, on the `libraries` mailing list.

But there you have it. Next time you go to write a `Functor`, you can rest assured that you only need to prove `fmap id = id`, and you can get the other result for free!

-[Edward Kmett](mailto:ekmett@gmail.com)

February 15, 2015