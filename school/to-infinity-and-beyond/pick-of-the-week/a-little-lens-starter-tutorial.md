# A Little Lens Starter Tutorial

# What is lens?

`lens` is a package which provides the type synonym `Lens` which is one of a few implementations of the concept of lenses, or functional references. `lens` also provides a number of generalizations of lenses including `Prism`s, `Traversal`s, `Iso`s, and `Fold`s.

# Why do I care?

Lenses and their cousins are a way of declaring how to focus deeply into a complex structure. They form a combinator library with sensible, general behavior for things like composition, failure, multiplicity, transformation, and representation.

Lenses let you apply some of the power of libraries like `hxt` to all data types in all programs that you write. As this comparison suggests, the initial steps might be very complicated, but, unlike `hxt`, lenses are ubiquitous enough to make learning the complexity pay off over time.

Eventually, you can use lenses to bring things like record syntax, lookups in lists or maps, pattern matching, `Data.Traversable`, scrap your boilerplate, the `newtype` library, and all kinds of type isomorphisms all together under the same mental model.

The end result is a way to efficiently construct and manipulate "methods of poking around inside things" as simple first-class values.

# Okay, what is a Lens?

A lens is a first-class reference to a subpart of some data type. For instance, we have `_1` which is the lens that "focuses on" the first element of a pair. Given a lens there are essentially three things you might want to do

1. View the subpart
2. Modify the whole by changing the subpart
3. Combine this lens with another lens to look even deeper

The first and the second give rise to the idea that lenses are getters and setters like you might have on an object. This intuition is often morally correct and it helps to explain the lens laws.

# The lens laws?

Yep. Like the monad laws, these are expectations you should have about lenses. Lenses that violate them are *weird*. Here they are

1. Get-Put: If you modify something by changing its subpart to exactly what it was before... then nothing happens
2. Put-Get: If you modify something by inserting a particular subpart and then viewing the result... you'll get back exactly that subpart
3. Put-Put: If you modify something by inserting a particular subpart `a`, and then modify it again inserting a different subpart `b`... it's exactly as if you only did the second step.

Lenses that follow these laws are called "very well-behaved".

# What does it look like to use a lens?

Well, let's look at `_1` again, the lens focusing on the first part of a tuple. We view the first part of the tuple using `view`

    >>> view _1 ("goal", "chaff")
    "goal"

    >>> forall $ \tuple -> view _1 tuple == fst tuple
    True

We modify `_1`'s focused subpart by using `over` (mnemonic: we're mapping our modification "over" the focal point of `_1`).

    >>> over _1 (++ "!!!") ("goal", "the crowd goes wild")
    ("goal!!!", "the crowd goes wild")

    >>> forall $ \tuple -> over _1 f tuple == (\(fst, snd) -> (f fst, snd)) tuple
    True

As a special case of modification, we can set the subpart to be a new subpart. This is called `set`

    >>> set _1 "set" ("game", "match")
    ("set", "match")

    >>> forall $ \tuple -> set _1 x tuple = over _1 (const x) tuple
    True

## (Sidenote: What is that `forall` thing?)

It's actually a lie, sorry about that. It's meant to be read as a sentence, a statement of fact, like "forall tuples `tuple`, `view _1 tuple == fst tuple`". You can simulate this behavior by using [QuickCheck's `quickCheck` function](http://hackage.haskell.org/package/QuickCheck-2.6/docs/Test-QuickCheck.html) but `forall` is much stronger.

## Any more examples?

Yeah, here are the lens laws again written using actual code. Below, `l` is any "very well-behaved" lens.

    -- Get-Put
    >>> forall $ \whole -> set l (view l whole) whole == whole
    True

    -- Put-Get
    >>> forall $ \whole part -> view l (set l part whole) == part
    True

    -- Put-Put
    >>> forall $ \whole part1 part2 -> set l part2 (set l part1 whole) = set l part2 whole

# What are some large scale, practical examples of lens?

Don't expect to be able to read the code yet, but here's an example from `lens-aeson` which queries and modifies JSON data.

    -- Returns all of the major versions of an 
    -- array of JSON objects.
    someString ^.. _JSON        -- a parser/printer prism
                 . _Array       -- another prism
                 . traverse     -- a traversal (using Data.Traversable on Aeson's Vector)
                 . _Object      -- another another prism
                 . ix "version" -- a traversal across a "map-like thing"
                 . _1           -- a lens into a tuple (major, minor, patch)

    -- Increments all of the versions above
    someString & _JSON
               . _Array
               . traverse
               . _Object
               . ix "version"
               . _1
               %~ succ          -- apply a function to our deeply focused lens

    -- We can factor out the lens
    allVersions :: Traversal' ByteString Int
    allVersions = _JSON . _Array . traverse . _Object . ix "version" . _1

    -- and then rewrite the two examples quickly

    someString ^.. allVersions
    someString & allVersions %~ succ


    -- Because lenses, prisms, traversals, are all first class in Haskell!

# Wait a second, GHCi is telling me the types of these things are absurd!

Yeah, sorry about that. "It'll make sense eventually", but the types start out tricky.

Try to look at the type synonyms only. We can use `:info` to make sure that GHCi tells us the type synonym instead of the really funky fully expanded types.

I'd show you `_1` but it's not a great example of an understandable type. It uses some weird extra machinery. Instead, how about an example.

    {-# LANGUAGE TemplateHaskell #-}

    type Degrees = Double
    type Latitude = Degrees
    type Longitude = Degrees

    data Meetup = Meetup { _name :: String, _location :: (Latitude, Longitude) }
    makeLenses ''Meetup

Let's assume we have lenses `name` and `location` which focus on the slots `_name` and `_location` respectively. The underscores are a convention only, but you see them a lot because the Template Haskell magic going on in `makeLenses` will automatically make `name` and `location` if use underscores like that.

The type of these lenses is 

    >>> :info name
    name :: Lens Meetup Meetup String String

    >>> :info location
    location  :: Lens Meetup Meetup (Latitude, Longitude) (Latitude, Longitude)

    >>> :type location
    location  :: Functor f => ((Latitude, Longitude) -> f (Latitude, Longitude)) -> (Meetup -> f Meetup)

    -- whoops! Ignore that for now please

# Four type parameters? Isn't that a bit much?

You're right, we'll use the simplified forms for now. This is highly recommended until you get the hang of things. The simplified types are appended with apostrophes. Now the types of `name` and `location` are

    name :: Lens' Meetup String
    location  :: Lens' Meetup (Latitude, Longitude)

These types tell us that, for instance, the `name` lens focuses *from* a `Meetup` and *to* a `String`. Generally we write `Lens' s a` and throughout the documentation `s` and `t` tend to be where a lens is focusing *from* and `a` or `b` tend to be where the lens is focusing *to*.

# Okay, that makes sense. Didn't you say we can compose lenses?

Yup, if we've got a `Lens' s x` and another lens `Lens' x a` we can stick them together to get `Lens' s a`. Strangely, we just use regular old `(.)` to do it.

    la :: Lens' s x
    lb :: Lens' x a
    
    la . lb :: Lens' s a

Or, more concretely.

    meetupLat = location . _1 :: Lens' Meetup Latitude
    meetupLon = location . _2 :: Lens' Meetup Longitude

# Lenses compose backwards. Can't we make (.) behave like functions?

You're right, we could. We don't for various reasons, but the intuition is right. Lenses should combine just like functions. One thing that's important about that is `id` can either pre- or post- compose with any lens without affecting it.

    forall $ \lens -> lens . id == lens
    forall $ \lens -> id . lens == lens

(N.B. If you're categorically inclined, lenses-with-apostrophes would form a `Category` if we did somehow reverse the composition order---`flip (.)` would do it. The instance still cannot be made without a newtype which `lens` trades off for uses mentioned below.)

## That's still pretty annoying

It's true. On the bright side, lenses often feel a whole lot like OO-style slot access like `Person.name`. The reversed composition thing can be thought of as punning on that.

    >>> Meetup { ... } ^. location . _1
    80.3 :: Latitude

Furthermore, we can do composition using a convenient syntax without using `import Prelude hiding ...`.

# What is that (^.)?

Oh, it's just `view` written infix. Here's the Put-Get law again

    >>> forall $ \lens whole part -> (set lens part whole) ^. lens == part
    True

## Actually there are a whole lot of operators in lens

Yup, some people find them convenient.

# Actually there are a WHOLE LOT of operators in lens---over 100

Very convenient! But that *is* a lot. To make it bearable, there are some tricks for remembering them.

1. Operators that begin with `^` are kinds of `view`s. The only example we've seen so far is `(^.)` which is... well, it's just `view` exactly.
2. Operators that end with `~` are like `over` or `set`. In fact, `(.~) == set` and `(%~)` is `over`.
3. Operators that have `.` in them are usually somehow "basic"
4. Operators that have `%` in them usually take functions
5. Operators that have `=` in them are just like their cousins where `=` is replaced by `~`, but instead of taking the whole object as an argument, they apply their modifications in a `State` monad.

## Is that really worth it?

Maybe. Who knows!

If you don't like them, then all of the operators have regular named functions as well.

## ... Some examples would be nice

Ok.

    (.~) :: Lens' s a -> a        -> (s -> s)
    (.=) :: Lens' s a -> a        -> State s ()
    (%~) :: Lens' s a -> (a -> a) -> (s -> s)
    (%=) :: Lens' s a -> (a -> a) -> State s ()

Sometimes we get new operators by augmenting tried and true operators like `(&&)` with `~` and `=`

    (&&~) :: Lens' s Bool -> Bool -> (s -> s)
    (&&=) :: Lens' s Bool -> Bool -> State s ()

    lens &&~ bool = lens %~ (bool &&)
    lens &&= bool = lens %= (bool &&)

    (<>~) :: Monoid m => Lens' s m -> m -> (s -> s)
    lens <>~ m = lens %~ (m <>)

## What about combinators with (^)? Do they show up anywhere else?

Yes, but we have to talk about `Prism`s and `Traversal`s first.

# Okay. What are Prisms?

They're like lenses for sum types.

## What does that mean?

Well, what happens if we try to make lenses for a sum type?

    -- This doesn't work... or exist
    _left :: Lens' (Either a b) a

    >>> view _left (Left ())
    ()                           -- okay, I buy that

    >>> view _left (Right ())
    error!                       -- oh, there's no subpart there

`Prism`s are kind of like `Lens`es that can fail or miss.

## So we use Maybe instead, right?

We could try that.

    _left :: Lens' (Either a b) (Maybe a)

    >>> view _left (Left ())
    Just ()

    >>> view _left (Right ())
    Nothing

But it doesn't compose well.

    _left . name :: Lens (Either Meetup Meetup) ???   -- String? Maybe String?

We'd need a `Lens` that looks into `Maybe`:

    _just :: Lens' (Maybe a) (Maybe a)
    _just = id

Oh. That doesn't get us anywhere. Let's start over.

# Okay. What are Prisms? (Take Two)

Prisms are the duals of lenses. While lenses pick apart some subpart of a product type like a tuple, prisms go down one branch of a sum type like `Either`... Or else they fail.

## Right, a Lens splits out one subpart of a whole. A Prism takes one subbranch.

Exactly!

Think of a product type as being made of both a subpart and a "whole with a hole in it" where the subpart used to go.

    product           =       (a, b)
    subpart           =        a
    whole-with-a-hole = \x -> (x, b)

Whenever we can do that, we can make a lens that focuses just on that subpart.

## A sum type can be broken into some particular subbranch and all-the-other-ones

That's how prisms are dual to lenses. They select just one branch to go down.

    preview :: Prism' s a -> s -> Maybe a
    
Which also lets us "go back up" that one branch.

    review :: Prism' s a -> a -> s

For instance

    _Left :: Prism' (Either a b) a

    >>> preview _Left (Left "hi")
    Just "hi"

    >>> preview _Left (Right "hi")
    Nothing

    >>> review _Left "hi"
    Left "hi"

    _Just :: Prism' (Maybe a) a

    >>> preview _Just (Just "hi")
    Just "hi"

    >>> review _Just "hi"
    Just "hi"

    >>> Left "hi" ^? _Left
    Just "hi"

## Oh, there's another (^)-like operator!

Yep, `(^?)` is like `(^.)` for `Prism'`s.

# Are there any other interesting Prisms? [Printer/Parsers]

Here's a simple one. We can deconstruct `[]`s using `Prism'`s. 

    _Cons :: Prism' [a] (a, [a])

    >>> preview _Cons []
    Nothing

    >>> preview _Cons [1,2,3]
    Just (1, [2,3])

    _Nil :: Prism' [a] ()
   
    >>> preview _Nil []
    Just ()

    >>> preview _Nil [1,2,3]
    Nothing

Here's a strange one. `String` is a very strange sum type.

## No it isn't. It's just a List again.

But `Char` can be thought of as the sum of all characters. And if we "flatten" `Char` into `[]` we can think of `String` as being the "sum of all possible strings"

    data String = "a" | "b" | "c" | "europe" | "curry" | "dosa" | "applejacks" ...

## Okay, sure. Is that important?

What if we make `Prism'`s that focus on particular branches of `String`?

    _ParseTrue  :: Prism' String Bool
    _ParseFalse :: Prism' String Bool

    >>> preview _ParseTrue "True"
    Just True

    >>> preview _ParseFalse "True"
    Nothing

    >>> review _ParseTrue True
    "True"

This is how printer-parsers can sometimes be valid Prisms.

# I think I understand Prisms now. What are Traversals?

Traversals are Lenses which focus on multiple targets simultaneously. We actually don't know how many targets they might be focusing on: it could be exactly 1 (like a Lens) or maybe 0 (like a Prism) or 300.

A very simple `Traversal'` traverses all of the items in a list.

    items :: Traversal' [a] a
    items = traverse               -- from Data.Traversable, actually
                                   -- this is our first implementation of anything in `lens`
                                   -- but don't worry about it right now

In fact, `Traversal'`s are intimately related to lists since if we have a list we also may have either 1, or 0, or many elements. That's one way to `view` a `Traversal`.

    toListOf items :: [a] -> [a]
    
    >>> toListOf items [1,2,3]
    [1,2,3]

## That's pretty boring.

It is. We can define `items` for any `Traversable` though using the exact same definition.

    items :: Traversable t => Traversal' (t a) a
    items = traverse

    flatten :: Tree a -> [a]
    flatten = toListOf items

## That's no big deal, though. It's just another way to use Data.Traversable

For now, but `Traversal` will eventually show a close relationship with `Lens` and `Prism`.

## Do we have another (^)-like operator for Traversals at least?

Yep! It's just `toListOf`.

    >>> [1,2,3] ^.. items
    [1,2,3]

# What else can we do with Traversals? [How do they relate to Prisms/Lenses?]

We can get just the first target.

    firstOf :: Traversal' s a -> Maybe a
    firstOf items :: Traversable t => t a -> Maybe a

Or the last target

    lastOf :: Traversal' s a -> Maybe a
    
    >>> lastOf items [1,2,3] 
    Just 3

## Why does it use Maybe?

The `Traversal` could have no targets.

## Is this like `preview`?

Yeah, `firstOf` is `preview`.

## But I thought preview was preview---and that it was specialized to Prisms?

Nope, `preview` just handles access that focuses on either 0 or 1 targets.

## Can I use view on Traversals too?

Actually, yes, but you might be in for a surprise.

    >>> view items "hello world"
    
    <interactive>:56:6:
        No instance for (Data.Monoid.Monoid Char)
          arising from a use of `items'
        Possible fix:
          add an instance declaration for (Data.Monoid.Monoid Char)
        In the first argument of `view', namely `items'
        In the expression: view items "hello world"
        In an equation for `it': it = view items "hello world"

## What is Monoid doing here?

It represents the notion of failure. It's just like that `Maybe` we tried to use earlier when investigating `Prism`.

Moreover its a way to handle the "0, 1, or many" nature of the targets of a `Traversal`. If you think of `toListOf` as the canonical way to view a `Traversal`, then `Monoid` is a canonical way to compress lists into single values.

## So we can use view if our targets are Monoids?

Yep. What do you think `["hello ", "world"] ^. items` gets us?

## The Monoid product of the items---"hello world"

Exactly! What about `[] ^. items`?

## An ambiguous type error!

Okay, yes. What about `[] ^. items :: String`?

## The empty string, ""

And `[] ^. items :: Monoid m => m`?

## Whatever mempty is for m

Bingo.

# So, Traversals generalize Prisms and Lenses somehow?

Yeah. It works because `Prism`, `Lens`, and `Traversal` are all just type synonyms---Those scary types that GHCi sometimes prints out actually do line up. That's a big reason why they're left in.

# Is this subtyping?

Kind of! That's what [the scary chart on the lens Hackage page](http://hackage.haskell.org/package/lens) shows.

## But Haskell doesn't have subtyping

It's a big clever hack, really. Without it there'd be completely separate combinators for `Lens`, `Prism`, `Traversal`, and all the other things in `lens` that we haven't talked about yet... even though they all end up with identical code at their core.

That's not even the best part yet.

# What's the best part about the subtyping hack?

We can compose `Lens`es and `Prism`s and `Traversal`s all together and the types will "just work out". Admittedly at this point all that means is that compositions of Lenses are Lenses, compositions of Prisms are Prisms, and everything else is a Traversal.

But there are other things in this subtyping hierarchy, like `Iso`s.

# What are Isos?

Isos are isomorphisms, connections between types that are equivalent in every way.

More concretely, an Iso is a forward mapping and a backward mapping such that the forward mapping inverts the backward mapping and the backward mapping inverts the forward mapping.

    fw . bw = id
    bw . fw = id

(N.B. This is again exactly a `Category` isomorphism.)

## Isos are in the lens subtyping hierarchy?

Yep! They are more primitive than either `Lens` or `Prism`.

If we think of a `Lens' s a` as a type which splits the product type `s` into its subpart `a` and the context of that subpart, `a -> s`, an `Iso` is a trivial lens where the context is just `id`.

If we think of a `Prism' s a` as a type which splits the sum type `s` into a privileged branch and "all the other" branches then an `Iso` is a trivial `Prism` where there's just one branch and we're privileging exactly it.

## How do Isos compose with Lenses and Prisms?

They can compose on either end of a lens or a prism and the result is a lens or a prism respectively. Isos are "iso" (unchanging) because no matter where you put them in a lens pipeline they leave things as they were.

# What are some example Isos?

How about between `Maybe` and `Either ()`?

    someIso :: Iso' (Maybe a) (Either () a)
    
    >>> Just "hi" ^. someIso
    Right "hi"

That's a little boring. How about between strict and lazy `ByteString`s?

    strict :: Iso' Lazy.ByteString Strict.ByteString

    >>> "foobar" ^. strict
    "foobar"

    >>> "foobar" ^. strict . from strict
    "foobar"

Also kind of boring. [How about between an XML tree treating element names as strings and an XML tree treating element names as Qualified Names](http://hackage.haskell.org/package/hexpat-lens-0.0.7/docs/Text-XML-Expat-Lens-Names.html)?

    qualified :: Iso' (Node String String) (Node (QName String) String)

    >>> Element "foaf:Person" [("foaf:name", "Ernest")] [] ^. qualified

    Element (QName { qnPrefix = Just "foaf"
                   , qnLocalPart = "Person"
                   }) 
            [ (QName { qnPrefix = Just "foaf"
                     , qnLocalPart = "name"
                     }
              ,"Ernest")
            ] 
            []

    >>> Element "foaf:Person" [("foaf:name", "Ernest")] [] ^? qualified 
                                                            . attributes 
                                                            . _head . _1 . localPart
    Just "name"

# What is "from" doing there? [Iso Laws]

`from` "turns an iso around". Since `Iso`s are equivalences they're symmetric and we can treat them as maps in both directions between types. In fact, the `Iso` laws stated above are just that for any iso, `iso`
    
    view (iso . from iso) == id
    view (from iso . iso) == id

## That's pretty cool

`Iso`s are pretty cool. They're one of my favorite parts of `lens`.


# What if I have any more questions?

This list may grow over time. Please ask them and leave any comments.

But for now, go get yourself a peanut butter and marmalade sandwich.

---

With help from Reddit users penguinland, kqr, markus1189, MitchellSalad, DR6, NruJaC, and rwbarton.