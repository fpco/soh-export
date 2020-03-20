# QuickCheck and WebDriver

This article is a recap of a talk I gave last week at the NY Haskell Users Group Meetup on some of what makes Haskell great for web development.  Before the talk I was worried that web dev might strike listeners as a little lowbrow.  It's a great group of people, but a lot of what gets presented there is on the mathy side -- catnip for people who have forgotten more about the type system than I will ever know, but sometimes over my head.

But I think the talk found its audience, which is one more reason to be excited about Haskell's future.  Our community's deep end is well-known; there's a shallower end too, and more developers are wading in every day.

My particular topic was automated testing and how Haskell's tooling is a great match for the down-and-dirty testing that web projects often require.  We took things in four steps:

- An intro to the QuickCheck library.
- A quick look at a case study: a complex web project in need of end-to-end test coverage.
- An intro to Selenium WebDriver, the tool web developers across all languages rely on when they need to script browser sessions for testing purposes.
- A demo, with code, of how QuickCheck can interact with WebDriver to generate and run test cases.

# QuickCheck Intro

We started with an introduction to QuickCheck.  If you're familiar, skip ahead.

### Why QuickCheck?

Let's start with this: what does testing consist of?  I'd say it's a three-step process:

1. Come up with a set of test cases that you think rigorously test an invariant.
2. Implement the tests, and work on your code until it passes.
3. Discover (via bug reports) cases you forgot about, representing regions of your domain where the invariant should hold but doesn't.  Return to step 2.

Testing tools are pretty hot these days, especially for dynamically typed languages.  People get that there's a lot of benefit in reducing how often one reaches Step 3.

It seems to me that the popular tools mostly address Step 2 -- formatting your tests, running them, and communicating and reasoning about the results.  All of which is important.  But in my view, Step 1 is [where differences in approach have the biggest effect](http://en.wikipedia.org/wiki/Amdahl's_law).  Bugs thrive when parts of an invariant's domain are never tested at all.

The problem is that reasoning from an invariant to an exhaustive set of test cases is *hard*.  For many domains, nobody's smart enough to think of every case.  And mindlessly writing out cases in the hope of stumbling on one that fails is both inefficient and subject to the [confirmation bias](http://www.jonathanklein.net/2013/06/cognitive-biases-in-software-engineering.html).

Unless, that is, you can have a computer do it for you.

### QuickCheck Basics

This is where QuickCheck comes in.  It helps you test more rigorously by, in essence, writing your tests for you.  Instead of writing tests, your job is now to formally express your invariants (QuickCheck calls them properties) in the form "For each value of type `Arbitrary t => t` that satisfies precondition `pc`, property `p` should hold."  QuickCheck then generates `t`s that satisfy `pc` and tests that `p` holds for each of them.

It's a pretty simple idea.  The implementation feels really elegant too, once you figure out how the `Arbitrary` class and the `Gen` and `Property` types fit together.  (I had to stare at and write a lot of examples before I really got it.)  And there's a lot of fancy functionality to discover.  Some teasers:

- It can generate as few or as many test cases as you like -- 10 or 10,000,000.
- To help you decide whether the cases generated so far are adequately covering your domain, it can report on the distribution of the values it generates.
- You can take as much control as you want of how it generate cases for you.
- When it does find a counterexample, it can help you identify the simplest case that fails for the same reason.

If you're new to QuickCheck, a good place to start is [this tutorial](https://www.fpcomplete.com/user/pbv/an-introduction-to-quickcheck-testing).

### QuickCheck and I/O

The examples in that tutorial all assume that you only want to use pure code in your property definitions.  And in fact QuickCheck can't use a function as a property if its result is an I/O action.  You can write the function, of course, and run it on its own...

``` active haskell
import Test.QuickCheck (quickCheck)

prop_LengthIO__incorrect = do
    str <- getLine
    case str of
        []   -> return $ length str == 0
        c:cs -> return $ length str == length cs + 1 
main = prop_LengthIO__incorrect
```
       
... but try passing it to `quickCheck` and you'll learn that there's no `Testable` instance for its result type:

``` active haskell
import Test.QuickCheck (quickCheck)

prop_LengthIO__incorrect = do
    str <- getLine
    case str of
        []   -> return $ length str == 0
        c:cs -> return $ length str == length cs + 1 
        
main = quickCheck prop_LengthIO__incorrect
```

That's a silly example; we're not even asking QuickCheck to generate any random values.  But there are real situations where we'd like QuickCheck to do I/O.  We'll look closely at a real-world example in a few minutes.

For now, let's look at the basic mechanics of doing I/O in a QuickCheck property.  The key is the family of functions exported by `Test.QuickCheck.Monadic`.  Here's a correct version of the example above:

``` active haskell
import Test.QuickCheck
import Test.QuickCheck.Monadic as QCM

prop_LengthIO__correct = QCM.monadicIO $ do
    str <- QCM.run getLine
    QCM.assert $ case str of
        []   -> length str == 0
        c:cs -> length str == length cs + 1 
main = quickCheck prop_LengthIO__correct
```

First, notice `monadicIO`.  Its result is a `Property` and so can be passed to `quickCheck`.  What about its parameter (the code in the `do` block)?  It's a `PropertyM IO a`.  Think of that type as a no-man's land where we get access, via helper functions, to both `IO a` and `Gen b` computations.  In this example, `run` gives us access to an `IO [Char]` computation (i.e. it pulls a `[Char]` down out of `IO` into the no-man's-land).  `pick` pulls through the `Int` from a `Gen Int`.  Finally, we `assert` that our property holds.

The point is that running I/O code inside a property isn't too hard.  For slightly more detail, see [this great StackOverflow Q&A](http://stackoverflow.com/a/2946515/492075).  For full depth, there's [this paper by the library authors](https://www.dropbox.com/s/ie5n0wjlsdren43/Testing%20Monadic%20Code%20with%20QuickCheck.pdf) (the text is searchable in Acrobat or Acrobat Reader).

# A Case Study: Gloss

I mentioned that we would look at a project in need of some testing.  I'm working (pre-launch) on Gloss, a tool that makes it painless for a publisher to add annotation functionality to an article page with one line of Javascript.  Gloss handles all the backend stuff and gives the publisher flexibility to set permissions, moderate submissions, control styling, etc., exercising as much control as they care to.

Below is a quick video demo.  Right now the tool is activated via bookmarklet, since The Atlantic isn't actually using it.  If they were, it would be activated automatically during page load, with no progress bar cluttering things up in the bottom-left corner.

What you'll see:  When Gloss is active, you can highlight some text and it'll be intelligently polished up for you, and if you reload the page the highlighted snippet will still be there.

<youtube>nzp9t3Fu6vE</youtube>

So what are some of the invariants that need testing as we're developing this tool?  Here's an important one: 

    For every publisher who might use Gloss, for every page on that publisher's
    site, for every text selection that begins and ends in the 'main text content'
    of that page: the 'mouseup' event that completes the selection should result in
    a 'Gloss It' dialog being shown.  When that dialog is clicked, the native text
    selection should be replaced by highlighting indicating a Gloss selection.  The
    text of the Gloss selection should be identical to the text of the native
    selection, modulo the differences that should result from our polishing-up of 
    the selection.

That paragraph describes a property.  True, it's a big, gnarly, complicated one, composed of many narrower sub-properties that can be factored out and tested independently.  Gloss is covered by lots of frontend and backend unit tests and narrower integration tests, but you learn something irreducible when you test end-to-end.  It gives you a confidence greater than the confidence you get from the sum of your narrower tests: "It works!" as opposed to just "It *should* work, since all its components do."

Apart from being complicated in the sense that it comprises multiple sub-properties, it's also *enormous* in scope.  To a loose approximation, it says this tool should work across the entire internet.

Let's chop it down a little by dropping the first two sentence clauses, so that it we only have the bit starting with `for every text selection` -- i.e. we're only going to require that this feature of Gloss work for every pair of character indices in the main content area of a *single (unspecified) page*.  Assuming we're talking about [the page in the video above](http://www.theatlantic.com/health/archive/2014/06/study-when-remembering-practice-doesnt-always-make-perfect/373355/), that's ~4000 characters ^2 = ~16,000,000 possible pairs.  I.e. the domain is still enormous.

Of course, not all of those pairs need to be tested.  There is some subset that together represent all the structurally distinct possible cases.  ("Structurally distinct" isn't the right term, but I think you get what I mean.  Math types, please help us out with vocabulary here.)  But figuring out which cases belong to that subset, if indeed there are few enough that enumerating them would be feasible, is too much for my brain.  I want to use QuickCheck.

You may be asking: hold the phone.  How can we use QuickCheck for something like this?  Here's what I mean: If\* we can control a browser from our test code, we should be able to inspect the page's DOM, find out what its main text content consists of, generate a `Selection` value, and have the browser implement our chosen selection and then click the 'Gloss It' dialog.  (A `Selection` would be a `(DOMLocation, DOMLocation)`, where a `DOMLocation` specifies a text node in the DOM and a character offset within that node.)  Easy, right?

\* That thing about controlling the browser?  We'll just need WebDriver for that.

# WebDriver Intro

As with QuickCheck, if you're familiar, skip ahead.

WebDriver is basically a toolchain that lets us control a browser from code.  Basically, the major browsers, with the help of WebDriver drivers (is that redundant?), implement an API allowing programmatic control via a [Selenium](http://docs.seleniumhq.org/) server process.  ("Server" in this context I think indicates that there's some kind of RPC architecture here, but I've never given it any thought and you don't need to either.)

You can either install the tools (the Selenium server, the browser/s you want to use, and appropriate browser drivers) on a box you control or use a hosted service like [Sauce Labs](http://www.saucelabs.com).  I chose the former route; [this](http://alex.nederlof.com/blog/2012/11/19/installing-selenium-with-jenkins-on-ubuntu/) guide helped me get everything set up in my dev and test environments.

Once you're configured, you can send commands via your Selenium server, either via raw HTTP requests or by using a [WebDriver library in your language of choice](http://hackage.haskell.org/package/webdriver).  You can:
- Create and close browser sessions.
- Carry out various input actions defined in the WebDriver interface.
- Ask the browser to execute arbitrary Javascript.

An example:

``` haskell
import Test.QuickCheck
import Test.QuickCheck.Monadic as QCM
import Test.WebDriver
import Test.WebDriver.Commands

openPageAndPrintBodyText = runSession defaultSession capabilities $ do
    setWindowSize (720, 800)
    openPage url
    body <- findElem $ ByTag $ "body"
    bodyText <- getText body
    liftIO $ IO8.putStrLn $ "<body> text:\n"
                         ++ take 100 (unpack bodyText)
                         ++ "\n... That was the first 100 chars!"
  where
    capabilities = allCaps { browser=chrome }
    url = "http://www.theatlantic.com/health/archive/2014/06/study-when-remembering-practice-doesnt-always-make-perfect/373355/"
```
        
You can't run that code here; the [webdriver](http://hackage.haskell.org/package/webdriver) package doesn't seem to be in School of Haskell's package database, and even if it were the code would fail at runtime, finding no Selenium server at localhost:4444.  But here's a video of what it does:

<youtube>L8rdZXEjAJ0</youtube>

Ok, back to the meat of this article: how can QuickCheck power WebDriver?

# QuickCheck-powered WebDriver

Using WebDriver with QuickCheck basically boils down to one or both of two things:

- Inspecting a browser and using facts about it (a particular window, say, or the document in a given window) as inputs for your test.
- Manipulating a browser and checking that a desired property still holds afterwards.

In our analysis of the big hairy property description above (in the 'case study' section), we talked about wanting to implement a QuickCheck property function that, for a given article:

1. Generates a `Selection` representing start and end indices in the article's main text content.
2. Simulates a user mousing down, dragging, and mousing up to create the corresponding text selection, then clicking the 'Gloss It' button when it appears.
3. Verifies that the highlighted text which replaces the selection satisfies certain properties.

So let's build things up one step at a time.

Remember, a `Selection` is a pair of `DOMLocation`s that fall in the 'main article content' of the page under consideration.  To generate those, we'll need to know how long the 'main content' area is and where it begins in the overall document.  (Actually it's even more complicated than that -- the main content may have irrelevant content interspersed within it.  We need to make sure both `DOMLocation`s fall in true 'main content' territory.)

So before anything else, we'll need to load the page and activate Gloss on it.  Then we'll inspect the DOM and gather the info we need, pulling it out into Haskell-land as a JSON object.  Finally, we'll take that info and pass in into a function of type `Gen Selection`.

This is only step 1 of 4, but suppose it's all we wanted to do -- get some info from the DOM and use it to generate a `Selection`.  We could do that like this:

``` haskell
import Test.QuickCheck
import Test.QuickCheck.Monadic as QCM
import Test.WebDriver
import Test.WebDriver.Commands

prop_HighlightsSelectedText = QCM.monadicIO $ do
    domInfo <- QCM.run $ runSession defaultSession capabilities $ do
        openPage url
        activateGloss
        getDOMInfoNeededForGeneratingSelections
    selection <- QCM.pick $ generateSelection domInfo
    QCM.run $ liftIO $ putStrLn $ "Generated a selection: " ++ show selection
  where
    capabilities = allCaps { browser=chrome }
    url = "http://www.theatlantic.com/health/archive/2014/06/study-when-remembering-practice-doesnt-always-make-perfect/373355/"
```

What we have here is our first example of QuickCheck code (the function is indeed a `Property`) telling WebDriver what to do.  We wrap the whole thing in `monadicIO` and use `runSession` to pull a result down out of the `WD` monad into `IO`.

(Before we move on: what exactly does `activateGloss` do?  The details are well outside the scope of this article, but in essence it asks the browser to execute some custom Javascript -- specifically, the JS that a publisher using Gloss would include in their page.  `getDOMInfoNeededForGeneratingSelections` is similar in principle; it runs Javascript that does the DOM inspection we've been talking about and returns its findings as a JSON object.  For more info on running custom JS, see the `executeJS` and `asyncJS` functions in [`Test.WebDriver.Commands`](https://hackage.haskell.org/package/webdriver/docs/Test-WebDriver-Commands.html).

Ok.  What we have right now is a pretty silly QuickCheck property.  It doesn't *do* anything with the `Selection` it generates, much less actually test anything (there's no use of `assert`).  So let's revise it.
    
``` haskell
import Test.QuickCheck
import Test.QuickCheck.Monadic as QCM
import Test.WebDriver
import Test.WebDriver.Commands
import Test.WebDriver.Classes (getSession)

prop_HighlightsSelectedText = monadicIO $ do
    (domInfo, sesh) <- run $ runWD defaultSession $ do
        createSession capabilities
        openPage url >> activateGloss
        domInfo <- getDOMInfoNeededForGeneratingSelections
        s <- getSession
        return (domInfo, s)
    selection <- pick $ generateSelection domInfo
    domInfo' <- run $ runWD sesh $ do
        actuallySelectText selection
        closeSession
        getDOMInfoNeededForVerifyingPropertyOfHighlightedText
    assert $ highlightedTextHasProperty domInfo'
  where
    capabilities = allCaps { browser=chrome }
    url = "http://www.theatlantic.com/health/archive/2014/06/study-when-remembering-practice-doesnt-always-make-perfect/373355/"
```

Notice some structural differences between that snippet and the previous one.  First, we've replaced `runSession` with `runWD`.  `runSession` automatically initializes a browser session before executing the code in the `WD a` that's passed as its final parameter, then automatically closes the session afterwards.  `runWD` doesn't, which suits us: unlike in the previous snippet, what we now want to do is run a `WD` computation, then a `Gen` one, then another `WD` one.  Which means that coming out of the first `WD` computation we need to have a handle on a still-running WebDriver session.  

This way of defining a WebDriver-dependent `Property` is fine as far as it goes, but it could be improved in a couple of ways.

Before we continue, an observation: the only really stateful thing about a `WDSession` (I'm referring to a Haskell value of that type, not the browser session it represents) is that it records our most recent HTTP request to the Selenium server.  And the only time that information ever gets used is immediately after the request is issued, iff the request results in a Selenium server error.

If our `WD` computations won't actually change their `WDSession`s, then we can reuse the same `WDSession` for all of them.  For example, the snippet above could look like this:

``` haskell
import Test.QuickCheck
import Test.QuickCheck.Monadic as QCM
import Test.WebDriver
import Test.WebDriver.Commands
import Test.WebDriver.Classes (getSession)

prop_HighlightsSelectedText = monadicIO $ do
    sesh <- run $ runWD defaultSession $ createSession capabilities 
    run $ runWD sesh $ openPage url >> activateGloss
    domInfo <- run $ runWD sesh getDOMInfoNeededForGeneratingSelections
    selection <- pick $ generateSelection domInfo
    run $ runWD sesh $ actuallySelectText selection
    domInfo' <- run $ runWD sesh getDOMInfoNeededForVerifyingPropertyOfHighlightedText
    assert $ highlightedTextHasProperty domInfo'
    run $ runWD sesh closeSession
  where
    capabilities = allCaps { browser=chrome }
    url = "http://www.theatlantic.com/health/archive/2014/06/study-when-remembering-practice-doesnt-always-make-perfect/373355/"
```

The point is that we can bind `sesh` once and reuse it in all subsequent calls to `runWD`.\*

This has bigger implications.  Take the fact that right now we're both creating and closing our `WDSession` inside our `Property` function.  That means every time we generate a `Selection` and test it, we're going through all the overhead of creating a session, opening the page, etc.  What if we wanted to load our target page once and have QuickCheck generate and test a series of `Selection`s without ever closing it?

We can do that; we just have to initialize a `WDSession` before the call to `quickCheck` (and close it afterwards):

``` haskell
import Test.QuickCheck
import Test.QuickCheck.Monadic as QCM
import Test.WebDriver
import Test.WebDriver.Commands
import Test.WebDriver.Classes (getSession)

prop_HighlightsSelectedText sesh domInfo = monadicIO $ do
    selection <- pick $ generateSelection domInfo
    domInfo' <- run $ runWD sesh $ do
        actuallySelectText selection
        getDOMInfoNeededForVerifyingPropertyOfHighlightedText
    assert $ highlightedTextHasProperty domInfo'

setUpAndTest = do
    (domInfo, sesh) <- runWD defaultSession $ do
        createSession capabilities >> openPage url >> activateGloss
        domInfo <- getDOMInfoNeededForGeneratingSelections
        s <- getSession
        return (domInfo, s)
    quickCheck $ prop_HighlightsSelectedText sesh domInfo
    -- If you're finding lots of bugs, you might omit the next line so that the browser stays open for manual inspection.
    runWD sesh closeSession
  where
    capabilities = allCaps { browser=chrome }
    url = "http://www.theatlantic.com/health/archive/2014/06/study-when-remembering-practice-doesnt-always-make-perfect/373355/"
```

Now we're cooking with gas, starting to make real use of QuickCheck's case-generation capabilities to power some very-real-world tests.

This code could still be little cleaner and more integrated.  If we want to feed the result of a `WD` computation into a `Gen` computation (or a call to `assert`), it's annoying to have to first pull the result out of the `WD` `do` block just to get it into scope.

[The quickcheck-webdriver library](http://hackage.haskell.org/package/quickcheck-webdriver) helps out on this front.  Its `Test.QuickCheck.Monadic.WebDriver` module exports `monadicWD` (along with some helper functions and types), which lets us rewrite `prop_HighlightsSelectedText` like this:

``` haskell
import Test.QuickCheck
import Test.QuickCheck.Monadic as QCM
import Test.WebDriver
import Test.WebDriver.Commands
import Test.WebDriver.Classes (getSession)
-- show
import Test.QuickCheck.Monadic.WebDriver as QCWD

prop_HighlightsSelectedText context domInfo = QCWD.monadicWD context $ do
    selection <- pick $ generateSelection domInfo
    run $ actuallySelectText selection
    domInfo' <- QCM.run $ getDOMInfoNeededForVerifyingPropertyOfHighlightedText
    assert $ highlightedTextHasProperty domInfo'
-- /show
setUpAndTest = do
    (domInfo, sesh) <- runWD defaultSession $ do
        createSession capabilities >> openPage url >> activateGloss
        domInfo <- getDOMInfoNeededForGeneratingSelections
        s <- getSession
        return (domInfo, s)
    quickCheck $ prop_HighlightsSelectedText (ExistingSession sesh) domInfo
    -- If you're finding lots of bugs, you might omit the next line so that the browser stays open for manual inspection.
    runWD sesh closeSession
  where
    capabilities = allCaps { browser=chrome }
    url = "http://www.theatlantic.com/health/archive/2014/06/study-when-remembering-practice-doesnt-always-make-perfect/373355/"
```

This style is even more helpful if we want to jump back and forth between `WD` and `Gen` computations several times.

Here's a video of (basically) this code in action.  The first few moments are a little tedious, but starting at 0:45 you can see QuickCheck generating `Selection`s and using WebDriver to actually select text (and then verify that the contents of the original selections and the highlighted selections match).

<youtube>YZcTQ2ytDrY</youtube>

(The code running in that video is slightly different from our snippet in that it asks for manual intervention at just a couple of points -- to get us logged in at the beginning, and then to provide a 'continue' signal twice during each call to our property function.  Asking for that signal is only necessary because this is a demo and I didn't want it to blaze through cases too fast for us to see what's happening.)

# Conclusion

The combination of QuickCheck and WebDriver can feel very complicated if you're using either or both for the first time.  But that's true for almost every testing toolset.

This combination of tools isn't appropriate for every testing situation, not even every end-to-end browser-based testing situation.  But when we have a browser-based application that needs to behave consistently across a very large document domain, I think the QuickCheck+WebDriver combination is hard to beat.

\* Of course, the underlying browser session **is** stateful, very much so, and it's up to us to make sure that each command we issue is compatible with the current browser state.