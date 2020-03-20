# Y Combinator Performance

```active haskell
module Main where
import Data.Time.Clock (diffUTCTime, getCurrentTime)
import qualified Data.MemoCombinators as Memo

-- symple recursion
fac0 :: (Num a, Eq a) => a -> a
fac0 1 = 1
fac0 n = n * fac0 (n-1)

fib0 :: (Num a, Eq a) => a -> a
fib0 0 = 0
fib0 1 = 1
fib0 n = fib0 (n-1) + fib0 (n-2)
```

```active haskell
test0a :: IO()
test0a = print $ map fac0 [1..9]

test0b :: IO()
test0b = print $ map fib0 [1..9]
```

```active haskell
-- Y combinator as fixed point
fac :: (Num a, Eq a) => (a -> a) -> a -> a
fac _ 0 = 1
fac f n = n * f (n-1)

fib :: (Num a, Eq a) => (a -> a) -> a -> a
fib f 0 = 0
fib f 1 = 1
fib f n = f (n-1) + f (n-2)

y :: (a -> a) -> a
y x = x (y x)
```

```active haskell
test1a :: IO()
test1a = print $ map (y fac) [1..9]

test1b :: IO()
test1b = print $ map (y fib) [1..9]
```

```active haskell
-- Y as SLL combinator
s :: (a -> b -> c) -> (a -> b) -> a -> c
s x y z = x z (y z)

k :: a -> b -> a
k x y = x  -- aka const

i :: a -> a
i = s k k -- aka id

c :: (a -> b -> c) -> b -> a -> c
c = s (b b s) (k k) -- aka flip

b :: (b -> c) -> (a -> b) -> a -> c
b = s (k s) k  -- aka (.)

newtype I a = I (I a -> a)

unI :: I a -> I a -> a
unI (I x) = x

m :: I a -> a
m = s unI i

l :: (a -> b) -> I a -> b
l = c b m

y2 :: (a -> a) -> a
y2 = s l (b I l)
```

```active haskell
test2a :: IO()
test2a = print $ map (y2 fac) [1..9]

test2b :: IO()
test2b = print $ map (y2 fib) [1..9]

-- MemoCombinator library - non-standard - fast!
--import qualified Data.MemoCombinators as Memo
-- type fixed?
mfc :: Integer -> Integer
mfc = Memo.integral f where
    f 1 = 1
    f x = x * mfc (x-1)

mfb :: Integer -> Integer
mfb = Memo.integral f where
    f 0 = 0
    f 1 = 1
    f x = mfb (x-1) + mfb (x-2)
```

```active haskell
test3a :: IO()
test3a = print $ map mfc [1..9]

test3b :: IO()
test3b = print $ map mfb [1..9]
```

```active haskell
main :: IO ()
main = do
	t0 <- getCurrentTime
	print $ fib0 32
	t1 <- getCurrentTime
	putStrLn $ show (t1 `diffUTCTime` t0)
	
	t0 <- getCurrentTime
	print $ y fib 32
	t1 <- getCurrentTime
	putStrLn $ show (t1 `diffUTCTime` t0)
	
	t0 <- getCurrentTime
	print $ y2 fib 32
	t1 <- getCurrentTime
	putStrLn $ show (t1 `diffUTCTime` t0)
	return ()

	t0 <- getCurrentTime
	print $ mfb 32
	t1 <- getCurrentTime
	putStrLn $ show (t1 `diffUTCTime` t0)
	
	t0 <- getCurrentTime
	print $ mfb 1024
	t1 <- getCurrentTime
	putStrLn $ show (t1 `diffUTCTime` t0)
	
```


[output]

```
2178309
12.6552654s
2178309
13.5133512s
2178309
13.6733672s
2178309
0s
(39.08 secs, 3610337556 bytes)
```


```
> print $ mfb 1024
4506699633677819813104383235728886049367860596218604830803023149600030645708721396248792609141030396244873266580345011219530209367425581019871067646094200262285202346655868899711089246778413354004103631553925405243
(0.03 secs, 3136396 bytes)
```

source:
http://d.hatena.ne.jp/kazu-yamamoto/20100519/1274240859


```
Sxyz = xz(yz)     -- <*>
Kxy = x               -- const
Ix = x                  -- id,   I = SKK
Bxyz = x(yz)       -- (.),  B = S(KS)K
Cxyz = xzy         -- flip, C = S(BBS)(KK) 
Txy = yx             --       T = CI
Wxy = xyy          --       W = SS(KI) = ST
Mx = xx             --       M = SII = STT = WI = WT
Lxy = x(yy)         --       L = CBM = BWB = QM
Qxyz = y(xz)      --       Q = CB
Oxy = y(xy)        --       O = SI
Uxy = y(xxy)      --       U = LO
Yx = x(Yx)         --       Y = SLL = BML = UU
```
