# Playing with DataKinds

# Playing with DataKinds

*Disclaimer: this is not a tutorial, but rather a problem statement and a few of it's solutions that do the job. But the actual intent of this post is to raise a question regarding how to do things better.*

Haskell has a quite popular feature of giving "field accessors" for it's datatypes. But there is one problem with them: they let you fail at runtime easily. This seems to be quite an embarrassing thing to have, especially for a language that tries to be "if it compiles -- it works" as much as possible. Here's an example illustrating that (imaging we're writing producer-consumer program):

```active haskell
data JobDescription = JobOne { n :: Int }
                    | JobTwo
                    | JobThree { n :: Int }
  deriving (Show, Eq)

taskOneWorker :: JobDescription -> IO ()
taskOneWorker t = do
    putStrLn $ "n: " ++ (show $ n t)

main :: IO ()
main = do
  -- this runs ok:
  taskOneWorker (JobOne 10)

  -- this fails at runtime:
  -- taskOneWorker JobTwo

  -- this works, but we didn't want it to:
  -- taskOneWorker (JobThree 10)
```

Feel free to uncomment last lines and press "run" to see the problem.

You can see we've defined three tasks, and then wanted to write a worker function for first of them. Obvious problem is that it would:

- fail at runtime if called with JobTwo argument
- not fail at compile nor runtime for JobThree arg

It looks like we need to somehow distinguish at a type-level and state that `taskOneWorker` takes exactly `JobOne`. For that, I've found something called [Datatype promotion](https://www.haskell.org/ghc/docs/7.8.3/html/users_guide/promotion.html). From what I understood, it lets you automatically have new distinct types for `JobOne` and `JobTwo`. While this looks exactly what we need to have something close to `taskOneWorker :: JobOne -> ...`, I didn't find explanation on what exactly to do next to resolve the problem I have described.

While I've heard there are libraries that help you solve this problem, I wanted to instead have a clear understanding of what would be the simplest thing you can build in order to understand problem better.

## Solution one

First solution I came up with looks like this:

```active haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE KindSignatures #-}
{-# LANGUAGE GADTs #-}

data JobDescription = JobOne
                    | JobTwo
                    | JobThree
  deriving (Show, Eq)

data SJobDescription :: JobDescription -> * where
    SJobOne :: { jobOneN :: Int } -> SJobDescription JobOne
    SJobTwo :: SJobDescription JobTwo
    SJobThree :: { jobThreeN :: Int } -> SJobDescription JobThree

taskOneWorker :: SJobDescription JobOne -> IO ()
taskOneWorker t = do
    putStrLn $ "Job: " ++ (show $ jobOneN t)

main :: IO ()
main = do
  -- this typechecks: 
  taskOneWorker (SJobOne 10)

  -- these two don't type-check:
  -- taskOneWorker SJobTwo
  -- taskOneWorker (SJobThree 10)
```

Few things about this:

- we left our tasks as simple algebraic data type (ADT) without any parameters. This lets `DataKinds` extension easily promote our data constructors to types (so now we have types `JobOne`, `JobTwo`, `JobThree`, and their kind is `JobDescription`)
- we created a GADT (Generalized ADT), which is parameterized by a type of newly-created kind `JobDescription`, thanks to which we now have data-constructors `SJobOne`, `SJobTwo`, `SJobThree`, which create a value of type, that's stating it's job description inside.

To recap, we have three new data-constructors: `SJobOne :: Int -> SJobDescription JobOne`, `SJobTwo :: SJobDescription JobTwo` and `SJobThree :: Int -> SJobDescription JobThree`.

So, now we have everything working. Good stuff type-checks, bad stuff doesn't (try it right here!).

Few problems I see:

- we had to put distinct names `jobOneN` and `jobThreeN`, since these functions now have different types
- we can't write a worker that doesn't care which task it is, but only cares if it has some concrete fields

## Solution two

That's why I decided to also add a typeclass-based accessors, similar to how lens library gives you `makeFields` TH function.

```active haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE KindSignatures #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE FlexibleInstances #-}

data JobDescription = JobOne
                    | JobTwo
                    | JobThree
  deriving (Show, Eq)

class HasN x where
    n :: x -> Int

instance HasN (SJobDescription JobOne) where
    n (SJobOne x) = x

instance HasN (SJobDescription JobThree) where
    n (SJobThree x) = x

data SJobDescription :: JobDescription -> * where
    SJobOne :: Int -> SJobDescription JobOne
    SJobTwo :: SJobDescription JobTwo
    SJobThree :: Int -> SJobDescription JobThree

taskOneWorker :: SJobDescription JobOne -> IO ()
taskOneWorker t = do
    putStrLn $ "n: " ++ (show $ n t)

taskWorkerForAnythingWithN :: (HasN j, j ~ SJobDescription a) => j -> IO ()
taskWorkerForAnythingWithN t = do
    putStrLn $ "n: " ++ (show $ n t)

main :: IO ()
main = do
  let jobOne = SJobOne 10
  -- this one is good:
  taskOneWorker jobOne

  -- these don't type-check:
  -- taskOneWorker SJobTwo
  -- taskOneWorker (SJobThree 10)

  -- these work good
  -- taskWorkerForAnythingWithN (SJobOne 10)
  -- taskWorkerForAnythingWithN (SJobThree 10)
```

Here, you can not only see that we were able to use same "n" field-accessor to access different task's data, but also we are now good with writing the `taskWorkerForAnythingWithN` function, which doesn't care of it's concrete job type.

## Questions I still have

So last solution looks almost ideal to what I want to have. Questions I have are:

- is this solution is considered good to go?
- are there solutions to generate boilerplate for these? (if yes -- how close would they be to code from snippet?)
- what should a person with similar problem do (which materials to read)?

Stackoverflow page if you want to answer is here: [http://stackoverflow.com/questions/26439597/type-safe-named-fields-in-haskell-adts](http://stackoverflow.com/questions/26439597/type-safe-named-fields-in-haskell-adts).

Thank you!