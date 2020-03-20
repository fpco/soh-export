# Exceptions Best Practices

<!DOCTYPE html>
<p>This is an FP Complete coding standards document written by Michael Snoyman.
I'm exposing it to the outside world, but some of the prose definitely
maintains the coding standard approach. This piece is highly opinionated, and
I'm sure some people will have different thoughts on how to do this.</p><hr/><p>There is much debate in the Haskell community around exception handling. One
commonly stated position goes something like "all exceptions should be explicit
at the type level, and async exceptions are terrible." We can argue as much as
we want about this point in a theoretical sense. However, practically, it is
irrelevant, because GHC has already chosen a stance on this: it supports async
exceptions, and all code that runs in <code>IO</code> can have exceptions of <i>any</i> type
which is an instance of <code>Exception</code>.</p><p>As far as our coding standards go, we need to accept the world as it is, and
realize that any <code>IO</code> code can throw any exception. (We can also discuss the
theoretical benefits of the chosen setup, versus the terrible situation of
checked exceptions in Java, but that's really a separate matter.) Additionally,
all code must be written to be async-exception safe. How this is done is <i>not</i>
covered in this document.</p><p>Let's identify a few anti-patterns in Haskell exception handling, and then move
on to recommended practices.</p><h2>The bad</h2><h3>ExceptT IO anti-pattern</h3><p>A common (bad) design pattern I see is something like the following:</p><pre><code class="haskell">myFunction :: String -&gt; ExceptT MyException IO Int</code></pre><p>There are (at least) three problems with this:</p><ol><li>It's <i>non-composable</i>. If someone else has a separate exception type <code>HisException</code>, these two functions do not easily compose.</li><li>It gives an implication which is almost certainly false, namely: the only exception that can be thrown from this function is <code>MyException</code>. Almost any <code>IO</code> code in there will have the ability to throw some other type of exception, and additionally, almost any async exception can be thrown even if no synchronous exception is possible.</li><li>You haven't limited the possibility of exceptions, you've only added one extra avenue by which an exception can be thrown. <code>myFunction</code> can now either <code>throwE</code> or <code>liftIO . throwIO</code>.</li></ol><p>It is almost always wrong to wrap an <code>ExceptT</code>, <code>EitherT</code>, or <code>ErrorT</code> around an <code>IO</code>-based transformer stack.</p><p>Separate issue: it's also almost always a bad idea to have such a concrete
transformer stack used in a public-facing API. It's usually better to express a
function in terms of typeclass requirements, using mtl typeclasses as
necessary.</p><p>A similar pattern is</p><pre><code class="haskell">myFunction :: String -&gt; ExceptT Text IO Int</code></pre><p>This is usually done with the idea that in the future the error type will be changed from <code>Text</code> to something like <code>MyException</code>. However, <code>Text</code> may end up sticking around forever because it helps avoid the composition problems of a real data type. However that leads to expressing useful error data types as unstructured <code>Text</code>.</p><p>Generally the solution to the <code>ExceptT IO</code> anti-pattern is to return an <code>Either</code> from more functions and throw an exception for uncommon errors. Note that returning <code>Either</code> from <code>ExceptT IO</code> means there are now 3 distinct sources of errors in just one function.</p><p>Please note that using ExceptT, etc with a non-IO base monad (for example with pure code) is a perfectly fine pattern.</p><h3>Mask-them-all anti-pattern</h3><p>This anti-pattern goes like this: remembering to deal with async exceptions everywhere is hard, so I'll just mask them all.</p><p>Every time you do this, 17 kittens are mauled to death by the loch ness monster.</p><p>Async exceptions may be annoying, but they are vital to keeping a system
functioning correctly. The <code>timeout</code> function uses them to great benefit. The
Warp webserver bases all of its slowloris protection on async exceptions. The
cancel function from the async package will hang indefinitely if async
exceptions are masked. Et cetera et cetera.</p><p>Are async exceptions difficult to work with? Sometimes, yes. Deal with it anyway. Best practices include:</p><ul><li>Use the bracket pattern wherever possible.</li><li>If you have truly complex flow of control and non-linear scoping of resources, use the resourcet package.</li></ul><h2>The good</h2><h3>MonadThrow</h3><p>Consider the following function:</p><pre><code class="haskell">foo &lt;- lookup "foo" m
bar &lt;- lookup "bar" m
baz &lt;- lookup "baz" m
f foo bar baz</code></pre><p>If this function returns <code>Nothing</code>, we have no idea why. It could be because:</p><ol><li>"foo" wasn't in the map.</li><li>"bar" wasn't in the map.</li><li>"baz" wasn't in the map.</li><li><code>f</code> returned <code>Nothing</code>.</li></ol><p>The problem is that we've thrown away a lot of information by having our functions return <code>Maybe</code>. Instead, wouldn't it be nice if the types of our functions were:</p><pre><code class="haskell">lookup :: Eq k =&gt; k -&gt; [(k, v)] -&gt; Either (KeyNotFound k) v

f :: SomeVal -&gt; SomeVal -&gt; SomeVal -&gt; Either F'sExceptionType F'sResult</code></pre><p>The problem is that these types don't unify. Also, it's commonly the case that
we really don't need to know about <i>why</i> a lookup failed, we just need to deal with
it. For those cases, <code>Maybe</code> is better.</p><p>The solution to this is the <code>MonadThrow</code> typeclass from the exceptions package.
With that, we would write the type signatures as:</p><pre><code class="haskell">lookup :: (MonadThrow m, Eq k) =&gt; k -&gt; [(k, v)] -&gt; m v
f :: MonadThrow m =&gt; SomeVal -&gt; SomeVal -&gt; SomeVal -&gt; m F'sResult</code></pre><p>Versus the <code>Either</code> signature, we lose some information, namely the type of
exception that could be thrown. However, we gain composability and unification
with <code>Maybe</code> (as well as many other useful instances of <code>MonadThrow</code>, like
<code>IO</code>).</p><p>The <code>MonadThrow</code> typeclass is a tradeoff, but it's a well thought out tradeoff,
and usually the right one. It's also in line with Haskell's runtime exception
system, which does not capture the types of exceptions that can be thrown.</p><h3>Transformers</h3><p>The following type signature is overly restrictive:</p><pre><code class="haskell">foo :: Int -&gt; IO String</code></pre><p>This can always be generalized with a usage of <code>liftIO</code> to:</p><pre><code class="haskell">foo :: MonadIO m =&gt; Int -&gt; m String</code></pre><p>This allows our function to easily work with any transformer on top of <code>IO</code>.
However, given how easy it is to apply <code>liftIO</code>, it's not too horrible a
restriction. However, consider this function:</p><pre><code class="haskell">bar :: FilePath -&gt; (Handle -&gt; IO a) -&gt; IO a</code></pre><p>If you want your inner function to live in a transformer on top of <code>IO</code>, you'll
find it difficult to make it work. It can be done with <code>lifted-based</code>, but it's
non-trivial. Instead, it's much better to express this function in terms of
functions from either lifted-base or exceptions, and get one of the following
more generalized type signatures:</p><pre><code class="haskell">bar :: MonadBaseControl IO m =&gt; FilePath -&gt; (Handle -&gt; m a) -&gt; m a
bar :: (MonadIO m, MonadMask m) =&gt; FilePath -&gt; (Handle -&gt; m a) -&gt; m a</code></pre><p>This doesn't just apply to exception handling, but also to dealing with things
like forking threads. Another thing to consider in these cases is to use the
<code>Acquire</code> type from resourcet.</p><h3>Custom exception types</h3><p>The following is bad practice:</p><pre><code class="haskell">foo = do
    if x then return y else error "something bad happened"</code></pre><p>The problem is the usage of arbitrary string-based error messages. This makes
it difficult to handle this exceptional case directly in a higher level in the
call stack. Instead, despite the boilerplate overhead involved, it's best to
define a custom exception type:</p><pre><code class="haskell">data SomethingBad = SomethingBad
    deriving Typeable
instance Show SomethingBad where
    show SomethingBad = "something bad happened"
instance Exception SomethingBad
foo = do
    if x then return y else throwM SomethingBad</code></pre><p>Now it's trivial to catch the <code>SomethingBad</code> exception type at a higher level.
Additionally, <code>throwM</code> gives better exception ordering guarantees than <code>error</code>,
which creates an exception in a pure value that needs to be evaluated before
it's thrown.</p><p>One sore point is that some people strongly oppose a <code>Show</code> instance like this.
This is an open discussion, but for now I believe we need to make the tradeoff
at this point in the spectrum. I've proposed to the libraries mailing list to
add a new method to the <code>Exception</code> typeclass used for user-friendly display of
exceptions, which will make this less of a sore point.</p><h2>See also</h2><ul><li><a href="https://www.fpcomplete.com/user/snoyberg/general-haskell/exceptions/catching-all-exceptions">Catching all exceptions</a></li><li><a href="https://www.fpcomplete.com/user/snoyberg/general-haskell/exceptions/exceptions-and-monad-transformers">Exceptions and monad transformers</a></li><li><a href="http://www.yesodweb.com/blog/2014/05/exceptions-cont-monads">Exceptions in continuation-based monads</a></li><li><a href="http://stackoverflow.com/questions/25752900/exceptions-and-monad-transformers/25753497#25753497">A Stack Overflow answer on this subject</a></li></ul>