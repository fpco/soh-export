# Monad Transformers

<!DOCTYPE html>
<h1>Monad Transformers</h1><h2>Basic Transformers</h2><p>The following is a list of some basic transformers:</p><h3>MaybeT</h3><p>A <code>Maybe a</code> wrapped in any other monad, i.e. <code>m (Maybe a)</code></p><h3>ReaderT</h3><p>A <code>Reader r a</code> in which the resulting <code>a</code> is wrapped in any other monad, i.e. <code>r -&gt; m a</code></p><h3>StateT</h3><p>A <code>State s a</code> in which the return value and state, namely <code>(a, s)</code>, are wrapped in any other monad, i.e. <code>s -&gt; m (a, s)</code></p><h3>ExceptT</h3><p>An <code>Either e a</code> wrapped in any other monad, i.e. <code>m (Either e a)</code></p><h2>Simple examples of usage</h2><p><a href="https://www.stackage.org/package/transformers">transformers</a> is a widely used package which provides transformer versions of various monads. It also provides two useful classes, <code>MonadTrans</code> and <code>MonadIO</code>.</p><p>Instances of <code>MonadTrans</code> are transformers which can be applied to other monads to create new monads. All of the transformers defined in the <code>transformers</code> package are instances of <code>MonadTrans</code>.</p><h3>MonadTrans</h3><p><code>MonadTrans</code> defines one method, <code>lift</code>, the signature of which is</p><pre><code class="haskell">lift :: Monad m =&gt; m a -&gt; t m a</code></pre><p>Given a monad <code>m</code>, we can "lift" into a constructed monad transformer <code>t</code> so long as <code>t</code> is an instance of <code>MonadTrans</code></p><h3>MonadIO</h3><p><code>MonadIO</code> defines one method, <code>liftIO</code>, the signature of which is</p><pre><code class="haskell">liftIO :: IO a -&gt; m a</code></pre><p><code>liftIO</code> allows us to lift an IO action into a transformer stack that is built on top of IO and it works no matter how deeply nested the stack is. We'll see some examples of this below.</p><p>Examples:</p><h3>MaybeT</h3><pre><code class="haskell">import Control.Monad 
import Control.Monad.Trans.Maybe 
import Control.Monad.Trans.Class 

main = do 
  password &lt;- runMaybeT getPassword
  case password of 
    Just p  -&gt; putStrLn "valid password!"
    Nothing -&gt; putStrLn "invalid password!"

isValid :: String -&gt; Bool
isValid = (&gt;= 10) . length

getPassword :: MaybeT IO String 
getPassword = do 
  password &lt;- lift getLine
  guard (isValid password)
  return password </code></pre><p>In this example, we combine the <code>IO</code> and <code>Maybe</code> monads. <code>lift getLine</code> allows us to embed the <code>IO</code> action into the <code>MaybeT</code> transformer, yielding a value of type <code>MaybeT IO String</code>.</p><p>Note that in this particular example, the use of <code>lift</code> in <code>lift getLine</code> is equivalent to <code>liftIO getLine</code> since we have a one-layer transformer on top of <code>IO</code>.</p><h3>lift vs liftIO</h3><p>Here's a (somewhat contrived) example that demonstrates the difference between <code>lift</code> and <code>liftIO</code> and the usefulness of the latter.</p><p>Suppose we added another layer to our transformer stack so that, instead of <code>MaybeT IO String</code>, we had <code>MaybeT (ExceptT MyPasswordError IO) String</code>.</p><p>As in our first example, we'd like to lift the <code>getLine</code> action into our transformer. Let's try.</p><pre><code class="haskell">getPassword' :: MaybeT (ExceptT MyPasswordError IO) String 
getPassword'  = do 
  password &lt;- lift getLine
  guard (isValid password)
  return password</code></pre><p>We get an error. Oops!</p><pre><code>Couldn't match type ‘IO’ with ‘ExceptT MyPasswordError IO’
Expected type: ExceptT MyPasswordError IO String
  Actual type: IO String
In the first argument of ‘lift’, namely ‘getLine’
In a stmt of a 'do' block: password &lt;- lift getLine</code></pre><p>If we look at the type of <code>lift</code> when specialized to various transformers, we can see the problem.</p><pre><code>&gt; :t \x -&gt; (lift x :: MaybeT IO String)
\x -&gt; (lift x :: MaybeT IO String) :: IO String -&gt; MaybeT IO String
``` 

In this example, we can use `lift` to go from `IO` into our transformer. But with a deeper stack, we run into problems:
</code></pre><blockquote><p>type MyDeeperStack = ReaderT Int (WriterT String IO) Bool
:t \x -&gt; (lift x :: MyDeeperStack)
\x -&gt; (lift x :: MyDeeperStack)
  :: WriterT String IO Bool -&gt; MyDeeperStack</p><p> ```</p></blockquote><p>In other words, the <code>m</code> from <code>lift :: m a -&gt; t m a</code> in our <code>MyDeeperStack</code> is <code>WriterT String IO</code>. So we would to need <code>lift</code> <i>again</i> in order to go from <code>IO Bool -&gt; MyDeeperStack</code>, i.e.</p><pre><code>&gt; :t \x -&gt; ((lift . lift) x :: MyDeeperStack)
\x -&gt; ((lift . lift) x :: MyDeeperStack)
  :: IO Bool -&gt; MyDeeperStack</code></pre><p>This is where <code>liftIO</code> helps us. It essentially lets us do a variable number of lifts. This lets us write less brittle code because if we decided to add yet another layer to our transformer stack, we wouldn't have to hardcode another call to <code>lift</code>.</p><p>As an example, what happens if we add a <code>MaybeT</code> to our stack?</p><pre><code class="haskell">type MyDeeperStack = ReaderT Int (WriterT String (MaybeT IO)) Bool</code></pre><p><code>lift . lift</code> will no longer allow us to lift an <code>IO</code> action into our stack because we now have a third layer.</p><pre><code>&gt; :t \x -&gt; ((lift . lift) x :: MyDeeperStack)
\x -&gt; ((lift . lift) x :: MyDeeperStack)
  :: MaybeT IO Bool -&gt; MyDeeperStack</code></pre><p>With <code>liftIO</code>, all is well:</p><pre><code>&gt; :t \x -&gt; (liftIO x :: MyDeeperStack)
\x -&gt; (liftIO x :: MyDeeperStack) :: IO Bool -&gt; MyDeeperStack</code></pre><p>Want to add another layer? No problem:</p><pre><code class="haskell">type MyDeeperStack = ReaderT Int (WriterT String (MaybeT (ExceptT String IO))) Bool</code></pre><pre><code>&gt; :t \x -&gt; (liftIO x :: MyDeeperStack)
\x -&gt; (liftIO x :: MyDeeperStack) :: IO Bool -&gt; MyDeeperStack</code></pre><p>Without <code>liftIO</code> we'd need to keep adjusting the number of lifts:</p><pre><code>&gt; :t \x -&gt; ((lift . lift . lift . lift)  x :: MyDeeperStack)
\x -&gt; ((lift . lift . lift . lift)  x :: MyDeeperStack)
  :: IO Bool -&gt; MyDeeperStack</code></pre><ul><li>More transformer usage examples</li><li>Pitfalls of Writer laziness</li><li>Dealing with exceptions and control structures (monad-control and exceptions packages), and losing state</li><li>Monad transformers: <a href="http://stackoverflow.com/questions/25752900/exceptions-and-monad-transformers/25753497#25753497">EitherT vs IO</a></li><li>https://github.com/kqr/gists/blob/master/articles/gentle-introduction-monad-transformers.md (need to get permission to relicense)</li></ul>