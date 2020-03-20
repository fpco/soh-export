# Primitive Haskell

<!DOCTYPE html>
<p>The point of this chapter is to help you peel back some of the layers of
abstraction in Haskell coding, with the goal of understanding things like
primitive operations, evaluation order, and mutation. Some concepts covered
here are generally "common knowledge" in the community, while others are less
well understood. The goal is to cover the entire topic in a cohesive manner. If
a specific section seems like it's not covering anything you don't already
know, skim through it and move on to the next one.</p><p>While this chapter is called "Primitive Haskell," the topics are very much
GHC-specific. I avoided calling it "Primitive GHC" for fear of people assuming
it was about the internals of GHC itself. To be clear: these topics apply to
anyone compiling their Haskell code using the GHC compiler.</p><p>Note that we will not be fully covering all topics here. There is a "further
reading" section at the end of this chapter with links for more details.</p><h2>Let's do addition</h2><p>Let's start with a really simple question: tell me how GHC deals with the
expression <code>1 + 2</code>. What <i>actually</i> happens inside GHC? Well, that's a bit of a
trick question, since the expression is polymorphic. Let's instead use the more
concrete expression <code>1 + 2 :: Int</code>.</p><p>The <code>+</code> operator is actually a method of <a href="http://www.stackage.org/haddock/lts-1.0/base-4.7.0.2/Prelude.html#t:Num">the <code>Num</code> type class</a>, so we need to look at <a href="http://www.stackage.org/haddock/lts-1.0/base-4.7.0.2/src/GHC-Num.html#Num">the <code>Num Int</code> instance</a>:</p><pre><code class="haskell">instance  Num Int  where
    I# x + I# y = I# (x +# y)</code></pre><p>Huh... well <i>that</i> looks somewhat magical. Now we need to understand both the
<code>I#</code> constructor and the <code>+#</code> operator (and what's with the hashes all of a
sudden?). If we <a href="http://www.stackage.org/snapshot/lts-1.0/hoogle?q=I%23">do a Hoogle
search</a>, we can easily
<a href="http://www.stackage.org/haddock/lts-1.0/ghc-prim-0.3.1.0/GHC-Types.html#t:Int">find the relevant
docs</a>,
which leads us to the following definition:</p><pre><code class="haskell">data Int = I# Int#</code></pre><p>So our first lesson: the <code>Int</code> data type you've been using since you first
started with Haskell isn't magical at all, it's defined just like any other
algebraic data type... except for those hashes. We can also <a href="http://www.stackage.org/snapshot/lts-1.0/hoogle?q=%2B%23">search for
<code>+#</code></a>, and end up at
<a href="http://www.stackage.org/haddock/lts-1.0/ghc-prim-0.3.1.0/GHC-Prim.html#v:-43--35-">some
documentation</a>
giving the type:</p><pre><code class="haskell">+# :: Int# -&gt; Int# -&gt; Int#</code></pre><p>Now that we know all the types involved, go back and look at the <code>Num</code> instance
I quoted above, and make sure you feel comfortable that all the types add up
(no pun intended). Hurrah, we now understand exactly how addition of <code>Int</code>s
works. Right?</p><p>Well, not so fast. The Haddocks for <code>+#</code> have a very convenient source link...
which (apparently due to a Haddock bug) doesn't actually work. However, it's
easy enough <a href="http://www.stackage.org/haddock/lts-1.0/ghc-prim-0.3.1.0/src/GHC-Prim.html#line-1386">to find the correct hyperlinked
source</a>.
And now we see the implementation of <code>+#</code>, which is:</p><pre><code class="haskell">infixl 6 +#
(+#) :: Int# -&gt; Int# -&gt; Int#
(+#) = let x = x in x</code></pre><p>That doesn't look like addition, does it? In fact, <code>let x = x in x</code> is another
way of saying bottom, or <code>undefined</code>, or infinite loop. We have now officially
entered the world of primops.</p><h2>primops</h2><p>primops, short for primary operations, are core pieces of functionality
provided by GHC itself. They are the magical boundary between "things we do in
Haskell itself" and "things which our implementation provides." This division
is actually quite elegant; as we already explored, the standard <code>+</code> operator
and <code>Int</code> data type you're used to are actually themselves defined in normal
Haskell code, which provides many benefits: you get standard type class
support, laziness, etc. We'll explore some of that in more detail later.</p><p>Look at <a href="http://www.stackage.org/haddock/lts-1.0/ghc-prim-0.3.1.0/src/GHC-Prim.html">the implementation of other functions in
<code>GHC.Prim</code></a>;
they're <i>all</i> defined as <code>let x = x in x</code>. When GHC reaches a call to one of
these primops, it automatically replaces it with the real implementation for
you, which will be some assembly code, LLVM code, or something similar.</p><p>You may be wondering: why bother with this dummy implementation at all? The
sole reason is to give Haddock documentation for the primops a place to live.
GHC.Prim is processed by Haddock more or less like any other module; but is
effectively ignored by GHC itself.</p><p>Why do all of these functions end in a <code>#</code>? That's called the magic hash
(enabled by the <code>MagicHash</code> language extension), and it is a convention to
distinguish boxed and unboxed types and operations. Which, of course, brings us
to our next topic.</p><h2>Unboxed types</h2><p>The <code>I#</code> constructor is actually just a normal data constructor in Haskell,
which happens to end with a magic hash. However, <code>Int#</code> is <i>not</i> a normal
Haskell data type. In <code>GHC.Prim</code>, we can see that it's implementation is:</p><pre><code class="haskell">data Int#</code></pre><p>Which, like everything else in <code>GHC.Prim</code> is really a lie. In fact, it's
provided by the implementation, and is in fact a normal <code>long int</code> from C
(32-bit or 64-bit, depending on architecture). We can see something even
funnier about it in GHCi:</p><pre><code>&gt; :k Int
Int :: *
&gt; :k Int#
Int# :: #</code></pre><p>That's right, <code>Int#</code> has a different <i>kind</i> than normal Haskell datatypes: <code>#</code>.
To quote <a href="https://downloads.haskell.org/~ghc/7.8.3/docs/html/users_guide/primitives.html">the GHC
docs</a>:</p><blockquote><p>Most types in GHC are boxed, which means that values of that type are
represented by a pointer to a heap object. The representation of a Haskell
<code>Int</code>, for example, is a two-word heap object. An unboxed type, however, is
represented by the value itself, no pointers or heap allocation are involved.</p></blockquote><p>See those docs for more information on distinctions between boxed and unboxed
types. It is vital to understand those differences when working with unboxed
values. However, we're not going to go into those details now. Instead, let's
sum up what we've learnt so far:</p><ul><li><code>Int</code> addition is just normal Haskell code in a typeclass</li><li><code>Int</code> itself is a normal Haskell datatype</li><li>GHC provides <code>Int#</code> and <code>+#</code> as an unboxed <code>long int</code> and addition on that type, respectively. This is exported by <code>GHC.Prim</code>, but the real implementation is "inside" GHC.</li><li>An <code>Int</code> contains an <code>Int#</code>, which is an unboxed type.</li><li>Addition of <code>Int</code>s takes advantage of the <code>+#</code> primop.</li></ul><h2>More addition</h2><p>Alright, we understand basic addition! Let's make things a bit more
complicated. Consider the program:</p><pre><code class="haskell">main = do
    let x = 1 + 2
        y = 3 + 4
    print x
    print y</code></pre><p>We know for certain that the program will first print <code>3</code>, and then print <code>7</code>.
But let me ask you a different question. Which operation will GHC perform
first: <code>1 + 2</code> or <code>3 + 4</code>? If you guessed <code>1 + 2</code>, you're <i>probably</i> right, but
not necessarily! Thanks to referential transparency, GHC is fully within its
rights to rearrange evaluation of those expressions and add <code>3 + 4</code> before
<code>1 + 2</code>. Since neither expression depends on the result of the other, we
know that it is irrelevant which evaluation occurs first.</p><p>Note: This is covered in much more detail on the GHC wiki's <a href="https://www.haskell.org/haskellwiki/Evaluation_order_and_state_tokens">evaluation order
and state
tokens</a>
page.</p><p>That begs the question: if GHC is free to rearrange evaluation like that, how
could I say in the previous paragraph that the program will always print <code>3</code>
before printing <code>7</code>? After all, it doesn't appear that <code>print y</code> uses the
result of <code>print x</code> at all, so we not rearrange the calls? To answer that, we
again need to unwrap some layers of abstraction. First, let's evaluate and
inline <code>x</code> and <code>y</code> and get rid of the <code>do</code>-notation sugar. We end up with the
program:</p><pre><code class="haskell">main = print 3 &gt;&gt; print 7</code></pre><p>We know that <code>print 3</code> and <code>print 7</code> each have type <code>IO ()</code>, so the <code>&gt;&gt;</code> operator being used comes from the <code>Monad IO</code> instance. Before we can understand that, though, we need to look at <a href="http://www.stackage.org/haddock/lts-1.0/ghc-prim-0.3.1.0/src/GHC-Types.html#IO">the definition of <code>IO</code> itself</a></p><pre><code class="haskell">newtype IO a = IO (State# RealWorld -&gt; (# State# RealWorld, a #))</code></pre><p>We have a few things to understand about this line. Firstly,
<a href="http://www.stackage.org/haddock/lts-1.0/ghc-prim-0.3.1.0/GHC-Prim.html#t:State-35-"><code>State#</code></a>
and
<a href="http://www.stackage.org/haddock/lts-1.0/ghc-prim-0.3.1.0/GHC-Prim.html#t:RealWorld"><code>RealWorld</code></a>.
For now, just pretend like they are a single type; we'll see when we get to
<code>ST</code> why <code>State#</code> has a type parameter.</p><p>The other thing to understand is that <code>(# ... #)</code> syntax. That's an <i>unboxed
tuple</i>, and it's a way of returning multiple values from a function. Unlike a
normal, boxed tuple, unboxed tuples involve no extra allocation and create no
thunks.</p><p>So <code>IO</code> takes a real world state, and gives you back a real world state and
some value. And that right there is how we model side effects and mutation in a
referentially transparent language. You may have heard the description of <code>IO</code>
as "taking the world and giving you a new one back." What we're doing here is
threading a specific state token through a series of function calls. By
creating a dependency on the result of a previous function, we are able to
ensure evaluation order, yet still remain purely functional.</p><p>Let's see this in action, by coming back to our example from above. We're now
ready to look at <a href="http://www.stackage.org/haddock/lts-1.0/base-4.7.0.2/src/GHC-Base.html">the <code>Monad IO</code>
instance</a>:</p><pre><code class="haskell">instance  Monad IO  where
    (&gt;&gt;) = thenIO

thenIO :: IO a -&gt; IO b -&gt; IO b
thenIO (IO m) k = IO $ \ s -&gt; case m s of (# new_s, _ #) -&gt; unIO k new_s

unIO :: IO a -&gt; (State# RealWorld -&gt; (# State# RealWorld, a #))
unIO (IO a) = a</code></pre><p>(Yes, I changed things a bit to make them easier to understand. As an exercise,
compare that this version is in fact equivalent to what is actually defined in
<code>GHC.Base</code>.)</p><p>Let's inline these definitions into <code>print 3 &gt;&gt; print 7</code>:</p><pre><code class="haskell">main = IO $ \s0 -&gt;
    case unIO (print 3) s0 of
        (# s1, res1 #) -&gt; unIO (print 7) s1</code></pre><p>Notice how, even though we ignore the <i>result</i> of <code>print 3</code> (the <code>res1</code>
value), we still depend on the new state token <code>s1</code> when we evaluate <code>print 7</code>,
which forces the order of evaluation to first evaluate <code>print 3</code> and then
evaluate <code>print 7</code>.</p><p>If you look through <code>GHC.Prim</code>, you'll see that a number of primitive
operations are defined in terms of <code>State# RealWorld</code> or <code>State# s</code>, which
allows us to force evaluation order.</p><p><b>Exercise</b>: implement a function <code>getMaskingState :: IO Int</code> using the
<code>getMaskingState#</code> primop and the <code>IO</code> data constructor.</p><h2>The ST monad</h2><p>Let's compare the definitions of the <code>IO</code> and <a href="http://www.stackage.org/haddock/lts-1.0/base-4.7.0.2/src/GHC-ST.html#ST"><code>ST</code></a> types:</p><pre><code class="haskell">newtype IO   a = IO (State# RealWorld -&gt; (# State# RealWorld, a #))
newtype ST s a = ST (State# s         -&gt; (# State# s,         a #))</code></pre><p>Well <i>that</i> looks oddly similar. Said more precisely, <code>IO</code> is isomorphic to <code>ST
RealWorld</code>. <code>ST</code> works under the exact same principles as <code>IO</code> for threading
state through, which is why we're able to have things like mutable references
in the <code>ST</code> monad.</p><p>By using an uninstantiated <code>s</code> value, we can ensure that we aren't "cheating"
and running arbitrary <code>IO</code> actions inside an <code>ST</code> action. Instead, we just have
"local state" modifications, for some definition of local state. The details of
using <code>ST</code> correctly and the Rank2Types approach to <code>runST</code> are interesting,
but beyond the scope of this chapter, so we'll stop discussing them here.</p><p>Since <code>ST RealWorld</code> is isomorphic to <code>IO</code>, we should be able to convert
between the two of them. <code>base</code> does in fact <a href="http://www.stackage.org/haddock/lts-1.0/base-4.7.0.2/Control-Monad-ST.html#v:stToIO">provide the
<code>stToIO</code></a>
function.</p><p><b>Exercise</b>: write a pair of functions to convert between <code>IO a</code> and <code>ST
RealWorld a</code>.</p><p><b>Exercise</b>: <code>GHC.Prim</code> has a <a href="http://www.stackage.org/haddock/lts-1.0/ghc-prim-0.3.1.0/GHC-Prim.html#g:12">section on mutable
variables</a>,
which forms the basis on <code>IORef</code> and <code>STRef</code>. Provide a new implementation of
<code>STRef</code>, including <code>newSTRef, </code>readSTRef<code>, and </code>writeSTRef`.</p><h2>PrimMonad</h2><p>It's a bit unfortunate that we have to have two completely separate sets of
APIs: one for <code>IO</code> and another for <code>ST</code>. One common example of this is <code>IORef</code>
and <code>STRef</code>, but- as we'll see at the end of this section- there are plenty of
operations that we'd like to be able to generalize.</p><p>This is where <code>PrimMonad</code>, from the <code>primitive</code> package, comes into play. Let's
look at <a href="http://www.stackage.org/haddock/lts-1.0/primitive-0.5.4.0/src/Control-Monad-Primitive.html#PrimMonad">its definition</a>:</p><pre><code class="haskell">-- | Class of primitive state-transformer monads
class Monad m =&gt; PrimMonad m where
  -- | State token type
  type PrimState m

  -- | Execute a primitive operation
  primitive :: (State# (PrimState m) -&gt; (# State# (PrimState m), a #)) -&gt; m a</code></pre><p>Note: I have <i>not</i> included the <code>internal</code> method, since <a href="https://github.com/haskell/primitive/pull/19">it will likely be
removed</a>. In fact, at the time
you're reading this, it may already be gone!</p><p><code>PrimState</code> is an associated type giving the type of the state token. For <code>IO</code>,
that's <code>RealWorld</code>, and for <code>ST s</code>, it's <code>s</code>. <code>primitive</code> gives a way to lift
the internal implementation of both <code>IO</code> and <code>ST</code> to the monad under question.</p><p><b>Exercise</b>: Write implementations of the <code>PrimMonad IO</code> and <code>PrimMonad (ST s)</code> instances, and compare against the real ones.</p><p>The primitive package provides a number of wrappers around types and functions
from <code>GHC.Prim</code> and generalizes them to both <code>IO</code> and <code>ST</code> via the <code>PrimMonad</code>
type class.</p><p><b>Exercise</b>: Extend your previous <code>STRef</code> implementation to work in any
<code>PrimMonad</code>. After you're done, you may want to <a href="http://www.stackage.org/haddock/lts-1.0/primitive-0.5.4.0/Data-Primitive-MutVar.html">have a look at
Data.Primitive.MutVar</a>.</p><p>The <code>vector</code> package builds on top of the <code>primitive</code> package to provide
mutable vectors that can be used from both <code>IO</code> and <code>ST</code>. This chapter is <i>not</i>
a tutorial on the <code>vector</code> package, so we won't go into any more details now.
However, if you're curious, please <a href="http://www.stackage.org/haddock/nightly-2015-01-08/vector-0.10.12.2/Data-Vector-Generic-Mutable.html">look through the
<code>Data.Vector.Generic.Mutable</code>
docs</a>.</p><h2>ReaderIO monad</h2><p>To tie this off, we're going to implement a <code>ReaderIO</code> type. This will flatten
together the implementations of <code>ReaderT</code> and <code>IO</code>. Generally speaking, there's
no advantage to doing this: GHC should always be smart enough to generate the
same code for this and for <code>ReaderT r IO</code> (and in my benchmarks, they perform
identically). But it's a good way to test that you understand the details here.</p><p>You may want to try implementing this yourself before looking at the
implementation below.</p><pre><code class="haskell">{-# LANGUAGE FlexibleInstances     #-}
{-# LANGUAGE MagicHash             #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE TypeFamilies          #-}
{-# LANGUAGE UnboxedTuples         #-}
import Control.Applicative        (Applicative (..))
import Control.Monad              (ap, liftM)
import Control.Monad.IO.Class     (MonadIO (..))
import Control.Monad.Primitive    (PrimMonad (..))
import Control.Monad.Reader.Class (MonadReader (..))
import GHC.Base                   (IO (..))
import GHC.Prim                   (RealWorld, State#)

-- | Behaves like a @ReaderT r IO a@.
newtype ReaderIO r a = ReaderIO
    (r -&gt; State# RealWorld -&gt; (# State# RealWorld, a #))

-- standard implementations...
instance Functor (ReaderIO r) where
    fmap = liftM
instance Applicative (ReaderIO r) where
    pure = return
    (&lt;*&gt;) = ap

instance Monad (ReaderIO r) where
    return x = ReaderIO $ \_ s -&gt; (# s, x #)
    ReaderIO f &gt;&gt;= g = ReaderIO $ \r s0 -&gt;
        case f r s0 of
            (# s1, x #) -&gt;
                let ReaderIO g' = g x
                 in g' r s1

instance MonadReader r (ReaderIO r) where
    ask = ReaderIO $ \r s -&gt; (# s, r #)
    local f (ReaderIO m) = ReaderIO $ \r s -&gt; m (f r) s

instance MonadIO (ReaderIO r) where
    liftIO (IO f) = ReaderIO $ \_ s -&gt; f s

instance PrimMonad (ReaderIO r) where
    type PrimState (ReaderIO r) = RealWorld

    primitive f = ReaderIO $ \_ s -&gt; f s

    -- Cannot properly define internal, since there's no way to express a
    -- computation that requires an @r@ input value as one that doesn't. This
    -- limitation of @PrimMonad@ is being addressed:
    --
    -- https://github.com/haskell/primitive/pull/19
    internal (ReaderIO f) =
        f (error "PrimMonad.internal: environment evaluated")</code></pre><p><b>Exercise</b>: Modify the <code>ReaderIO</code> monad to instead be a <code>ReaderST</code> monad, and
take an <code>s</code> parameter for the specific state token.</p><h2>Further reading</h2><ul><li><a href="https://downloads.haskell.org/~ghc/7.8.3/docs/html/users_guide/primitives.html">GHC docs on primitives</a></li><li><a href="https://ghc.haskell.org/trac/ghc/wiki/Commentary/PrimOps">GHC Wiki on PrimOps</a></li><li><a href="https://www.haskell.org/haskellwiki/Evaluation_order_and_state_tokens">Evaluation order and state tokens</a></li></ul>