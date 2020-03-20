# Episode 3 - Configurator

## Foreword

This is part of [The Pragmatic Haskeller](https://github.com/cakesolutions/the-pragmatic-haskeller) series.

**At the moment, the examples of this post are not runnable, because they require disk access.**


## Beating hardcoding, not hard coding

This episode will be relatively short, but I hope insightful nevertheless.
If you was an accurate reader, you might have noticed a small code smell in
our previous episode. Look at this excerpt:


``` haskell
app :: SnapletInit Pragmatic Pragmatic
app = makeSnaplet "pragmatic" "Pragmatic web service" Nothing $ do
    d <- nestSnaplet "db" db $ mongoDBInit 10 (host "127.0.0.1") "pragmatic-haskeller"
    addRoutes routes
    return $ Pragmatic d
```

As you can see, some configuration parameters are hardcoded, specifically the
hostname and the name of the database. A much better solution would be to move
this stuff into a configuration file, so what are we waiting for?

## A side note
 
Before showing you the solution, let me make a small digression. Before finding
the current configuration (no pun intended), I was looking for a library to manage my configuration files.
There was a couple of options, but to my eyes the best solution was, once again,
provided by [bos](http://www.serpentine.com/blog/) and from his library [Configurator](http://hackage.haskell.org/package/configurator).
While I was wrapping my head around the plumbing to integrate Configurator with
my webapp, I discovered that Snap [already uses it](https://github.com/snapframework/snap/blob/master/snap.cabal#L153)!
Not only was this a nice surprise, but once again it's the proof of how well thought
Snap was. Lo and behold, it's very easy to load configuration files from Snap;
generally, you have two options:

1. Per app configuration. It's a configuration file stored in the root of your
   application.

2. Per snaplet, meaning you can achieve decoupling and separation of interest
   having specific configuration files in your snaplet sub directories. How nice.

From my understanding and taking a peek to some snaplets around the web, Snap
does not export directly Configurator's functions. This means that:

* If you are using 1. you'll still need to include Configurator to the list
  of dependencies in your project, because you need to explicitly import it to
  have access to some functions as `require`.

* If you are using 2. you don't need Configurator, but it's snaplet developer's
  responsibility to lookup specific parameters inside the configuration file.
  If the snaplet is correctly coded, all it takes for the final user is to put
  a `devel.cfg` file inside the snaplet subfolder and he's sorted!

If I misunderstood something please let me know.

## Putting pieces together

Which approach we'll use? It turns out we need approach 1. Let's place our 
config file in the top level:


``` json
pragmatic {
    db = "pragmatic-haskeller"
    host = "127.0.0.1"
}
```

The syntax is somewhat similar to JSON, but not quite equal. Configurator is
pretty powerful, and more examples can be found [here](http://hackage.haskell.org/packages/archive/configurator/latest/doc/html/Data-Configurator.html). With such file, the
last step is to load it up and use our new shining external parameters:

``` haskell
app :: SnapletInit Pragmatic Pragmatic
app = makeSnaplet "pragmatic" "Pragmatic web service" Nothing $ do
    conf <- getSnapletUserConfig
    dbName <- liftIO $ require conf "pragmatic.db"
    dbHost <-  liftIO $ require conf "pragmatic.host"
    d <- nestSnaplet "db" db $ mongoDBInit 10 (host dbHost) dbName
    addRoutes routes
    return $ Pragmatic d
```

Unsurprisingly, we are accessing nested fields in the config file fully
qualifying them. Well, not bad! We now have a configuration file, bye bye
hardcoded parameters!

## External References

Refer to the official documentation, as always:

* [Haddock Documentation for Configurator](http://hackage.haskell.org/package/configurator)

## The code

Grab the code [here](https://github.com/cakesolutions/the-pragmatic-haskeller/tree/master/03-config).
The example is self contained, just cabal-install it!

## Next Time

What have in common recipes and puppies?
Stay tuned!

A.
