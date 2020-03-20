# Covariance, contravariance, and positive and negative position

<!DOCTYPE html>
<p><b>NOTE</b> This article is still awaiting review for correctness.</p><p>Typeclasses such as
<a href="http://haddock.stackage.org/lts-3.10/base-4.8.1.0/Data-Bifunctor.html">Bifunctor</a>
are often expressed in terms of whether they are <i>covariant</i> or
<i>contravariant</i>. While these terms may appear intimidating to the unfamiliar,
they are a precise language for discussing these concepts, and once explained
are relatively easy to understand. Furthermore, the related topics of <i>positive
and negative position</i> can greatly simplify how you think about complex data
structures. This topic also naturally leads into <i>subtyping</i>.</p><p>This article is intended to give a developer-focused explanation of the terms
without diving into the category theory behind them too much. For more
information, please see <a href="https://en.wikipedia.org/wiki/Covariance_and_contravariance_%28computer_science%29">the Wikipedia page on covariance and
contravariance</a>.</p><h2>The Functor typeclass: covariant functor</h2><p>Let's consider the following functions (made monomorphic for clarity):</p><pre><code class="haskell">showInt :: Int -&gt; String
showInt = show

floorInt :: Double -&gt; Int
floorInt = floor</code></pre><p>Now suppose that we have a value:</p><pre><code class="haskell">maybeInt :: Maybe Int
maybeInt = Just 5</code></pre><p>We know <code>Maybe</code> is an instance of <code>Functor</code>, providing us with the following function:</p><pre><code class="haskell">fmapMaybe :: (a -&gt; b) -&gt; Maybe a -&gt; Maybe b
fmapMaybe = fmap</code></pre><p>We can use <code>fmapMaybe</code> and <code>showInt</code> together to get a new, valid, well-typed value:</p><pre><code class="haskell">maybeString :: Maybe String
maybeString = fmapMaybe showInt maybeInt</code></pre><p>However, we can't do the same thing with <code>floorInt</code>. The reason for this is
relatively straightforward: in order to use <code>fmapMaybe</code> on our <code>Maybe Int</code>, we
need to provide a function that takes an <code>Int</code> as an input, whereas <code>floorInt</code>
returns an <code>Int</code> as an output. This is a long-winded way of saying that <code>Maybe</code>
is covariant on its type argument, or that the <code>Functor</code> typeclass is a
covariant functor.</p><p>Doesn't make sense yet? Don't worry, it shouldn't. In order to understand this
better, let's contrast it with something different.</p><h2>A non-covariant data type</h2><p>Consider the following data structure representing how to create a <code>String</code>
from something:</p><pre><code class="haskell">newtype MakeString a = MakeString { makeString :: a -&gt; String }</code></pre><p>We can use this to convert an <code>Int</code> into a <code>String</code>:</p><pre><code class="haskell">newtype MakeString a = MakeString { makeString :: a -&gt; String }

showInt :: MakeString Int
showInt = MakeString show

main :: IO ()
main = putStrLn $ makeString showInt 5</code></pre><p>The output for this program is, as expected, <code>5</code>. But suppose we want to both
add <code>3</code> to the <code>Int</code> and turn it into a <code>String</code>. We can do:</p><pre><code class="haskell">newtype MakeString a = MakeString { makeString :: a -&gt; String }

plus3ShowInt :: MakeString Int
plus3ShowInt = MakeString (show . (+ 3))

main :: IO ()
main = putStrLn $ makeString plus3ShowInt 5</code></pre><p>But this approach is quite non-compositional. We'd ideally like to be able to
just apply more functions to this data structure. Let's first write that up
without any typeclasses:</p><pre><code class="haskell">newtype MakeString a = MakeString { makeString :: a -&gt; String }

mapMakeString :: (b -&gt; a) -&gt; MakeString a -&gt; MakeString b
mapMakeString f (MakeString g) = MakeString (g . f)

showInt :: MakeString Int
showInt = MakeString show

plus3ShowInt :: MakeString Int
plus3ShowInt = mapMakeString (+ 3) showInt

main :: IO ()
main = putStrLn $ makeString plus3ShowInt 5</code></pre><p>But this kind of mapping inside a data structure is exactly what we use the
<code>Functor</code> type class for, right? So let's try to write an instance!</p><pre><code class="haskell">instance Functor MakeString where
    fmap f (MakeString g) = MakeString (g . f)</code></pre><p>Unfortunately, this doesn't work:</p><pre><code>Main.hs:4:45:
    Couldn't match type ‘b’ with ‘a’
      ‘b’ is a rigid type variable bound by
          the type signature for
            fmap :: (a -&gt; b) -&gt; MakeString a -&gt; MakeString b
          at Main.hs:4:5
      ‘a’ is a rigid type variable bound by
          the type signature for
            fmap :: (a -&gt; b) -&gt; MakeString a -&gt; MakeString b
          at Main.hs:4:5
    Expected type: b -&gt; a
      Actual type: a -&gt; b
    Relevant bindings include
      g :: a -&gt; String (bound at Main.hs:4:24)
      f :: a -&gt; b (bound at Main.hs:4:10)
      fmap :: (a -&gt; b) -&gt; MakeString a -&gt; MakeString b
        (bound at Main.hs:4:5)
    In the second argument of ‘(.)’, namely ‘f’
    In the first argument of ‘MakeString’, namely ‘(g . f)’</code></pre><p>To understand why, let's compare the type for <code>fmap</code> (specialized to
<code>MakeString</code>) with our <code>mapMakeString</code> type:</p><pre><code class="haskell">mapMakeString :: (b -&gt; a) -&gt; MakeString a -&gt; MakeString b
fmap          :: (a -&gt; b) -&gt; MakeString a -&gt; MakeString b</code></pre><p>Notice that <code>fmap</code> has the usual <code>a -&gt; b</code> parameter, whereas <code>mapMakeString</code>
instead has a <code>b -&gt; a</code>, which goes in the opposite direction. More on that
next.</p><p><b>Exercise</b>: Convince yourself that the <code>mapMakeString</code> function has the only
valid type signature we could apply to it, and that the implementation is the
only valid implementation of that signature. (It's true that you can change the
variable names around to cheat and make the first parameter <code>a -&gt; b</code>, but then
you'd also have to modify the rest of the type signature.)</p><h2>Contravariance</h2><p>What we just saw is that <code>fmap</code> takes a function from <code>a -&gt; b</code>, and lifts it to
<code>f a -&gt; f b</code>. Notice that the <code>a</code> is always the "input" in both cases, whereas
the <code>b</code> is the "output" in both cases. By contrast, <code>mapMakeString</code> has the
normal <code>f a -&gt; f b</code>, but the initial function has its types reversed: <code>b -&gt; a</code>.
This is the core of covariance vs contravariance:</p><ul><li>In covariance, both the original and lifted functions point in the same
direction (from <code>a</code> to <code>b</code>)</li><li>In contravariance, the original and lifted functions point in <i>opposite</i>
directions (one goes from <code>a</code> to <code>b</code>, the other from <code>b</code> to <code>a</code>)</li></ul><p>This is what is meant when we refer to the normal <code>Functor</code> typeclass in
Haskell as a covariant functor. And as you can probably guess, we can just as
easily define a contravariant functor. In fact, <a href="http://haddock.stackage.org/lts-3.10/contravariant-1.3.3/Data-Functor-Contravariant.html#t:Contravariant">it exists in the contravariant
package</a>.
Let's go ahead and use that typeclass in our toy example:</p><pre><code class="haskell">import Data.Functor.Contravariant

newtype MakeString a = MakeString { makeString :: a -&gt; String }

instance Contravariant MakeString where
    contramap f (MakeString g) = MakeString (g . f)

showInt :: MakeString Int
showInt = MakeString show

plus3ShowInt :: MakeString Int
plus3ShowInt = contramap (+ 3) showInt

main :: IO ()
main = putStrLn $ makeString plus3ShowInt 5</code></pre><p>Our implementation of <code>contramap</code> is identical to the <code>mapMakeString</code> used
before, which hopefully isn't too surprising.</p><h3>Example: filtering with <code>Predicate</code></h3><p>Let's say we want to print out all of the numbers from 1 to 10, where the
English word for that number is more than three characters long. Using a simple
helper function <code>english :: Int -&gt; String</code> and <code>filter</code>, this is pretty simple:</p><pre><code>greaterThanThree :: Int -&gt; Bool
greaterThanThree = (&gt; 3)

lengthGTThree :: [a] -&gt; Bool
lengthGTThree = greaterThanThree . length

englishGTThree :: Int -&gt; Bool
englishGTThree = lengthGTThree . english

english :: Int -&gt; String
english 1 = "one"
english 2 = "two"
english 3 = "three"
english 4 = "four"
english 5 = "five"
english 6 = "six"
english 7 = "seven"
english 8 = "eight"
english 9 = "nine"
english 10 = "ten"

main :: IO ()
main = print $ filter englishGTThree [1..10]</code></pre><p>The contravariant package provides a newtype wrapper around such <code>a -&gt; Bool</code>
functions, called <code>Predicate</code>. We can use this newtype to wrap up our helper
functions and avoid explicit function composition:</p><pre><code class="haskell">import Data.Functor.Contravariant

greaterThanThree :: Predicate Int
greaterThanThree = Predicate (&gt; 3)

lengthGTThree :: Predicate [a]
lengthGTThree = contramap length greaterThanThree

englishGTThree :: Predicate Int
englishGTThree = contramap english lengthGTThree

english :: Int -&gt; String
english 1 = "one"
english 2 = "two"
english 3 = "three"
english 4 = "four"
english 5 = "five"
english 6 = "six"
english 7 = "seven"
english 8 = "eight"
english 9 = "nine"
english 10 = "ten"

main :: IO ()
main = print $ filter (getPredicate englishGTThree) [1..10]</code></pre><p><b>NOTE</b>: I'm not actually recommending this as a better practice than the
original, simpler version. This is just to demonstrate the capability of the
abstraction.</p><h2>Bifunctor and Profunctor</h2><p>We're now ready to look at something a bit more complicated. Consider the
following two typeclasses:
<a href="http://haddock.stackage.org/lts-3.10/profunctors-5.1.1/Data-Profunctor.html">Profunctor</a>
and
<a href="http://haddock.stackage.org/lts-3.10/base-4.8.1.0/Data-Bifunctor.html">Bifunctor</a>.
Both of these typeclasses apply to types of kind <code>* -&gt; * -&gt; *</code>, also known as
"a type constructor that takes two arguments." But let's look at their
(simplified) definitions:</p><pre><code class="haskell">class Bifunctor p where
    bimap :: (a -&gt; b) -&gt; (c -&gt; d) -&gt; p a c -&gt; p b d

class Profunctor p where
    dimap :: (b -&gt; a) -&gt; (c -&gt; d) -&gt; p a c -&gt; p b d</code></pre><p>They're identical, except that <code>bimap</code> takes a first parameter of type <code>a -&gt;
b</code>, whereas <code>dimap</code> takes a first parameter of type <code>b -&gt; a</code>. Based on this
observation, and what we've learned previously, we can now understand the
documentation for these two typeclasses:</p><blockquote><p><code>Bifunctor</code>: Intuitively it is a bifunctor where both the first and second
arguments are covariant.</p><p><code>Profunctor</code>: Intuitively it is a bifunctor where the first argument is
contravariant and the second argument is covariant.</p></blockquote><p>These are both bifunctors since they take two type parameters. They both treat
their second parameter in the same way: covariantly. However, the first
parameter is treated differently by the two: <code>Bifunctor</code> is covariant, and
<code>Profunctor</code> is contravariant.</p><p><b>Exercise</b> Try to think of a few common datatypes in Haskell that would be
either a <code>Bifunctor</code> or <code>Profunctor</code>, and write the instance.</p><p><b>Hint</b> Some examples are <code>Either</code>, <code>(,)</code>, and <code>-&gt;</code> (a normal function from
<code>a</code> to <code>b</code>). Figure out which is a <code>Bifunctor</code> and which is a <code>Profunctor</code>.</p><p><b>Solution</b></p><pre><code class="haskell">class Bifunctor p where
    bimap :: (a -&gt; b) -&gt; (c -&gt; d) -&gt; p a c -&gt; p b d

class Profunctor p where
    dimap :: (b -&gt; a) -&gt; (c -&gt; d) -&gt; p a c -&gt; p b d


instance Bifunctor Either where
    bimap f _ (Left x) = Left (f x)
    bimap _ f (Right x) = Right (f x)
instance Bifunctor (,) where
    bimap f g (x, y) = (f x, g y)

instance Profunctor (-&gt;) where -- functions
    dimap f g h = g . h . f</code></pre><p>Make sure you understand <i>why</i> these instances work the way they do before
moving on.</p><h2>Bivariant and invariant</h2><p>There are two more special cases for variance: bivariant means "both covariant
and contravariant," whereas invariant means "neither covariant nor
contravariant." The only types which can be bivariant are <i>phantoms</i>, where the
type doesn't actually exist. As an example:</p><pre><code class="haskell">import Data.Functor.Contravariant (Contravariant (..))

data Phantom a = Phantom
instance Functor Phantom where
    fmap _ Phantom = Phantom
instance Contravariant Phantom where
    contramap _ Phantom = Phantom</code></pre><p>Invariance will usually (always?) occur when a type parameter is used multiple
times in the data structure, e.g.:</p><pre><code class="haskell">data ToFrom a = ToFrom (a -&gt; Int) (Int -&gt; a)</code></pre><p><b>Exercise</b> Convince yourself that you can not make an instance of either <code>Functor</code> nor <code>Contravariant</code> for this datatype.</p><p><b>Exercise</b> Explain why there's also no way to make an instance of <code>Bifunctor</code> or <code>Profunctor</code> for this datatype.</p><p>As you can see, the <code>a</code> parameter is used as both the input to a function and
output from a function in the above data type. This leads directly to our next
set of terms.</p><h2>Positive and negative position</h2><p>Let's look at some basic covariant and contravariant data types:</p><pre><code class="haskell">data WithInt a = WithInt (Int -&gt; a)
data MakeInt a = MakeInt (a -&gt; Int)</code></pre><p>By now, you should hopefully be able to identify that <code>WithInt</code> is covariant on
its type parameter <code>a</code>, whereas <code>MakeInt</code> is contravariant. Please make sure
you're confident of that fact, and that you know what the relevant <code>Functor</code>
and <code>Contravariant</code> instance will be.</p><p>Can we give a simple explanation of why each of these is covariant and
contravariant? Fortunately, yes: it has to do with the position the type
variable appears in the function. In fact, we can even get GHC to tell us this
by using <code>Functor</code> deriving:</p><pre><code class="haskell">{-# LANGUAGE DeriveFunctor #-}

data MakeInt a = MakeInt (a -&gt; Int)
    deriving Functor</code></pre><p>This results in the (actually quite readable) error message:</p><pre><code>Can't make a derived instance of ‘Functor MakeInt’:
  Constructor ‘MakeInt’ must not use the type variable in a function argument
In the data declaration for ‘MakeInt’</code></pre><p>Another way to say this is "<code>a</code> appears as an input to the function." An even
better way to say this is that "<code>a</code> appears in negative position." And now we
get to define two new terms:</p><ul><li>Positive position: the type variable is the result/output/range/codomain of the function</li><li>Negative position: the type variable is the argument/input/domain of the function</li></ul><p>When a type variable appears in positive position, the data type is covariant
with that variable. When the variable appears in negative position, the data
type is contravariant with that variable. To convince yourself that this is
true, go review the various data types we've used above, and see if this logic
applies.</p><p>But why use the terms positive and negative? This is where things get quite
powerful, and drastically simplify your life. Consider the following newtype
wrapper intended for callbacks:</p><pre><code class="haskell">newtype Callback a = Callback ((a -&gt; IO ()) -&gt; IO ())</code></pre><p>Is it covariant or contravariant on <code>a</code>? Your first instinct may be to say
"well, <code>a</code> is a function parameter, and therefore it's contravariant. However,
let's break things down a bit further.</p><p>Suppose we're just trying to deal with <code>a -&gt; IO ()</code>. As we've established many
times above: this function is contravariant on <code>a</code>, and equivalently <code>a</code> is in
negative position. This means that this function expects on input of type <code>a</code>.</p><p>But now, we wrap up this entire function as the input to a new function, via:
<code>(a -&gt; IO ()) -&gt; IO ()</code>. As a whole, does this function <i>consume</i> an <code>Int</code>, or
does it <i>produce</i> an <code>Int</code>? To get an intuition, let's look at an
implementation of <code>Callback Int</code> for random numbers:</p><pre><code class="haskell">supplyRandom :: Callback Int
supplyRandom = Callback $ \f -&gt; do
    int &lt;- randomRIO (1, 10)
    f int</code></pre><p>It's clear from this implementation that <code>supplyRandom</code> is, in fact,
<i>producing</i> an <code>Int</code>. This is similar to <code>Maybe</code>, meaning we have a solid
argument for this also being covariant. So let's go back to our
positive/negative terminology and see if it explains why.</p><p>In <code>a -&gt; IO ()</code>, <code>a</code> is in negative position. In <code>(a -&gt; IO ()) -&gt; IO ()</code>, <code>a -&gt;
IO ()</code> is in negative position. Now we just follow multiplication rules: when
you multiply two negatives, you get a positive. As a result, in <code>(a -&gt; IO ())
-&gt; IO ()</code>, <code>a</code> is in positive position, meaning that <code>Callback</code> is covariant on
<code>a</code>, and we can define a <code>Functor</code> instance. And in fact, GHC agrees with us:</p><pre><code class="haskell">{-# LANGUAGE DeriveFunctor #-}
import System.Random

newtype Callback a = Callback
    { runCallback :: (a -&gt; IO ()) -&gt; IO ()
    }
    deriving Functor

supplyRandom :: Callback Int
supplyRandom = Callback $ \f -&gt; do
    int &lt;- randomRIO (1, 10)
    f int

main :: IO ()
main = runCallback supplyRandom print</code></pre><p>Let's unwrap the magic, though, and define our <code>Functor</code> instance explicitly:</p><pre><code class="haskell">newtype Callback a = Callback
    { runCallback :: (a -&gt; IO ()) -&gt; IO ()
    }

instance Functor Callback where
    fmap f (Callback g) = Callback $ \h -&gt; g (h . f)</code></pre><p><b>Exercise 1</b>: Analyze the above <code>Functor</code> instance and understand what is occurring.</p><p><b>Exercise 2</b>: Convince yourself that the above implementation is the only one
that makes sense, and similarly that there is no valid <code>Contravariant</code>
instance.</p><p><b>Exercise 3</b>: For each of the following newtype wrappers, determine if they
are covariant or contravariant in their arguments:</p><pre><code class="haskell">newtype E1 a = E1 (a -&gt; ())
newtype E2 a = E2 (a -&gt; () -&gt; ())
newtype E3 a = E3 ((a -&gt; ()) -&gt; ())
newtype E4 a = E4 ((a -&gt; () -&gt; ()) -&gt; ())
newtype E5 a = E5 ((() -&gt; () -&gt; a) -&gt; ())

-- trickier:
newtype E6 a = E6 ((() -&gt; a -&gt; a) -&gt; ())
newtype E7 a = E7 ((() -&gt; () -&gt; a) -&gt; a)
newtype E8 a = E8 ((() -&gt; a -&gt; ()) -&gt; a)
newtype E9 a = E8 ((() -&gt; () -&gt; ()) -&gt; ())</code></pre><h2>Lifting <code>IO</code> to <code>MonadIO</code></h2><p>Let's look at something seemingly unrelated to get a feel for the power of our
new analysis tools. Consider the base function <code>openFile</code>:</p><pre><code class="haskell">openFile :: FilePath -&gt; IOMode -&gt; IO Handle</code></pre><p>We may want to use this from a monad transformer stack based on top of the <code>IO</code>
monad. The standard approach to that is to use the <code>MonadIO</code> typeclass as a
constraint, and its <code>liftIO</code> function. This is all rather straightforward:</p><pre><code class="haskell">import System.IO
import Control.Monad.IO.Class

openFileLifted :: MonadIO m =&gt; FilePath -&gt; IOMode -&gt; m Handle
openFileLifted fp mode = liftIO (openFile fp mode)</code></pre><p>But of course, we all prefer using the <code>withFile</code> function instead of
<code>openFile</code> to ensure resources are cleaned up in the presence of exceptions. As
a reminder, that function has a type signature:</p><pre><code class="haskell">withFile :: FilePath -&gt; IOMode -&gt; (Handle -&gt; IO a) -&gt; IO a</code></pre><p>So can we somehow write our lifted version with type signature:</p><pre><code class="haskell">withFileLifted :: MonadIO m =&gt; FilePath -&gt; IOMode -&gt; (Handle -&gt; m a) -&gt; m a</code></pre><p>Try as we might, this can't be done, at least not directly (if you're really
curious, see <a href="http://www.stackage.org/package/lifted-base">lifted-base</a> and its
implementation of <code>bracket</code>). And now, we have the vocabulary to explain this
succinctly: the <code>IO</code> type appears in both positive and negative position in
<code>withFile</code>'s type signature. By contrast, with <code>openFile</code>, <code>IO</code> appears
exclusively in positive position, meaning our transformation function
(<code>liftIO</code>) can be applied to it.</p>