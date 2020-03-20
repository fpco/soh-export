# Designing APIs for Extensibility

<!DOCTYPE html>
<p>Every time you make a breaking change in your API, it means that- potentially-
one or more of your users will need to change his/her code to adapt. Even if
this update is trivial, it adds friction to the code maintenance process. On
the other hand, we don't want to be constrained by bad design choices early on
in a project, and sometimes a breaking API change is the best option.</p><p>The point of this document, however, is to give you a third option: design your
APIs from the outset to be extensible. There are common techniques employed in
the Haskell world to make APIs that are resilient to changing feature-sets, and
by employing them early on in your design process, you can hopefully avoid the
painful choices between a better API and happy users.</p><p>Almost all techniques start with implementation hiding. Guidelines here are
simple: don't expose anything non-public. For example, if you write a number of
helper functions, you may not want to start off by exposing them, since you're
then telling users that these are good, stable functions to be relied upon.
Instead, use explicit export lists on your modules and only include functions
that are intended for public consumption.</p><p>More important- and more tricky- than functions are data constructors. In many
cases, you want to avoid exposing the internals of your data types to users, to
allow you to expand on them in the future. A common use case for this is some
kind of a data type providing configuration information. Consider that you're
going to communicate with some web services, so you write up the following API:</p><pre><code class="haskell">module MyAPI
    ( Settings (..)
    , makeAPICall
    ) where

data Settings = Settings
    { apiKey :: Text
    , hostName :: Text
    }

makeAPICall :: Settings -&gt; Foo -&gt; IO Bar</code></pre><p>The way your users will access this will be something like:</p><pre><code class="haskell">makeAPICall Settings
    { apiKey = myAPIKey
    , hostName = "www.example.com"
    } myFoo</code></pre><p>Now suppose a user points out that, in some cases, the standard port 80 is
<i>not</i> used for the API call. So you add a new field <code>port :: Int</code> to your
<code>Settings</code> constructor. This will break your user's code, since the <code>port</code>
field will not be set.</p><p>Instead, a more robust way of specifying your API will look like:</p><pre><code class="haskell">module MyAPI
    ( Settings
    , mkSettings
    , setHostName
    , makeAPICall
    ) where

data Settings = Settings
    { apiKey :: Text
    , hostName :: Text
    }

-- | Create a @Settings@ value. Uses default value for host name.
mkSettings :: Text -- ^ API Key
           -&gt; Settings
mkSettings key = Settings
    { apiKey = key
    , hostName = "www.example.com"
    }

setHostName :: Text -&gt; Settings -&gt; Settings
setHostName hn s = s { hostName = hn }

makeAPICall :: Settings -&gt; Foo -&gt; IO Bar</code></pre><p>Now your user code will instead look like:</p><pre><code class="haskell">makeAPICall (mkSettings myAPIKey) myFoo</code></pre><p>This has the following benefits:</p><ul><li>The user is not bothered to fill in default values (in this case, the hostname).</li><li>Extending this API to allow for more fields in the future is trivial: add a new <code>set*</code> function. Internally, you'll add a field to <code>Settings</code> and set a default value in <code>mkSettings</code>.</li></ul><p>One thing to note: please do <i>not</i> expose the field accessors directly. If you
want to provide getter functions in addition to setters, write them explicitly,
e.g.:</p><pre><code class="haskell">getHostName :: Settings -&gt; Text
getHostName = hostName</code></pre><p>The reason for this is that by exposing field accessors, users will be able to write code such as:</p><pre><code class="haskell">(mkSettings myAPIKey) { hostName = "www.example.org" }</code></pre><p>This ties your hand for future internal improvements, since you are now
required to keep a field of name <code>hostName</code> with type <code>Text</code>. By just using
<code>set</code> and <code>get</code> functions, you can change your internal representation
significantly and still provide a compatibility layer.</p><p>For those of you familiar with other languages: this is in fact quite similar
to the approach taken in Java or C#. Just because Java does it doesn't mean
it's wrong.</p><p>Note that this advice is different to, and intended to supersede, <a href="http://www.yesodweb.com/book/settings-types">the settings
type approach</a>. Projects like Warp
which previously used that settings type approach are currently migrating to
this more extensible approach.</p><p>Also, while settings have been used here as a motivating example, the same
advice applies to other contexts.</p><h2>Internal modules</h2><p>One downside of implementation hiding is that it can make it difficult for
users to do things you didn't intend for them to do with your API. You can
always add more functionality on demand, but the delay can be a major nuissance
for users. A compromise solution in the Haskell community is to provide a
<code>.Internal</code> module for your project which exports non-quite-public components.
For example, in wai, the <code>Response</code> constructors are exposed in a
<code>Network.Wai.Internal</code> module. Normally, users are supposed to use smart
constructors like <code>responseFile</code>, but occasionally they'll want more
fine-grained control.</p>