# Pretext by experiments and guesses

Edward Kmett recently published [a very interesting exploration of modeling Cellular Automata using lenses and comonads](https://www.fpcomplete.com/user/edwardk/cellular-automata). Within it he suggested one might want to peer down the rabbit hole at a very interesting Haskell type, the `Pretext`.

```haskell
newtype Pretext s a = Pretext { 
    runPretext :: forall f. Functor f => (s -> f s) -> f a 
  } deriving Functor
```

In particular, he suggested that it was a total characterization of the `Store` comonad used throughout the rest of that article, and further that it's an enlightening challenge to define the `Comonad` instance for `Pretext`.

To me that was invitation enough to finally figure out what those `Pretext`s are after seeing them far too often in Lens code.

**Warning, spoilers ahead.** By the end of this article I will explain how to write a comonad instance for `Pretext` and how `Pretext` and `Store` are related. It's a little messy and a little neat but will hopefully dramatically simplify the relationship between these two types. Try it yourself first if you don't want the spoilers.


# Store and Experiment

Let's look at `Store` itself again. Its definition is rather understated.

```haskell
data Store s a = Store s (s -> a)

instance Functor (Store s) where
  fmap f (Store s k) = Store s (f . k)
```

It's a function application with a "privileged" point in the domain. This privileged point gives us the Comonadic `extract`.

```haskell
extract :: Store s a -> a
extract (Store s k) = k s
```

we also might want to move that point around to change our focus

```haskell
seek :: (s -> s) -> (Store s a -> Store s a)
seek f (Store s k) = Store (f s) k
```

One intuition for a `Store` comonad is it's a pointer into a particular codomain, a particular strand (or fiber) stretching from the domain to the codomain extended over any number of extra mapping steps stacked atop via `fmap h`. We can always begin these strands in the identity mapping as well.

```haskell
exact :: s -> Store s s
exact s = Store s id
```

Getting to the meat of this article, we also might want to perform an `experiment` on a `Store`, which might be thought of as perturbing our privileged point in a computational context and seeing what final result occurs.

```haskell
experiment :: Functor f => Store s a -> (s -> f s) -> f a
experiment (Store s k) f = k <$> f s
```

This was the core of  using `Store` to model cellular automata—`experiment`ing in the `[]` context allows the `Store`d rule to be stretched across the entire state of the automata. Taken in one light this might be just one of many things you could do with a `Store` comonad, but I'd like to explore the idea that `experiment`ing on `Store`s is using them to the absolute fullest. In particular, I'm going to eventually show that `s` is isomorphic to `experiment s`.


# Some Pretext first

Let's, as [described in Edward's article](https://www.fpcomplete.com/user/edwardk/cellular-automata#a-glimpse-down-the-rabbit-hole) define exactly the result of `experiment`ing on a `Store` as a `Pretext`.

```haskell
newtype Pretext s a = 
  Pretext { runPretext :: runPretext :: forall f. Functor f => (s -> f s) -> f a }

instance Functor (Pretext s) where
  fmap f (Pretext k) = Pretext (fmap f . k)

experiment :: Store s a -> Pretext s a
experiment (Store s k) = Pretext $ \f -> k <$> f s
```

Since the claim is that `Pretext`s are just as powerful as `Store`s we ought to be able to define `instance Comonad (Pretext s)`.

```haskell
instance Comonad (Store s) where
  extract (Store s k) = k s
  duplicate (Store s k) = Store s (\s -> Store s k)

instance Comonad (Pretext s) where
  extract   :: Pretext s a -> a
  extract   (Pretext k) = ???

  duplicate :: Pretext s a -> Pretext s (Pretext s a)
  duplicate (Pretext k) = Pretext $ \(f :: forall f . Functor f => s -> f s) -> ???
```

For each function we become the users of a `Pretext` and must carefully construct functors and injections `inj :: s -> f s` that we give to the `Pretext` in order to get the results we want. Doing this directly is a little bit challenging, so instead we'll examine the correspondence between `Store` and `Pretext` more directly.


# Reverse Engineering a Pretext

Let's analyze `Pretext`s through reverse engineering in the style of [Dan Piponi's *Reverse Engineering Machines*](http://blog.sigfpe.com/2006/11/yoneda-lemma.html). For a particular `s` and a particular `a` we have a machine `m :: (s -> f s) -> f a` that works for any `Functor => f` we can imagine. Is it possible to reverse engineer that machine?

How can the machine possibly work? Since it doesn't know what `Functor` we're going to choose nor how we're going to inject `s` there's not a whole lot that can be going on behind the scenes. Here are some things that can't be happening, for instance.

1. The machine cannot simply contain a secret `f a` and return it like `m = const secret_f_a` since while it does know what `a` is it cannot know which `Functor` we're going to choose ahead of time. 

2. It also can't have a secret `a` and examine the injection we give it to get `f`, since if all it knows about `f` is that it's a `Functor` the machine will have no way to uniformly get its secret `a` inside of `f`: there is no such function `forall f . Functor f => a -> f a`.

Let's look at it from a different angle. Assuming the machine uses what we give it, it almost certainly must apply our injection function to some secret `s` which it's holding on to. Once it's done that it'll have `(our_injector :: s -> f s) (secret_s :: s) :: f s`. Can it get an `f a` from an `f s` regardless of `f`? 

Of course, if it has a function `go :: s -> a` it can always use `fmap go :: f s -> f a`.

By this exploration, we can come to believe that any implementation of `Pretext s a` must secretly contain an `s` and a `s -> a`—which is exactly what we need to build a `Store s a` as well. This suggests that there ought to be a function like this

```haskell
guess :: Pretext s a -> Store s a
```

Can we build it?


# The finer points of Pretext

Since `Store` is just a product type, let's build `guess` one piece at a time. We want to find some sequence of operations with our `Pretext` machine that gets it to hand us its secret `s` and then some other sequence that gets it to give us its secret `s -> a`.

```haskell
guess (Pretext m) = Store (get_s m) (get_s_to_a m)
```

## Store smuggling

What is the type of `get_s`?

```haskell
get_s (m :: forall f . Functor f => (s -> f s) -> f a) :: s
```

Since the return type of `m` is `f a` there's no way we can get `m` to give us its secret `s` directly... we'll need to smuggle it out inside of `f`. For instance, if `f` we `(s, a)` then the return type of `m` would be `(s, a)` and we could just look at the `fst` component to get our needed `s`.

Unfortunately, we can't build an injector `s -> (s, a)` without an `a`, which we don't have. Instead, we'll need a different trick---a `Functor` which actually holds a secret payload while only pretending to hold its real payload.

```haskell
newtype Secret s a = Secret { openOSesame :: s }   -- this is also known as `Const`, btw.

instance Functor (Secret s) where
  fmap _ (Secret s) = Secret s

secret :: s -> Secret s s
secret = Secret
```

This trick functor `Secret` does just what we want. It uses a phantom type to appear to be holding an `a` while actually fully ignoring any `fmap`ping that happens to it in order to just protect its hidden cargo. Finally, the injector function `secret` builds a `Secret s a` without ever needing an actual `a`. What happens when we use this injector on `m`?

```haskell
openOSesame $ (m :: forall f . Functor f => (s -> f s) -> f a) (secret :: s -> Secret s a)
===
openOSesame $ (m :: (s -> Secret s s) -> Secret s a) (secret :: s -> Secret s a)
===
openOSesame $ (m secret :: Secret s a)
===
openOSesame (m secret) :: s    -- ka. boom.
```

Our `Secret` container does the trick and we smuggle the context `s` out of the `Pretext` `m`.

```haskell
get_s m = openOSesame (m secret)
```

## Storing functions

We can use pretty much the same trick to extract the `Pretext`'s `s -> a` map as well. We want a box which ignores one thing while secretly capturing and preserving another. In this case, unlike above, we want to ignore the `s` that gets passed during injection and nab the function which gets passed by `fmap`. Can we do that?

I'll skip the longer exploration this time and note that if we take advantage of the law `f . id == f` we have a very handy way of capturing functions.

```haskell
newtype Capture s a = Capture { release :: s -> a }

instance Functor (Capture s) where
  fmap f (Capture s) = Capture (f . s)   -- append the function to capture it

capture :: s -> Capture s s
capture _ = Capture id                   -- id won't affect our captured function
```

Which lets us define

```haskell
get_s_to_a m = release (m capture)
```

# Guessing the merchandise

Together these pieces build our `guess` function just right while also giving a lot of intuition about what `Pretext` must be doing behind the scenes. If we look back at `experiment` it's a complete description.

```haskell
experiment (Store s k) = Pretext $ \f -> k <$> f s
```

apply the given injector to the `Store`'s `s` and then `fmap` the `Store`'s `k :: s -> a` over the result.

And then `guess` just packs up our smuggling functors to rebuild the `Store`.

```haskell
guess :: Pretext s a -> Store s a
guess (Pretext m) = Store (openOSesame $ m secret) (release $ m capture)
```

With these pieces, we can prove that `guess` and `experiment` are isomorphisms.

```haskell
guess . experiment
===
\store@(Store s k) -> guess . experiment $ store
===
\(Store s k) -> guess $ Pretext $ \f -> k <$> f s
===
\(Store s k) -> (\(Pretext m) -> Store (openOSesame $ m secret) (release $ m capture)) $ Pretext $ \f -> k <$> f s
===
\(Store s k) -> (\m -> Store (openOSesame $ m secret) (release $ m capture)) $ \f -> k <$> f s
===
\(Store s k) -> Store (openOSesame $ (\f -> k <$> f s) secret) (release $ (\f -> k <$> f s) capture)
===
\(Store s k) -> Store (openOSesame (k <$> secret s)) (release (k <$> capture s))
===
\(Store s k) -> Store (openOSesame (k <$> Secret s)) (release (k <$> Capture id))
===
\(Store s k) -> Store (openOSesame (Secret s)) (release (Capture (k . id)))
===
\(Store s k) -> Store s (k . id)
===
\(Store s k) -> Store s k 
===
id
```

Though the other direction is significantly tougher. (The proof, due to [Russell O'Connor, Appendix C](http://arxiv.org/abs/1103.2841v1) relies on generating a [free theorem](http://ttic.uchicago.edu/~dreyer/course/papers/wadler.pdf) from the parametricity of `Pretext`, clearly necessary to bootstrap the meaning it has.)

# Another way of looking at it

One particularly interesting simplification of `guess` comes from noting that `Secret` and `Capture` both represent two sides of the `Store` comonad, the pointer in to the domain and the mapping function respectively. It turns out that given the right injection function and extraction functions, we can use `Store` itself to replace both `Secret` and `Capture`. Here's an alternative definition of `guess`.

```haskell
guess :: Pretext s a -> Store s a
guess (Pretext m) = Store (pos $ m exact) (peek $ m exact)
  where
    pos :: Store s a -> s
    pos (Store s _) = s
    peek :: Store s a -> (s -> a)
    peek (Store _ k) = k
```

This is absolutely the "right" want to see the correspondance between `Pretext` and `Store`.

# So what about Comonads?

I promised at the start that this whole effort would go toward defining `instance Comonad (Pretext s)` but we haven't even looked at that yet. What gives?

Well, given an isomorphism we have a really trivial definition of `Comonad`—we'll just lift it from `Store`.

```haskell
instance Comonad (Store s) where
  extract (Store s k)  = k s
  extend f (Store s k)  = Store s (f . flip Store k)
  duplicate (Store s k) = Store s (flip Store k)

instance Comonad (Pretext s) where
  extract = extract . guess
  extend f = experiment . extend (f . experiment) . guess
  duplicate = experiment . fmap experiment . duplicate . guess
```

But that's unfair, right? Fortunately, this is Haskell and we can just expand our isomorphisms and simplify mechanically using equational reasoning to get a more compact definition.

#### Extract

```haskell
experiment . guess
===
\(Pretext m) -> (\(Store s k) -> k s) $ Store (openOSesame $ m secret) (release $ m capture)
===
\(Pretext m) -> (peek $ m exact) (pos $ m exact)
===
\(Pretext m) -> peek <*> pos $ m exact
```

#### Duplicate

```haskell
experiment . fmap experiment . duplicate . guess
===
\(Pretext m) -> (Pretext $ \f -> (\s -> Pretext $ \f -> (peek $ m exact) <$> f s) <$> f (pos $ m exact))
```

```haskell
instance Comonad (Pretext s) where
  extract (Pretext m)   = peek <*> pos $ m exact
  duplicate (Pretext m) = 
    Pretext $ \f ->     f (pos $ m exact)
                    <&> \s -> Pretext $ \f -> (peek $ m exact) <$> f s
```

Which I'm glad to only ever have to write once.


---

[Comments on Reddit](http://www.reddit.com/r/haskell/comments/1kkoup/pretext_by_experiments_and_guesses/)