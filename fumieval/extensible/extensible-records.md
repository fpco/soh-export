# Extensible Records Explained

[extensible](https://hackage.haskell.org/package/extensible) is a library for extensible data types. It provides extensible __product__ and __sum__ types parameterized by type-level lists.

This tutorial introduces extensible records, an application of the extensible products.

First, we pass a space-separated list of field names to `mkField`:

``` haskell
{-# LANGUAGE TemplateHaskell, DataKinds, TypeOperators, FlexibleContexts #-}
{-# OPTIONS_GHC -fno-warn-unticked-promoted-constructors #-}

import Data.Extensible
import Control.Lens hiding ((:>))

mkField "name collective cry"
```

Template Haskell will generate values that represent field names. Don't worry about the atrocious type.

``` haskell
$ stack ghci --package lens --package extensible
> :load tutorial.hs
...
> :t name
name
  :: forall (kind :: BOX)
            (f :: * -> *)
            (p :: * -> * -> *)
            (t :: (Assoc GHC.TypeLits.Symbol kind -> *)
                  -> [Assoc GHC.TypeLits.Symbol kind] -> *)
            (xs :: [Assoc GHC.TypeLits.Symbol kind])
            (h :: kind -> *)
            (v :: kind)
            (n :: Data.Extensible.Internal.Nat).
     (Labelling "name" p, Wrapper h,
      Data.Extensible.Internal.KnownPosition n, Extensible f p t,
      Elaborate "name" (FindAssoc "name" xs) ~ 'Expecting (n ':> v)) =>
     Data.Extensible.Internal.Rig.Optic' p f (t (Field h) xs) (Repr h v)
```

Now we define a data type that contains a name of an animal, its collective noun, and an onomatopoeia if it exists.

```
type Animal = Record
  [ "name" :> String
  , "collective" :> String
  , "cry" :> Maybe String
  ]
```

`Record` is a type of record that takes a list of pairs of field name (which has a kind `Symbol`) and value type (`*`).

```
Record :: [Assoc Symbol *] -> *
(:>) :: Symbol -> * -> Assoc Symbol *
```

Let there be a dove and a swan.
Construction is similar to a normal list. `Nil` is an empty record, and `(<:)` appends a value.
`@=` annotates a value by the field name.

``` haskell
-- (@=) :: FieldName k -> v -> Field Identity (k :> v)

-- infix 1 @=

dove :: Animal
dove = name @= "dove"
  <: collective @= "dule"
  <: cry @= Just "coo"
  <: emptyRecord

swan :: Animal
swan = name @= "swan"
  <: collective @= "lamentation"
  <: cry @= Nothing
  <: emptyRecord
```

The field names we've defined can be used as lenses.

``` haskell
> swan ^. name
"swan"
```

We can update fields with the lens operators. A group of swans on the ground is called a __bank__; let's apply this.

``` haskell
> swan & collective .~ "bank"
name @= "swan" <: collective @= "bank" <: cry @= Nothing <: Nil
```

Now we can define a function that takes an `Animal` and makes a phrase.

```haskell
collectiveOf :: Animal -> String
collectiveOf a = unwords ["a", a ^. collective, "of", a ^. name ++ "s"]
```

``` haskell
> collectiveOf dove
"a dule of doves"
> collectiveOf swan
"a lamentation of swans"
```

`collectiveOf` can be generalized as follows:

``` haskell
collectiveOf :: (Associate "name" String s, Associate "collective" String s)
  => Record s -> String
```

The argument no longer has to be `Animal`; this can be used for any records that have `name` and `collective` as `String` fields.

This may look like complex machinery, but the number of constructs needed is just 6: `Record`, `:>`, `mkField`, `@=`, `<:`, `Nil`.

*extensible* has close affinity with *lens* and field names are reusable. It's useful as a replacement for standard haskell records.

Another advantage of extensible records is the ease of creating generic functions. The following example is a `FromJSON` instance of records.
It's much smaller than [the implementation based on GHC.Generics](https://github.com/bos/aeson/blob/8d27887b3ab4e5ba5844d52695adbfdb90456fef/Data/Aeson/Types/FromJSON.hs#L708-#L1039).

``` haskell
{-# LANGUAGE TemplateHaskell, DataKinds, TypeOperators, FlexibleContexts, UndecidableInstances #-}
{-# OPTIONS_GHC -fno-warn-unticked-promoted-constructors #-}

import Data.Extensible
import Control.Lens hiding ((:>))
import Data.Aeson (FromJSON(..), withObject)
import Data.Proxy
import Data.String
import GHC.TypeLits

instance Forall (KeyValue KnownSymbol FromJSON) xs => FromJSON (Record xs) where
  parseJSON = withObject "Object" $ \v -> hgenerateFor (Proxy :: Proxy (KeyValue KnownSymbol FromJSON))
    $ \m -> let k = symbolVal (proxyAssocKey m) in case v ^? ix (fromString k) of
      Just a -> Field . pure <$> parseJSON a
      Nothing -> fail $ "Missing key: " ++ 
```

`Forall` and `hgenerateFor` are the important points. `Forall c xs` is a constraint that every element of `xs` satisfies the constraint `c`. In this case, it requires that every field name is a type level string and every field type is an instance of `FromJSON`.
`hgenerateFor` instantiates those constraints and yields a record.

``` haskell
Forall :: (k -> Constraint) -> [k] -> Constraint
hgenerateFor
  :: (Applicative f, Forall c xs) =>
     proxy c
     -> (forall (x :: k). c x => Membership xs x -> f (Field Identity x))
     -> f (Record xs)
```

`Membership xs x` is the type of a witness that `x` is an element of `xs`. It is used to reify the field name using `symbolVal` and `proxyAssocKey` in the example above. `Field . pure` constructs a field without specifying a concrete field name, unlike `@=`.

```haskell
> decode <$> fromString <$> getLine :: IO (Maybe Animal)
{"name": "zebra", "collective": "dazzle", "cry": "whoop"}
Just (name @= "zebra" <: collective @= "dazzle" <: cry @= Just "whoop" <: nil)
```

Here is another [example of a ToJSON instance](https://github.com/fumieval/extensible/blob/master/examples/aeson.hs).

Using extensible, you should be able to define a generic instance of your own typeclass without pain.