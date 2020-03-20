# The vector package

<!DOCTYPE html>
<p>The de facto standard package in the Haskell ecosystem for integer-indexed
array data is the <a href="http://www.stackage.org/package/vector">vector package</a>.
This corresponds at a high level to arrays in C, or the vector class in C++'s
STL. However, the vector package offers quite a bit of functionality not
familiar to those used to the options in imperative and mutable languages.</p><p>While the interface for vector is relatively straightforward, the abundance of
different modules can be daunting. This article will start off with an overview
of terminology to guide you, and then step through a number of concrete
examples of using the package.</p><h2>Example</h2><p>Since we're about to jump into a few sections of descriptive text, let's kick
this off with a concrete example to whet your appetite. We're going to count
the frequency of different bytes that appear on standard input, and then
display this content.</p><p>Note that this example is purposely written in a very generic form. We'll build
up to handling this form throughout this article.</p><pre><code class="haskell">{-# LANGUAGE FlexibleContexts #-}
import           Control.Monad.Primitive     (PrimMonad, PrimState)
import qualified Data.ByteString.Lazy        as L
import qualified Data.Vector.Generic.Mutable as M
import qualified Data.Vector.Unboxed         as U
import           Data.Word                   (Word8)

main :: IO ()
main = do
    -- Get all of the contents from stdin
    lbs &lt;- L.getContents

    -- Create a new 256-size mutable vector
    -- Fill the vector with zeros
    mutable &lt;- M.replicate 256 0

    -- Add all of the bytes from stdin
    addBytes mutable lbs

    -- Freeze to get an immutable version
    vector &lt;- U.unsafeFreeze mutable

    -- Print the frequency of each byte
    -- In newer vectors: we can use imapM_
    U.zipWithM_ printFreq (U.enumFromTo 0 255) vector

addBytes :: (PrimMonad m, M.MVector v Int)
         =&gt; v (PrimState m) Int
         -&gt; L.ByteString
         -&gt; m ()
addBytes v lbs = mapM_ (addByte v) (L.unpack lbs)

addByte :: (PrimMonad m, M.MVector v Int)
        =&gt; v (PrimState m) Int
        -&gt; Word8
        -&gt; m ()
addByte v w = do
    -- Read out the old count value
    oldCount &lt;- M.read v index
    -- Write back the updated count value
    M.write v index (oldCount + 1)
  where
    -- Indices in vectors are always Ints. Our bytes come in as Word8, so we
    -- need to convert them.
    index :: Int
    index = fromIntegral w

printFreq :: Int -&gt; Int -&gt; IO ()
printFreq index count = putStrLn $ concat
    [ "Frequency of byte "
    , show index
    , ": "
    , show count
    ]</code></pre><h2>Terminology</h2><p>There are two different varieties of vectors: immutable and mutable. Immutable
vectors (such as provided by the <code>Data.Vector</code> module) are essentially
swappable with normal lists in Haskell, though with drastically different
performance characteristics (discussed below). The high-level API is similar to
lists, it implements common typeclasses like <code>Functor</code> and <code>Foldable</code>, and
plays quite nicely with parallel code.</p><p>By contrast, mutable vectors are much closer to C-style arrays. Operations
working on these values must live in the <code>IO</code> or <code>ST</code> monads (see <code>PrimMonad</code>
below for more details). Concurrent access from multiple threads has all of the
normal concerns of shared mutable state. And perhaps most importantly for
usage: mutable vectors can be <i>much</i> more efficient for certain use cases.</p><p>However, that's not the only dimension of choice you get in the vector package.
vector itself defines three flavors: boxed
(<code>Data.Vector</code>/<code>Data.Vector.Mutable</code>), storable (<code>Data.Vector.Storable</code> and
<code>Data.Vector.Storable.Mutable</code>), and unboxed (<code>Data.Vector.Unboxed</code> and
<code>Data.Vector.Unboxed.Mutable</code>). (There's also technically primitive vectors,
but in practice you should always prefer unboxed vectors; see the module
documentation for more information on the distinction here.)</p><p>And our final point: in addition to having these three flavors, the vector
package provides a typeclass-based interface which allows you to write code
that works in any of these three (plus other vector types that may be defined
in other packages, like
<a href="http://www.stackage.org/package/hybrid-vectors">hybrid-vectors</a>). These
interfaces are in <code>Data.Vector.Generic</code> and <code>Data.Vector.Generic.Mutable</code>. When
using these interfaces, you must still eventually choose a concrete
representation, but your helper code can be agnostic to what it is.</p><p>What's nice is that - with small differences - all four mutable modules have
the same interface, and all four immutable modules have the same interface.
This means you can focus on learning one type of vector, and almost for free
have that knowledge apply to other types as well. It then just becomes a
question of choosing the representation that best fits your use case, which
we'll get to shortly.</p><h2>Efficiency</h2><p>Standard lists in Haskell are immutable, singly-linked lists. Every time you
add another value to the front of the list, it has to allocate another heap
object for that cell, create a pointer to the head of the original list, and
create a pointer to the value in the current cell. This takes up a lot of
memory for holding pointers, and makes it inefficient to index or traverse the
list (indexing to position N requires N pointer dereferences).</p><p>In contrast, vectors are stored in a contiguous set of memory locations, meaning random access
is a constant time operation, and the memory overhead per additional item in the vector
is much smaller (depending on the type of vector, which we'll cover in a
moment). However, compared to lists, prepending an item to a vector is
relatively expensive: it requires creating a new buffer in memory, copying the
old values, and then adding the new value.</p><p>There are other data structures that can be considered for list-like data, such
as <code>Seq</code> from containers, or in some cases a <code>Set</code>, <code>IntMap</code>, or <code>Map</code>.
Figuring out the best choice for each use case can only be reliably determined
via profiling and benchmarking. As a general rule though, a densely populated
collection requiring integral or random access to the values will be best served by
a vector.</p><p>Now let's talk about some of the other things that make vector so efficient.</p><h3>Boxed, storable and unboxed</h3><p>Boxed vectors hold normal Haskell values. These can be <i>any</i> values at all, and
are stored on the heap with pointers kept in the vector. The advantage is that
this works for all datatypes, but the extra memory overhead for the pointers
and the indirection of needing to dereference those pointers makes them
(relative to the next two types) inefficient.</p><p>Storable and unboxed vectors both store their data in a byte array, avoiding
pointer indirection. This is more memory efficient and allows better usage of
caches. The distinction between storable and unboxed vectors is subtle:</p><ul><li>Storable vectors require data which is an instance of the <a href="http://haddock.stackage.org/lts-3.11/base-4.8.1.0/Foreign-Storable.html#t:Storable"><code>Storable</code> type
class</a>.
This data is stored in <code>malloc</code>ed memory, which is <i>pinned</i> (the garbage
collector can't move it around). This can lead to memory fragmentation, but
allows the data to be shared over the C FFI.</li><li>Unboxed vectors require data which is an instance of the <a href="http://haddock.stackage.org/lts-3.11/primitive-0.6.1.0/Data-Primitive-Types.html"><code>Prim</code> type
class</a>.
This data is stored in GC-managed <i>unpinned</i> memory, which helps avoid memory
fragmentation. However, this data cannot be shared over the C FFI.</li></ul><p>Both the <code>Storable</code> and <code>Prim</code> typeclasses provide a way to store a value as
bytes, and to load bytes into a value. The distinction is what type of
bytearray is used.</p><p>As usual, the only true measure of performance will be benchmarking. However,
as a general guideline:</p><ul><li>If you don't need to pass values to a C FFI, and you have a <code>Prim</code> instance,
use unboxed vectors.</li><li>If you have a <code>Storable</code> instance, use a storable vector.</li><li>Otherwise, use a boxed vector.</li></ul><p>There are also other issues to consider, such as the fact that boxed vectors
are instances of <code>Functor</code> while storable and unboxed vectors are not.</p><h3>Stream fusion</h3><p>Take a guess how much memory the following program will take to run:</p><pre><code class="haskell">import qualified Data.Vector.Unboxed as V

main :: IO ()
main = print $ V.sum $ V.enumFromTo 1 (10^9 :: Int)</code></pre><p>A valid guess may be <code>10^9 * sizeof int</code> bytes. However, when compiled with
optimizations (<code>-O2</code>) on my system, it allocates a total of only 52kb! How it
is possible to create a one billion integer array without using up 4-8GB of
memory?</p><p>The vector package has a powerful technique: stream fusion. Using GHC rewrite
rules, it's able to find many cases where creating a vector is unnecessary, and
instead create a tight inner loop. In our case, GHC will end up generating code
that can avoid touching system memory, and instead work on just the <a href="https://en.wikipedia.org/wiki/Processor_register">registers</a>,
yielding not only a tiny memory footprint, but performance close to a for-loop
in C. This is one of the beauties of this library: you can write high-level
code, and optimizations can churn out something much more CPU-friendly.</p><h3>Slicing</h3><p>Above we discussed the problem of appending values to the front of a vector.
However, one place where vector shines is with <i>slicing</i>, or taking a subset of
the vector. When dealing with immutable vectors, slicing is a safe operation,
with slices being sharable with multiple threads. Slicing also works with
mutable vectors, but as usual you need to be a bit more careful.</p><h2>Replacing lists</h2><p>Enough talk! Let's start using vector. Assuming you're familiar with the list
API, this should look rather boring.</p><pre><code class="haskell">import qualified Data.Vector as V

main :: IO ()
main = do
    let list = [1..10] :: [Int]
        vector = V.fromList list :: V.Vector Int
        vector2 = V.enumFromTo 1 10 :: V.Vector Int
    print $ vector == vector2 -- True
    print $ list == V.toList vector -- also True
    print $ V.filter odd vector -- 1,3,5,7,9
    print $ V.map (* 2) vector -- 2,4,6,...,20
    print $ V.zip vector vector -- (1,1),(2,2),...(10,10)
    print $ V.zipWith (*) vector vector -- (1,4,9,16,...,100)
    print $ V.reverse vector -- 10,9,...,1
    print $ V.takeWhile (&lt; 6) vector -- 1,2,3,4,5
    print $ V.takeWhile odd vector -- 1
    print $ V.takeWhile even vector -- []
    print $ V.dropWhile (&lt; 6) vector -- 6,7,8,9,10
    print $ V.head vector -- 1
    print $ V.tail vector -- 2,3,4,...,10
    print $ V.head $ V.takeWhile even vector -- exception!</code></pre><p>Hopefully there's nothing too surprising about this. Most <code>Prelude</code> functions
that apply to lists have a corresponding vector function. If you know what a
function does in <code>Prelude</code>, you probably know what it does in <code>Data.Vector</code>.
This is the simplest usage of the vector package: import <code>Data.Vector</code>
qualified, convert to/from lists with <code>V.fromList</code> and <code>V.toList</code>, and then
prefix your function calls with <code>V.</code>.</p><ul><li><p>Exercise 1: Try out some other functions available in the <a href="http://haddock.stackage.org/lts-3.11/vector-0.10.12.3/Data-Vector.html"><code>Data.Vector</code>
module</a>.
In particular, try some of the fold functions, which we haven't covered here.</p></li><li><p>Exercise 2: Try using the <code>Functor</code>, <code>Foldable</code>, and <code>Traversable</code> versions of
functions with a vector</p></li><li><p>Exercise 3: Use an unboxed (or storable) vector instead of the boxed vectors
we were using above. What code did you have to change from the original
example?  Do all of your examples from exercise 2 still work?</p></li></ul><p>There are also a number of functions in the <code>Data.Vector</code> module with no
corresponding function in <code>Prelude</code>. Many of these are related to mutable
vectors (which we'll cover shortly). Others are present to provide more
efficient means of manipulating a vector, based on their special in-memory
representation.</p><h2>Mutable vectors</h2><p>I want to test how fair the <code>System.Random</code> number generator is at generating
numbers between 0 and 9, inclusive. I want to generate 1,000,000 random values,
count the frequency of each result, and then print how often each value
appeared. Let's first implement this using immutable vectors:</p><pre><code class="haskell">import           Data.Vector.Unboxed ((!), (//))
import qualified Data.Vector.Unboxed as V
import           System.Random       (randomRIO)

main :: IO ()
main = do
    let v0 = V.replicate 10 (0 :: Int)

        loop v 0 = return v
        loop v rest = do
            i &lt;- randomRIO (0, 9)
            let oldCount = v ! i
                v' = v // [(i, oldCount + 1)]
            loop v' (rest - 1)

    vector &lt;- loop v0 (10^6)
    print vector</code></pre><p>We've introduced the <code>!</code> operator for indexing, and the <code>//</code> operator for
updating. Other than that, this is fairly straightforward code. When I ran this
on my system, it had 48MB maximum memory residency, and took 1.968s to
complete. Surely we can do better.</p><p>This problem is inherently better as a mutable state one: instead of generating
a new immutable <code>Vector</code> for each random number generated, we'd like to simply
increment a piece of memory. Let's rewrite this to use a mutable, unboxed
vector:</p><pre><code class="haskell">import           Control.Monad               (replicateM_)
import           Data.Vector.Unboxed         (freeze)
import qualified Data.Vector.Unboxed.Mutable as V
import           System.Random               (randomRIO)

main :: IO ()
main = do
    vector &lt;- V.replicate 10 (0 :: Int)

    replicateM_ (10^6) $ do
        i &lt;- randomRIO (0, 9)
        oldCount &lt;- V.read vector i
        V.write vector i (oldCount + 1)

    ivector &lt;- freeze vector
    print ivector</code></pre><p>Once again, we use <code>replicate</code> to create a size-10 vector filled with 0. But
now we've created a mutable vector (note the change in import). We then use
<code>replicateM_</code> to perform the inner action 1,000,000 times, namely: generate a
random index, read the old value at that index, increment it, and write it
back.</p><p>After we're finished, we <i>freeze</i> the vector (more on that in the next section)
and print it. The resulting distribution of values is the same (or close - we are dealing with random
numbers here) as the previous calculation using an immutable vector. But instead of 48MB and 1.968s,
this program has a maximum residency of 44KB and runs in 0.247s! That's a
significant improvement!</p><p>If we feel like being even more adventurous, we can replace our <code>read</code> and
<code>write</code> calls with <code>unsafeRead</code> and <code>unsafeWrite</code>. That will disable some
bounds checks before reading and writing. This can be a nice performance boost
in very tight loops, but has the potential to <a href="https://en.wikipedia.org/wiki/Segmentation_fault">segfault</a>
your program, so caveat
emptor! For example, try replacing <code>replicate 10</code> with <code>replicate 9</code>, change
the <code>read</code> for an <code>unsafeRead</code>, and run your program. You'll see something
like:</p><pre><code>internal error: evacuate: strange closure type -1944718914
    (GHC version 7.10.2 for x86_64_unknown_linux)
    Please report this as a GHC bug:  http://www.haskell.org/ghc/reportabug
Aborted (core dumped)</code></pre><p>The same logic applies to the other <code>unsafe</code> functions in vector. The
nomenclature means: <code>unsafe</code> may segfault your whole process, while
not-marked-unsafe may just throw an impure exception (also not great, but
certainly better than a segfault).</p><p>And if you were curious: on my system using <code>unsafeRead</code> and <code>unsafeWrite</code>
speeds the program up marginally, from 0.247s to 0.233s. In our example, most
of our time is spent on generating the random numbers, so taking off the safety
checks does not have a significant impact.</p><h2>Freezing and thawing</h2><p>We used the <code>freeze</code> function above. The behavior of this may not be
immediately obvious. When you freeze a mutable vector, what happens is:</p><ol><li>A new mutable vector of the same size is created</li><li>Each value in the original mutable vector is copied to the new mutable vector</li><li>A new immutable vector is created out of the memory space used by the new mutable vector</li></ol><p>Why not just freeze it in place? Two reasons, actually:</p><ol><li><p>It has the potential to break referential transparency. Consider this code:</p><pre><code class="haskell">import           Data.Vector.Unboxed         (freeze)
import qualified Data.Vector.Unboxed.Mutable as V

main :: IO ()
main = do
    vector &lt;- V.replicate 1 (0 :: Int)
    V.write vector 0 1
    ivector &lt;- freeze vector
    print ivector
    V.write vector 0 2
    print ivector</code></pre><p>If we froze the vector in-place in the call to <code>freeze</code>, then the second
<code>write</code> call would modify our <code>ivector</code> value, meaning that the first and
second call to <code>print ivector</code> would have different results!</p></li><li><p>When you freeze a mutable vector, its memory is marked differently for
garbage collection purposes. Later attempts to write to that same memory can
lead to a segfault.</p></li></ol><p>However, if you really want to avoid that extra buffer copy, and are certain
it's safe, you can use <code>unsafeFreeze</code>. And in fact, our random number example
above is a case where <code>freeze</code> can be safely replaced by <code>unsafeFreeze</code>, since
after the freeze, the original mutable vector is never used again.</p><ul><li>Exercise 1: Go ahead and make that swap and confirm that your program works
as expected.</li><li>Exercise 2: In the program just above (with <code>V.replicate 1 (0 :: Int)</code>),
replace <code>freeze</code> with <code>unsafeFreeze</code>. What result do you see?</li></ul><p>The opposite of <code>freeze</code> is <code>thaw</code>. Similar to <code>freeze</code>, <code>thaw</code> will copy to a
new mutable vector instead of exposing the current memory buffer. And also,
like <code>freeze</code>, there's an <code>unsafeThaw</code> that turns off the safety measures. Like
everything <code>unsafe</code>: caveat emptor!</p><p>(We'll cover some functions like <code>create</code> that provide safe wrappers around
<code>unsafeFreeze</code> and <code>unsafeThaw</code> later.)</p><h2>PrimMonad</h2><p>If you look at the mutation functions we used above like <code>read</code> and <code>write</code>,
you can tell that they were looking in the <code>IO</code> monad. However, vector is more
generic than that, and will allow your mutations to live in any <i>primitive
monad</i>, meaning: <code>IO</code>, strict <code>ST s</code>, and transformers sitting on top of those
two. The type class controlling this is <code>PrimMonad</code>.</p><p>You can get more information on <code>PrimMonad</code> in the <a href="primitive-haskell">Primitive
Haskell</a> article. Without diving into details: every
primitive monad also has an associated primitive state token type, which is
captured with <code>PrimState</code>. As a result, the type signatures for <code>read</code> and
<code>write</code> (for boxed vectors) look like:</p><pre><code class="haskell">read :: PrimMonad m =&gt; MVector (PrimState m) a -&gt; Int -&gt; m a
write :: PrimMonad m =&gt; MVector (PrimState m) a -&gt; Int -&gt; a -&gt; m ()</code></pre><p>Every mutable vector takes two type parameters: the state token of the monad it
lives in, and the type of value it holds. These gymnastics may seem overkill
now, but are necessary for making mutable vectors both versatile in multiple
monads, and type safe.</p><h2>modify and the ST monad</h2><p>Let's check out a particularly complicated type signature (for unboxed vectors):</p><pre><code class="haskell">modify :: Unbox a =&gt; (forall s. MVector s a -&gt; ST s ()) -&gt; Vector a -&gt; Vector a</code></pre><p>What this function does is:</p><ol><li>Creates a new mutable buffer the same length as the original vector</li><li>Copies the values from the original vector into the new mutable vector</li><li>Runs the provided <code>ST</code> action on the provided mutable vector</li><li>Unsafely freezes the mutable vector and returns it.</li></ol><p>What's great about this function is that it does the minimal amount of buffer
copying to be safe, and that it can be used from pure code (since all
side-effects are captured inside the <code>ST</code> action you provide).</p><ul><li>Exercise 1: Steps 1 and 2 should look pretty similar to a function we
discussed above. Can you figure out which one it is?</li><li>Exercise 2: Implement <code>modify</code> yourself using functions we've discussed and
<code>runST</code> from <code>Control.Monad.ST</code>.</li></ul><p>Let's use our new function to implement a Fisher-Yates shuffle. If we start
with a vector of size 20, we'll generate a random number between 0 and 19. Then
we'll swap position 19 with that generated random number. Then we'll loop, but
this time with a random number between 0 and 18 and swapping with position 18.
We continue until we get down to 0.</p><pre><code class="haskell">import           Control.Monad.Primitive     (PrimMonad, PrimState)
import qualified Data.Vector.Unboxed         as V
import qualified Data.Vector.Unboxed.Mutable as M
import           System.Random               (StdGen, getStdGen, randomR)

shuffleM :: (PrimMonad m, V.Unbox a)
         =&gt; StdGen
         -&gt; Int -- ^ count to shuffle
         -&gt; M.MVector (PrimState m) a
         -&gt; m ()
shuffleM _ i _ | i &lt;= 1 = return ()
shuffleM gen i v = do
    M.swap v i' index
    shuffleM gen' i' v
  where
    (index, gen') = randomR (0, i') gen
    i' = i - 1

shuffle :: V.Unbox a
        =&gt; StdGen
        -&gt; V.Vector a
        -&gt; V.Vector a
shuffle gen vector = V.modify (shuffleM gen (V.length vector)) vector

main :: IO ()
main = do
    gen &lt;- getStdGen
    print $ shuffle gen $ V.enumFromTo 1 (20 :: Int)</code></pre><p>Notice how <code>shuffleM</code> is a mutable, side-effecting function. However, <code>shuffle</code>
itself is pure.</p><h2>Generic</h2><p>After everything else we've dealt with, <code>Generic</code> is a relatively easy
addition. We introduce two new typeclasses:</p><pre><code class="haskell">class MVector v a
class MVector (Mutable v) a =&gt; Vector v a</code></pre><p>Said in English: an instance <code>MVector v a</code> is a mutable vector of type <code>v</code> that
can hold values of type <code>a</code>. The <code>Vector v a</code> is the immutable counterpart to
some mutable vector. You can find the mutable version with <code>Mutable v</code>.</p><p>One important thing to keep in mind is <i>kinds</i>. The kind of the <code>v</code> is <code>MVector
v a</code> is <code>* -&gt; * -&gt; *</code>, since it takes parameters for both the state token and
the value it holds. With the immutable <code>Vector v a</code>, the <code>v</code> is of kind <code>* -&gt;
*</code>. Was that a little abstract? No problem, some type signatures should help:</p><pre><code class="haskell">length :: MVector v a =&gt; v s a -&gt; Int
length :: Vector v a =&gt; v a -&gt; Int

read :: (PrimMonad m, MVector v a) =&gt; v (PrimState m) a -&gt; Int -&gt; m a</code></pre><p>It takes a bit of time to get used to these generic classes, but once you do
it's fairly easy to use them. The best advice is to practice! And as such:</p><ul><li><p>Exercise: modify the <code>shuffle</code> program above to work on a generic vector
instead of specifically on an unboxed vector.</p></li></ul><p>The final trick when working with generic vectors is that, ultimately, you will
need to provide a concrete type. If you forget to do so, you'll end up with
error messages that look like the following:</p><pre><code class="haskell">stream.hs:28:13:
    No instance for (V.Vector v0 Int) arising from a use of ‘shuffle’
    In the expression: shuffle gen
    In the second argument of ‘($)’, namely
      ‘shuffle gen $ V.enumFromTo 1 (20 :: Int)’
    In a stmt of a 'do' block:
      print $ shuffle gen $ V.enumFromTo 1 (20 :: Int)</code></pre><h2>vector-algorithms</h2><p>A package of note is
<a href="http://www.stackage.org/package/vector-algorithms">vector-algorithms</a>, which
provides some algorithms (mostly sort) on mutable vectors. For example, let's
generate 100 random numbers and then sort them.</p><pre><code class="haskell">import           Data.Vector.Algorithms.Merge (sort)
import qualified Data.Vector.Generic.Mutable  as M
import qualified Data.Vector.Unboxed          as V
import           System.Random                (randomRIO)

main :: IO ()
main = do
    vector &lt;- M.replicateM 100 $ randomRIO (0, 999 :: Int)
    sort vector
    V.unsafeFreeze vector &gt;&gt;= print</code></pre><ul><li>Exercise 1: write a helper function <code>sortImmutable</code> that uses <code>modify</code> and <code>sort</code> from vector-algorithms to sort an immutable vector safely</li><li>Exercise 2: rewrite the main function above to use <code>sortImmutable</code> and only the immutable vector API</li><li>Exercise 3: is your new version more efficient, less efficient, or the same? Explain.</li></ul><h2>mwc-random</h2><p>One final library to mention now is mwc-random, a random number generation
library built on top of vector and primitive. Its API can be a bit daunting
initially, but given your newfound understanding of the vector package, the API
might make a lot more sense now. It provides a <code>Gen s</code> type, where <code>s</code> is some
state token. You can then use <code>uniform</code> and <code>uniformR</code> to get random numbers
out of that generator.</p><p>As a final example, here's how we can shuffle the numbers 1-20 using
mwc-random.</p><pre><code class="haskell">import           Control.Monad.ST            (ST)
import qualified Data.Vector.Unboxed         as V
import qualified Data.Vector.Unboxed.Mutable as M
import           System.Random.MWC           (Gen, uniformR, withSystemRandom)

shuffleM :: V.Unbox a
         =&gt; Gen s
         -&gt; Int -- ^ count to shuffle
         -&gt; M.MVector s a
         -&gt; ST s ()
shuffleM _ i _ | i &lt;= 1 = return ()
shuffleM gen i v = do
    index &lt;- uniformR (0, i') gen
    M.swap v i' index
    shuffleM gen i' v
  where
    i' = i - 1

main :: IO ()
main = do
    vector &lt;- withSystemRandom $ \gen -&gt; do
        vector &lt;- V.unsafeThaw $ V.enumFromTo 1 (20 :: Int)
        shuffleM gen (M.length vector) vector
        V.unsafeFreeze vector
    print vector</code></pre>