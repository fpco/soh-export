# Part VI: A Most Significant Comparison

I was going to finally get to the point, but I decided it would be good to consolidate our understanding of "the most significant difference" from the [previous parts](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication/) into a single short summary.

The [groundhog](http://en.wikipedia.org/wiki/Punxsutawney_Phil) has seen his shadow and you are in for six more weeks of winter.

Morton Order Redux
==================

Recall our rather boring definition of a Morton-ordered `Key`.

```active haskell
import Data.Word

-- show
data Key = Key {-# UNPACK #-} !Word {-# UNPACK #-} !Word
-- /show
  deriving (Show, Read, Eq)

main = putStrLn "It typechecks."
```

I want to refactor the trick for comparing in [Morton order](http://en.wikipedia.org/wiki/Z-order_curve) from [Part 2](https://www.fpcomplete.com/tutorial-edit/revisiting-matrix-multiplication/part-2) that we revisited in [Part 4](https://www.fpcomplete.com/tutorial-edit/revisiting-matrix-multiplication/part-4) into a more reusable form.

To that end, let us consider how to compare two unsigned words for how they differ in the placement of their most significant bit.

Logically I want to `on compare msb`, without paying for calculating the position of the most significant bit directly.

To do I first observe that we can first compare our two values `a` and `b`. 

If they match, then trivially they agree on the position of their most significant set bit!

If they don't, then either `a < b` or `b < a`.

Without loss of generality, let's assume `a < b`. Then either `a` had the same `msb` as `b` or it doesn't. 

If `a` had the same `msb` as `b` then `xor a b` will not have that bit set, so `a < xor a b` will be `False` as `a` as a more significant bit set than `xor a b` does.

If `a` does not have the same `msb` as `b`, and `a < b`, then `b` has it set, and `a` does not, so the more significant bit will be set in `xor a b`, and `a < xor a b` will be `True`.

Putting all of this logic together yields the following combinator:

```active haskell
import Data.Bits
import Data.Word

-- show
compares :: Word -> Word -> Ordering
compares a b = case compare a b of
  LT | a < xor a b -> LT
  GT | b < xor a b -> GT
  _ -> EQ
  
main = print $ compares 4 7
-- /show
```

We can similarly reason through specialized scenaros to obtain `<`, `<=`, `==`, `/=`, `>=`, `>` restricted to the most significant bit.

```active haskell
import Data.Bits
import Data.Word

-- show
lts, les, eqs, nes, ges, gts :: Word -> Word -> Bool
lts a b = a < b && a < xor a b
les a b = a <= b || xor a b <= b
-- /show
eqs a b = a >= min b c && b >= max a c where c = xor a b
nes a b = a <  min b c || b <  min a c where c = xor a b
-- show ...
gts a b = a > b && xor a b > b
ges a b = a >= b || a >= xor a b

main = print $ les 4 7
-- /show
```
  
With that we can see our earlier `Ord` instance for `Key` is just:
  
```active haskell
import Data.Bits
import Data.Word

data Key = Key {-# UNPACK #-} !Word {-# UNPACK #-} !Word
  deriving (Show, Read, Eq)

lts :: Word -> Word -> Bool
lts a b = a < b && a < xor a b

-- show
instance Ord Key where
  Key a b `compare` Key c d
    | xor a c `lts` xor b d = compare b d
    | otherwise             = compare a c
-- /show

main = putStrLn "It typechecks."
```

Now that is much easier to read:

If the most significant difference betwen `a` and `c` is less significant than the most significant difference between `b` and `d`, then we should just compare `b` and `d`, otherwise we compare `a` with `c`.

[One of these things is not like the others](http://www.youtube.com/watch?v=FClGhto1vIg&t=11s)
==========================================

This makes it clear why we had three uses of `xor` in the original, the first two were to calculate the differences themselves, while the last `xor` was simply to compare by most significant bit!

Switching to these internally made almost a factor of two difference in the performance of the overall multiplier relative to actually performing the masking! This bodes well for the practicality of the as-yet-still-unbenchmarked `IntMap` alternative described in [part 4](https://www.fpcomplete.com/user/edwardk/revisiting-matrix-multiplication/part-4).

-[Edward Kmett](mailto:ekmett@gmail.com)

August 25 2013

P.S. This also means I effectively just claimed No-Prize #5 for myself. ;)