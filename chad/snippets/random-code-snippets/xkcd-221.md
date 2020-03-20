# xkcd 221

![getRandomNumber()](http://imgs.xkcd.com/comics/random_number.png)

From: [http://xkcd.com/221/](http://xkcd.com/221/)

```active haskell cache
import System.Random

main = randomRIO (1, 6 :: Int)
```

@@@ OK, I give up. What's going on here?

This is actually a hidden feature of the site: you can create "cacheable" snippets. We'll store the result produced the first time it's run and serve that to subsequent users to give a faster result. But beware: if the result depends on any kind of input, such as user input, random numbers, or the current time, the results will be unexpected.

As another example:

```active haskell cache
import Data.Time

main = getCurrentTime
```

The Markdown code for the above two snippets is:

    ```active haskell cache
    import System.Random

    main = randomRIO (1, 6 :: Int)
    ```

    ```active haskell cache
    import Data.Time

    main = getCurrentTime
    ```

This should be considered an "easter egg" feature, and may be removed at some point in the future.
@@@