# Yesod's Monads




  



As you've read through this book, there have been a number of monads which have appeared: <code>Handler</code>, <code>Widget</code> and <code>YesodDB</code> (for Persistent). As with most monads, each one provides some specific functionality: <code>Handler</code> gives access to the request and allows you to send responses, a <code>Widget</code> contains HTML, CSS, and Javascript, and <code>YesodDB</code> let's you make database queries. In Model-View-Controller (MVC) terms, we could consider <code>YesodDB</code> to be the model, <code>Widget</code> to be the view, and <code>Handler</code> to be the controller.




  



So far, we've presented some very straight-forward ways to use these monads: your main handler will run in <code>Handler</code>, using <code>runDB</code> to execute a <code>YesodDB</code> query, and <code>defaultLayout</code> to return a <code>Widget</code>, which in turn was created by calls to <code>toWidget</code>.




  



However, if we have a deeper understanding of these types, we can achieve some fancier results.




 



## Monad Transformers



  



<blockquote>



Monads are like onions. Monads are *not* like cakes._Shrek, more or less_



</blockquote>




  



Before we get into the heart of Yesod's monads, we need to understand a bit about monad transformers. (If you already know all about monad transformers, you can likely skip this section.) Different monads provide different functionality: <code>Reader</code> allows read-only access to some piece of data throughout a computation, <code>Error</code> allows you to short-circuit computations, and so on.




  



Often times, however, you would like to be able to combine a few of these features together. After all, why not have a computation with read-only access to some settings variable, that could error out at any time? One approach to this would be to write a new monad like <code>ReaderError</code>, but this has the obvious downside of exponential complexity: you'll need to write a new monad for every single possible combination.




  



Instead, we have monad transformers. In addition to <code>Reader</code>, we have <code>ReaderT</code>, which adds reader functionality to any other monad. So we could represent our <code>ReaderError</code> as (conceptually):




  



```haskell
type ReaderError = ReaderT Error

```




  



In order to access our settings variable, we can use the <code>ask</code> function. But what about short-circuiting a computation? We'd like to use <code>throwError</code>, but that won't exactly work. Instead, we need to <code>lift</code> our call into the next monad up. In other words:




  



```haskell
throwError :: errValue -> Error
lift . throwError :: errValue -> ReaderT Error

```




  



There are a few things you should pick up here:




  



* A transformer can be used to add functionality to an existing monad.



* A transformer must always wrap around an existing monad.



* The functionality available in a wrapped monad will be dependent not only on the monad transformer, but also on the inner monad that is being wrapped.




  



A great example of that last point is the <code>IO</code> monad. No matter how many layers of transformers you have around an <code>IO</code>, there's still an <code>IO</code> at the core, meaning you can perform I/O in any of these __monad transformer stacks__. You'll often see code that looks like <code>liftIO
    $ putStrLn "Hello There!"</code>.




 



## The Three Transformers



  



We've already discussed two of our transformers previously: <code>Handler</code> and <code>Widget</code>. Just to recap, there are two special things about these transformers:




  



1. In order to simplify error messages, they are not actual transformers. Instead, they are newtypes that hard-code their inner monads. This is why Yesod provides a specialized <code>lift</code> function, which works for <code>Handler</code> and <code>Widget</code>.



1. In reality they have extra type parameters for the sub and master site. As a result, the Yesod libraries provide <code>GHandler sub master a</code> and <code>GWidget sub master
     a</code>, and each site gets a pair of type synonyms <code>type Handler = GHandler MyApp
     MyApp</code> and <code>type Widget = GWidget MyApp My App ()</code>.




  



In [persistent](http://hackage.haskell.org/package/persistent), we have a typeclass called <code>PersistStore</code>. This typeclass defines all of the primitive operations you can perform on a database, like <code>get</code>. This typeclass essentially looks like <code>class (Monad (b m)) => PersistStore b m</code>. <code>b</code> is the backend itself, and is in fact a monad transformer, while <code>m</code> is the inner monad that <code>b</code> wraps around. Both SQL and MongoDB have their own instances; in the case of SQL, it looks like:




  



```haskell
instance MonadBaseControl IO m => PersistBackend SqlPersist m

```




  



This means that you can run a SQL database with any underlying monad, so long as that underlying monad supports <code>MonadBaseControl IO</code>, which allows you to properly deal with exceptions in a monad stack. That basically means any transformer stack built around <code>IO</code> (besides exceptional cases like <code>ContT</code>). Fortunately for us, that includes both <code>Handler</code> and <code>Widget</code>. The takeaway here is that we can layer our Persistent transformer on top of <code>Handler</code> or <code>Widget</code>.




  



This wasn't always the case. Before Yesod 0.10, Yesod was built on top of enumerators, which do not support <code>MonadBaseControl</code>. In Yesod 0.10, we moved over to [conduit](http://hackage.haskell.org/package/conduit), which greatly simplified everything we're discussing here.




  



In order to make it simpler to refer to the relevant Persistent transformer, the [yesod-persistent](http://hackage.haskell.org/package/yesod-persistent) package defines the <code>YesodPersistBackend</code> associated type. For example, if I have a site called <code>MyApp</code> and it uses SQL, I would define something like <code>type instance YesodPersistBackend MyApp =
   SqlPersist</code>.




  



When we want to run our database actions, we'll have a <code>SqlPersist</code> wrapped around a <code>Handler</code> or <code>Widget</code>. We can then use the standard Persistent unwrap functions (like <code>runSqlPool</code>) to run the action and get back a normal <code>Handler</code>/<code>Widget</code>. To automate this, we provide the <code>runDB</code> function. Putting it all together, we can now run database actions inside our handlers and widgets.




  



Most of the time in Yesod code, and especially thus far in this book, widgets have been treated as actionless containers that simply combine together HTML, CSS and Javascript. But if you look at that last paragraph again, you'll realize that's not the way things have to be. Since a widget is a transformer on top of a handler, anything you do in a handler can be done in a widget, including database actions. All you have to do is <code>lift</code>.




 



## Example: Database-driven navbar



  



Let's put some of this new knowledge into action. We want to create a <code>Widget</code> that generates its output based on the contents of the database. Previously, our approach would have been to load up the data in a <code>Handler</code>, and then pass that data into a <code>Widget</code>. Now, we'll do the loading of data in the <code>Widget</code> itself. This is a boon for modularity, as this <code>Widget</code> can be used in any <code>Handler</code> we want, without any need to pass in the database contents.




  



```haskell active web
{-# LANGUAGE OverloadedStrings, TypeFamilies, TemplateHaskell, FlexibleContexts,
             QuasiQuotes, MultiParamTypeClasses, GADTs #-}
import Yesod
import Database.Persist.Sqlite
import Data.Text (Text)
import Data.Time
import Control.Monad.Logger (runStderrLoggingT)

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persist|
Link
    title Text
    url Text
    added UTCTime
|]

data LinksExample = LinksExample ConnectionPool

mkYesod "LinksExample" [parseRoutes|
/ RootR GET
/add-link AddLinkR POST
|]

instance Yesod LinksExample

instance RenderMessage LinksExample FormMessage where
    renderMessage _ _ = defaultFormMessage

instance YesodPersist LinksExample where
    type YesodPersistBackend LinksExample = SqlPersist
    runDB db = do
        LinksExample pool <- getYesod
        runSqlPool db pool

getRootR :: Handler RepHtml
getRootR = defaultLayout [whamlet|
<form method=post action=@{AddLinkR}>
    <p>
        Add a new link to #
        <input type=url name=url value=http://>
        \ titled #
        <input type=text name=title>
        \ #
        <input type=submit value="Add link">
<h2>Existing links
^{existingLinks}
|]

existingLinks :: Widget
existingLinks = do
    links <- lift $ runDB $ selectList [] [LimitTo 5, Desc LinkAdded]
    [whamlet|
<ul>
    $forall Entity _ link <- links
        <li>
            <a href=#{linkUrl link}>#{linkTitle link}
|]

postAddLinkR :: Handler ()
postAddLinkR = do
    url <- runInputPost $ ireq urlField "url"
    title <- runInputPost $ ireq textField "title"
    now <- liftIO getCurrentTime
    runDB $ insert $ Link title url now
    setMessage "Link added"
    redirect RootR

main :: IO ()
main = withSqlitePool "links.db3" 10 $ \pool -> do
    runStderrLoggingT $ runSqlPool (runMigration migrateAll) pool
    warpEnv $ LinksExample pool

```




  



Pay attention in particular to the <code>existingLinks</code> function. Notice how all we needed to do was apply <code>lift</code> to a normal database action. And from within <code>getRootR</code>, we treated <code>existingLinks</code> like any ordinary <code>Widget</code>, no special parameters at all. See the figure for the output of this app.




  




      




   



![](http://www.yesodweb.com/book/image/navbar)




  




 



## Example: Request information



  



Likewise, you can get request information inside a <code>Widget</code>. Here we can determine the sort order of a list based on a GET parameter.




  



```haskell active web
{-# LANGUAGE OverloadedStrings, TypeFamilies, TemplateHaskell,
             QuasiQuotes, MultiParamTypeClasses, GADTs #-}
import Yesod
import Data.Text (Text)
import Data.List (sortBy)
import Data.Ord (comparing)

data Person = Person
    { personName :: Text
    , personAge :: Int
    }

people :: [Person]
people =
    [ Person "Miriam" 25
    , Person "Eliezer" 3
    , Person "Michael" 26
    , Person "Gavriella" 1
    ]

data People = People

mkYesod "People" [parseRoutes|
/ RootR GET
|]

instance Yesod People

instance RenderMessage People FormMessage where
    renderMessage _ _ = defaultFormMessage


getRootR :: Handler RepHtml
getRootR = defaultLayout [whamlet|
<p>
    <a href="?sort=name">Sort by name
    \ | #
    <a href="?sort=age">Sort by age
    \ | #
    <a href="?">No sort
^{showPeople}
|]

showPeople :: Widget
showPeople = do
    msort <- lift $ runInputGet $ iopt textField "sort"
    let people' =
            case msort of
                Just "name" -> sortBy (comparing personName) people
                Just "age"  -> sortBy (comparing personAge)  people
                _           -> people
    [whamlet|
<dl>
    $forall person <- people'
        <dt>#{personName person}
        <dd>#{show $ personAge person}
|]

main :: IO ()
main = warpEnv People

```




  



Once again, all we need to do is <code>lift</code> our normal <code>Handler</code> code (in this case, <code>runInputGet</code>) to have it run in our <code>Widget</code>.




 



## Summary



  



If you completely ignore this chapter, you'll still be able to use Yesod to great benefit. The advantage of understanding how Yesod's monads interact is to be able to produce cleaner, more modular code. Being able to perform arbitrary actions in a <code>Widget</code> can be a powerful tool, and understanding how Persistent and your <code>Handler</code> code interact can help you make more informed design decisions in your app.




 

