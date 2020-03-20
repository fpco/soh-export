# Mismatched type variables

```haskell active
foo :: (a -> b) -> (b -> c) -> a -> c
foo f =
    bar
  where
    bar :: (b -> c) -> a -> c
    bar = (. f)
    
main = print $ foo (+ 1) (+ 2) 5
```