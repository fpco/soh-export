# Part 4: Category and Arrow

# Proxy forms a Category

If you've followed along with `pipes` at all, you'll know that `Proxy` forms a Category. In fact, it forms several. I chose to give `Proxy` such a definition as to capture the most commonly used form of composition and identity:

```haskell

idProxy :: Monad m => Proxy r m '(a,b) '(a,b)
idProxy = Proxy $ Consuming $ foreverK $ lift . yield >=> yield

(=$=) :: Monad m =>
  Proxy r m '(a,a') '(b,b') -> Proxy r m '(b,b') '(c,c') -> Proxy r m '(a,a') '(c,c')
Proxy proxyl =$= Proxy proxyr = undefined -- as defined previously

-- not yet valid Haskell, needs ghc-7.8 kind-polymorphic Category
instance (Monad m) => Cateogry (Proxy r m) where
  id = idProxy
  (.) = flip (=$=)
```

Proxies obey the category laws. Proxy composition is associative, and idProxy is both the left- and right-identity of composition.

```haskell
p1 =$= (p2 =$= p3)  ===  (p1 =$= p2) =$= p3
idProxy =$= p  ===  p  ===  p =$= idproxy
```

# Consuming forms another Category

Back to just looking at one interface at a time, notice that `Consuming` also forms a category. If we take two computations that suspend on *similar* interfaces, then we can fuse one end of each interface to create a new interface.

```haskell
echo :: Monad m => Consuming r m a a
echo = Consuming $ foreverK yield

fuse :: Monad m => Consuming r m a b -> Consuming r m b c -> Consuming r m a c
fuse p1 p2 = Consuming $ \a -> lift (resume (provide p1 a)) >>= \s1 -> case s1 of
    Done r -> return r
    Produced (b :: b) p1' -> lift (resume (provide p2 b)) >>= \s2 -> case s2 of
      Done r -> return r
      Produced (c :: c) p2' -> yield c >>= provide (fuse p1' p2')

instance (Monad m) => Category (Consuming r m) where
  id = echo
  (.) = flip fuse
```

This form of composition is much different than proxy composition. Remember, an interface suspends and resumes with a single value. We are simply tying the "suspend" end of one to the "resume" end of the other. When we send this composed computation the "resume" signal, internally the first one will churn for a while, and then it will pass control to the second, who will finally return control to us. Contrast this with proxy composition, where a proxy might interact several times on its upstream interface before it finally decides to interact with its downstream interface.

Though different, `Consuming` composition also obeys category laws.

```haskell
fuse p1 (fuse p2 p3)  ===  fuse (fuse p1 p2) p3
fuse echo p1  ===  p1  ===  fuse p1 echo
```

# Consuming instance for Arrow

In addition, `Consuming r a` can implement some `Arrow` classes.

```haskell
instance (Monad m) => Arrow (Consuming r m) where
  arr f = Consuming $ foreverK (yield . f)
  
  -- Consuming r m b c -> Consuming r m (b, d) (c, d)
  first p = Consuming $ \(b, d) -> lift (resume (provide p b)) >>= \s -> case s of
    Done r -> return r
    Produced c p' -> yield (c, d) >>= provide (first p')

instance (Monad m) => ArrowChoice (Consuming r m) where
  -- Consuming r m b c -> Consuming r m (Either b d) (Either c d)
  left p = Consuming go where
    go = \e -> case e of
      Right d -> yield (Right d) >>= go
      Left b -> lift (resume (provide p b)) >>= \s -> case s of
        Done r -> return r
        Produced c p' -> yield (Left c) >>= provide (left p')

-- Not so sure about this one.
instance (Monad m) => ArrowApply (Consuming r m) where
  -- Consuming r m (Consuming r m b c, b) c
  app = Consuming go where
    go (p, b) = lift (resume (provide p b)) >>= \s -> case s of
      Done r -> return r
      Produced c _ -> yield c >>= go
      -- It makes me sad to just forget the continuation

instance (Monad m, Monoid r) => ArrowZero (Consuming r m) where
  -- the Consuming that ends immediately
  zeroArrow = Consuming $ \_ -> return mempty

instance (Monad m, Monoid r) => ArrowPlus (Consuming r m) where
  -- Consuming concatenation: p2 takes over once p1 ends
  p1 <+> p2 = Consuming $ \b -> lift (resume (provide p1 b)) >>= \s -> case s of
    Done r -> liftM (r <>) (provide p2 b)
    Produced c p1' -> yield c >>= provide (p1' <+> p2)


-- The only one I haven't managed yet is ArrowLoop
-- Can you write the instance?
instance (MonadFix m) => ArrowLoop (Consuming r m) where
  -- Consuming r m (b, d) (c, d) -> Consuming r m b c
  loop p = ???
```

Notice how each of these follows a similar pattern. `lift (resume (provide ...))`, bind, pattern match on the state, return r, or else `yield something >>= (recurse)`.

I haven't checked the laws yet on these, but I'm feeling pretty good about the instances, except for ArrowApply.