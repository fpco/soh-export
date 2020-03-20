# Untitled

[http://stackoverflow.com/questions/26439889/extension-overloadedstring-doesnt-fully-infer-isstring-why-or-what-am-i-missi](http://stackoverflow.com/questions/26439889/extension-overloadedstring-doesnt-fully-infer-isstring-why-or-what-am-i-missi)

```active haskell
{-# LANGUAGE GADTs #-}
data A
data B a where 
    Foo :: a -> B A

main = let x = Foo "ok" in putStrLn "YAY!"
```

```active haskell
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE GADTs #-}
data A
data B a where 
    Foo :: a -> B A

main = let x = Foo "ok" in putStrLn "YAY!"
```
