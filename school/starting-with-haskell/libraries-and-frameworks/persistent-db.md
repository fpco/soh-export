# Database access

The most popular architecture for web applications is three layers - one running on the clients, talking to the layer running on one or more servers, which talks to the database layer. This tutorial is going to cover database access, using the sqlite database, which makes a perfect tool for development, though it might not be up to production in such an environment. A production database using PostgrSQL or MySQL would use most of the same code.

We're going to use the `persistent` and `esqueleto` database access packages for these reasons:

1. They provide type-safe integration with Haskell, leveraging Haskell's type system.
2. They are largely database neutral, allowing easy porting from sqlite for development to a production database.
3.  They are well-integrated with Yesod, the web platform recommended by FP Complete.

# Creating a database

We're going to create our database in Haskell. In this particular case, we're going to use the file `:memory:`, which causes sqlite to create an in-memory database. To keep the database on the file system, you'd replace that with a file name. Sqlite locks the file, so this is even safe for multi-process applications. We create the database by simply declaring the type we want in a table, and then migrate them into SQL. The following will print the SQL, which will be fixed in the next example.

``` active haskell
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}

import Data.Text (Text)
import Database.Persist
import Database.Persist.Sqlite (runSqlite, runMigration)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Tutorial
   title    Text
   url      Text
   school   Bool
   deriving Show
|]

main = runSqlite ":memory:" $ runMigration migrateTables
```

Don't let all the language options scare you. `TemplateHaskell` and `QuasiQuotes` are required by `persistent`, as it uses those features. These allow including `[` enclosed text which will be passed to the `share` function, and generate a lot of code that uses other extensions. Except for `OverloadedStrings`, you generally don't need to use them yourself. `OverloadedStrings` lets you use constant string for types that you would otherwise have to cast the string to the correct type.

The important part is the <hoogle search="Database.Persist.TH.share">share</hoogle> statement, which is a Template Haskell function invocation. The first argument is a list of actions to apply to the second part. In this case, the <hoogle search="Database.Persist.TH.mkPersist">mkPersist</hoogle> action creates the appropriate data statements and type classes for the table described in the second argument for an sql database. The <hoogle search="Database.Persist.TH.mkMigrate">mkMigrate</hoogle> action creates the function `migrateTables` that will create the tables described.

The bulk of the statement is the table declaration in the second argument:

```
Tutorial
   title    Text
   url      Text
   school   Bool
   deriving Show
```

Which defines the table to be created, as well as a Haskell type `Tutorial` with the three fields `title`, `url` and `school`. The last one is a `Bool` indicating that this is an official School tutorial of some kind. Using <hoogle search="Database.Persist.TH.persistLowerCase">persistLowerCase</hoogle> causes it to case the variable names properly.

Finally, the `main` function uses the <hoogle search="Database.Persist.Sqlite.runSqlite">runSqlite</hoogle> function to run SQL code, which just uses <hoogle search="Database.Persist.Sql.runMigration">runMigration</hoogle> to run the migration action the `share` statement created.

# Dumping a table

Of course, we want to do more than just create tables. Let's dump the raw contents of the table. We add the `dumpTable` function to just dump the `Tutorial` table via the <hoogle search="Database.Persist.Sql.rawQuery">rawQuery</hoogle> function. There's a collection of imports for it. We'll also change the `runMigration` to <hoogle search="Database.Persist.Sql.runMigrationSilent">runMigrationSilent</hoogle> so we don't get the table creation SQL in our output. Finally, we update `main` to call `dumpTable` after doing the migration. 

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Persist
-- show
import Database.Persist.Sqlite (runSqlite, {-hi-}runMigrationSilent{-/hi-})
-- /show
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
-- show
-- imports for dumpTable
import Database.Persist.Sql (rawQuery)
import Data.Conduit (($$))
import Data.Conduit.List as CL
import Control.Monad.IO.Class (liftIO)
-- /show

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Tutorial
   title    Text
   url      Text
   school   Bool
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    runMigrationSilent migrateTables{-hi-}
    dumpTable{-/hi-}
{-hi-}
dumpTable = rawQuery "select * from Tutorial" [] $$ CL.mapM_ (liftIO . print){-/hi-}
    
```

That was uninspiring. Let's try putting some data into the table. We can use the <hoogle search="Database.Persist.Sql.insert">insert</hoogle> function to add a `Tutorial` instance to the database. This also shows the creation of such a thing, which looks like any other similar thing in Haskell.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Persist
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
-- show
import Database.Persist.Sql (rawQuery, {-hi-}insert{-/hi-})
-- /show
import Data.Conduit (($$))
import Data.Conduit.List as CL
import Control.Monad.IO.Class (liftIO)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Tutorial
   title    Text
   url      Text
   school   Bool
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    runMigrationSilent migrateTables{-hi-}
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True{-/hi-}
    dumpTable
-- /show

dumpTable = rawQuery "select * from Tutorial" [] $$ CL.mapM_ (liftIO . print)
```

That's a little better. We now have a list with one entry. That includes an integer, which is the item id provided by the persistent database interaface, the title and the url of the `Tutorial` we inserted. You don't often have to resort to `rawSql`, and generally shouldn't as it ties you to both SQL and possibly a specific database, but I wanted to show that, should you need to, you can access the facilities of SQL directly.

Now let's try putting a bit more data into the table, and doing a query. We'll move the table creation into a new `buildDb` function so it doesn't clutter up our work in `main`.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Persist
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)

-- rawSql imports.
import Database.Persist.Sql (rawQuery, insert)
import Data.Conduit (($$))
import Data.Conduit.List as CL
import Control.Monad.IO.Class (liftIO)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Tutorial
   title    Text
   url      Text
   school   Bool
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    buildDb
    dumpTable

buildDb = do
    runMigrationSilent migrateTables
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False

-- /show

dumpTable = rawQuery "select * from Tutorial" [] $$ CL.mapM_ (liftIO . print)
```

# Some simple queries

Now that we've got a table with some data in it, let's do some simple queries. We'll also just print the results of the queries, instead of using `rawSql` to get them.

The first query is for the _Basic Haskell_ tutorial. To do this, we're going to use the <hoogle search="Database.Persist.selectList">selectList</hoogle> function from <hoogle>Database.Persist</hoogle>. The first argument to `selectList` is a list of expressions to select a row from a table. Each expression uses a constructor derived from the table declarations, being the name of a table and the name of a column in the table concatenated together, both with an uppercased firstter. In this case, we want the `title` row from the `Tutorial` table, so it's `TutorialTitle`. The operators are drawn from <hoogle>Database.Persist</hoogle>, and are documented there. They are generally the standard haskell comparison operators with a `.` appended. The exception is `!=.` instead of `/=.`, because `/=.` is used for updates. Here, we use `==.` The second argument is a list of options controlling the output list. Again, they are documented in <hoogle>Database.Persist</hoogle>. In this case, we have no entries for that list.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Persist
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (insert)
-- /show
import Data.Conduit (($$))
import Data.Conduit.List as CL
import Control.Monad.IO.Class (liftIO)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Tutorial
   title    Text
   url      Text
   school   Bool
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    buildDb{-hi-}
    basic <- selectList [TutorialTitle ==. "Basic Haskell"] []
    liftIO $ print basic{-/hi-}
    
-- /show
buildDb = do
    runMigrationSilent migrateTables
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False
```

That query found two articles with the given title. Let's add a second condition to the `selectList`, and search for one that is deemed official in the school by having it's `school` value be `True`. Since the conditions are anded together, we just need to add that condition to the list:

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Persist
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (insert)
-- /show
import Data.Conduit (($$))
import Data.Conduit.List as CL
import Control.Monad.IO.Class (liftIO)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Tutorial
   title    Text
   url      Text
   school   Bool
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    buildDb
    basic <- selectList [TutorialTitle ==. "Basic Haskell"{-hi-}, TutorialSchool ==. True{-/hi-}] []
    liftIO $ print basic
    
-- /show
buildDb = do
    runMigrationSilent migrateTables
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False
```

# A second table

Tutorials are usually written by people, and we might well want to keep track of them as well as the tutorials. So let's add a table of authors. A persistent `Author` table declaration is simple. Adding a link from the `Tutorial` table to the `Author` table is more interesting. The `author` entry uses the type `AuthorId`, which is automatically created by the template processor.

Now let's use `selectList` to find the author _Ann Author_. Here we use the second argument to `selectList` with the <hoogle search="Database.Persist.LimitTo">LimitTo</hoogle> constructor to get just the first such author. The other constructors that can be used here - controlling order, starting point for pagination, etc. - are documented in  <hoogle>Database.Persist</hoogle>.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Persist
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (rawQuery, insert)

-- dumpTable imports
import Data.Conduit (($$))
import Data.Conduit.List as CL
import Control.Monad.IO.Class (liftIO)
-- show
share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Author
   name     Text
   email    Text
   deriving Show
Tutorial
   title    Text
   url      Text
   school   Bool
   author   AuthorId
   deriving Show
|]

main = runSqlite ":memory:" $ do{-hi-}
    buildDb{-hi-}
    school <- selectList [AuthorName ==. "Ann Author"] [LimitTo 1]
    liftIO $ print school{-/hi-}

buildDb = do
    runMigrationSilent migrateTables{-hi-}
    school <- insert $ Author "School of Haskell" "school@example.com"
    anne <- insert $ Author "Ann Author" "ann@example.com"{-/hi-}
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True school
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False anne
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True school
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False anne
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False anne
-- /show
```

Now that we've got an author, let's get a list of their tutorials. First, we're going to make the author's email address a uniqueness constaint on the table. We do that by adding a line to the declaration, here `EmailKey email`. That creates a new Haskell constructor `EmailKey`, which by Haskell's rules has to start with an upper case letter. We can then use that with a <hoogle search="Database.Persist">getBy</hoogle> function to fetch a single row.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Persist
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (insert)
import Control.Monad.IO.Class (liftIO)

-- show
share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Author
   name     Text
   email    Text
   EmailKey email
   deriving Show
Tutorial
   title    Text
   url      Text
   school   Bool
   author   AuthorId
   deriving Show
|]

main = runSqlite ":memory:" $ do
    buildDb{-hi-}
    anne <- getBy $ EmailKey "anne@example.com"
    liftIO $ print anne
{-/hi-}
-- /show
buildDb = do
    runMigrationSilent migrateTables{-hi-}
    school <- insert $ Author "School of Haskell" "school@example.com"
    anne <- insert $ Author "Ann Author" "anne@example.com"{-/hi-}
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True school
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False anne
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True school
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False anne
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False anne
```

If you run this code, you'll see that the result is wrapped in a `Maybe`. We can pull it out of that, and then pull the rows `Key` out of that, and use the result to select all the tutorials by this author.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Persist
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (insert)
import Control.Monad.IO.Class (liftIO)


share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Author
   name     Text
   email    Text
   EmailKey email
   deriving Show
Tutorial
   title    Text
   url      Text
   school   Bool
   author   AuthorId
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    buildDb{-hi-}
    anne <- getBy $ EmailKey "anne@example.com"
    case anne of
        Nothing -> liftIO $ print "No such user in database."
        Just row -> do
            tuts <- selectList [TutorialAuthor ==. entityKey row] []
            liftIO $ print tuts{-/hi-}
-- /show
buildDb = do
    runMigrationSilent migrateTables{-hi-}
    school <- insert $ Author "School of Haskell" "school@example.com"
    anne <- insert $ Author "Ann Author" "anne@example.com"{-/hi-}
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True school
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False anne
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True school
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False anne
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False anne
```

# Getting the server to do the work

Those of you familiar with SQL will have noticed that we are extracting data from one table in order to query for it in another table. While there are some cases where this is acceptable, and possibly even desirable, the normal SQL idiom is to do the work in the database. The `persistent` package doesn't do that very well, so we're going to use the <hoogle search="Database.Esqueleto">esqueleto</hoogle> package for that. While `esqueleto` gives you access to more of the power of SQL than `persistent`, `esqueleto` is SQL-only, while `persistent` can be used with NoSQL databases like MongoDB.

The only changes in the code are to replace `import Database.Persist` with `import Database.Esqueleto`, and to write the query out in  `esqueleto` expressions instead of `selectList`.  The <hoogle search="Database.Esqueleto.select">select</hoogle> function is used to return values, just like it is in SQL. The <hoogle search="Database.Esqueleto.from">from</hoogle> function is used to run a function over each row in an implicit join. For selects using just one table, the function's argument is a single variable. Here, we want to use two tables, so we use a tuple of two elements to name them. The <hoogle search="Database.Esqueleto.where_">where\_</hoogle> function then takes an argument that is similar to the expressions in the first argument to `selectList`. Since it's not a list, `&&`. is used to _and_ the expressions together. We also have to provide the table names, well, the arguments to the function passed to `from` in the values, using `^.` to connect the column name constructors to the tables. This allows us to use `AuthorId` on the table instead of `entityKey` on the value from the table. Finally, <hoogle search="Database.Esqueleto.val">val</hoogle> is needed to lift a string value into the SQL expression. And of course, we use `return` to send the resulting argument back to the monad.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
-- show
{-hi-}
import Database.Esqueleto{-/hi-}
-- /show
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (insert)
import Control.Monad.IO.Class (liftIO)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Author
   name     Text
   email    Text
   EmailKey email
   deriving Show
Tutorial
   title    Text
   url      Text
   school   Bool
   author   AuthorId
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    buildDb{-hi-}
    tuts <- select $ from $ \(a, t) -> do
            where_ (a ^. AuthorEmail ==. val "anne@example.com" &&. t ^. TutorialAuthor ==. a ^. AuthorId)
            return t{-/hi-}
    liftIO $ print tuts
-- /show
buildDb = do
    runMigrationSilent migrateTables{-hi-}
    school <- insert $ Author "School of Haskell" "school@example.com"
    anne <- insert $ Author "Ann Author" "anne@example.com"{-/hi-}
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True school
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False anne
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True school
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False anne
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False anne
```

While I've talked about this in terms of the Haskell syntax used, `esqueleto` actually builds an SQL statement and runs it to get the results back. The various functions - except for select, which does the execution - build an SQL statement. Using the `esqueleto` equivalents of grouping and cumulative functions makes that a bit clearer.

For example, we can use <hoogle search="Database.Esqueleto.orderBy">orderBy</hoogle> to control the order of the resulting list. The argument to `orderBy` is a list of ordering expressions. We use <hoogle search="Database.Esqueleto.asc">asc</hoogle> to get an ascending sort on both the `AuthorEmail` and `TutorialTitle`. This example also shows how to `return` just specific elements from the result instead of the entire row, in this case the `AuthorEmail` and `TutorialTitle`.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}

module Main where

import Data.Text (Text)
import Database.Esqueleto
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (insert)
import Control.Monad.IO.Class (liftIO)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Author
   name     Text
   email    Text
   EmailKey email
   deriving Show
Tutorial
   title    Text
   url      Text
   school   Bool
   author   AuthorId
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    buildDb
    tuts <- select $ from $ \(t, a) -> do
            where_ (t ^. TutorialAuthor ==. a ^. AuthorId){-hi-}
            orderBy [asc (a ^. AuthorEmail), asc (t ^. TutorialTitle)]
            return (a ^. AuthorEmail, t ^. TutorialTitle){-/hi-}
    liftIO $ print tuts
-- /show
buildDb = do
    runMigrationSilent migrateTables{-hi-}
    school <- insert $ Author "School of Haskell" "school@example.com"
    anne <- insert $ Author "Ann Author" "anne@example.com"{-/hi-}
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True school
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False anne
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True school
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False anne
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False anne
```

While the list above is interesting, a list of authors and how many tutorials each has written is probably more informative. Lets generate that report. Two new functions here are <hoogle search="Database.Esqueleto.groupBy">groupBy</hoogle> which creates groups, in this case by `authorId`, and <hoogle search="Database.Esqueleto.countRows">countRows</hoogle> to count the rows in each group. We use a `let` to save the group count, represented by the variable `cnt` in the Haskell source, so we can both order the result on it, and return it without calculating it twice. The `orderBy` introduces <hoogle search="Database.Esqueleto.desc">desc</hoogle>, which creates a _descending_ order by `cnt`. Finally, note the type signature on `cnt`: this is required, as otherwise the compiler won't be able to derive the required type.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
{-# LANGUAGE ScopedTypeVariables #-}

module Main where

import Data.Text (Text)
import Database.Esqueleto
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (insert)
import Control.Monad.IO.Class (liftIO)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Author
   name     Text
   email    Text
   EmailKey email
   deriving Show
Tutorial
   title    Text
   url      Text
   school   Bool
   author   AuthorId
   deriving Show
|]

-- show
main = runSqlite ":memory:" $ do
    buildDb
    tuts <- select $ from $ \(a, t) -> do
            where_ (t ^. TutorialAuthor ==. a ^. AuthorId){-hi-}
            groupBy (a ^. AuthorId)
            let cnt = countRows :: SqlExpr (Value Int)
            orderBy [desc cnt]
            return (a ^. AuthorEmail, cnt){-/hi-}
    liftIO $ print tuts
-- /show
buildDb = do
    runMigrationSilent migrateTables
    school <- insert $ Author "School of Haskell" "school@example.com"
    anne <- insert $ Author "Ann Author" "anne@example.com"
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True school
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False anne
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True school
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False anne
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False anne
```

# Modifying the database

Of course, you don't often just load data into a database and then query it. You want to delete data if it goes stale, or possibly modify it.

So let's delete all the tutorials whose title is _Basic Haskell_ that aren't official school tutorials. The first new function here is <hoogle search="Database.Esqueleto.delete">delete</hoogle>.  It's essentially identical to select, except that it's value is `()` instead of a `selectList`. We modify the `select` to exract all unofficial tutorials, and only return the `Title` of each tutorial.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Esqueleto
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (insert)
import Control.Monad.IO.Class (liftIO)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Author
   name     Text
   email    Text
   EmailKey email
   deriving Show
Tutorial
   title    Text
   url      Text
   school   Bool
   author   AuthorId
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    buildDb{-hi-}
    delete $ from $ \t ->
             where_ (t ^. TutorialTitle ==. val "Basic Haskell" &&. t ^. TutorialSchool ==. val False){-/hi-}
    tuts <- select $ from $ \t -> do
            where_ {-hi-}(t ^. TutorialSchool !=. val True)
            return (t ^. TutorialTitle){-/hi-}
    liftIO $ print tuts
-- /show
buildDb = do
    runMigrationSilent migrateTables{-hi-}
    school <- insert $ Author "School of Haskell" "school@example.com"
    anne <- insert $ Author "Ann Author" "anne@example.com"{-/hi-}
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True school
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False anne
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True school
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False anne
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False anne
```

Now let's delete all the tutorials by `anne@example.com`. Since `delete` can only specify a single table in it's `from`, we need to use an SQL `select` expression. This is done with the <hoogle search="Database.Esqueleto.sub_select">sub\_select</hoogle> function. It's arguments are identical to `select`, but it returns an SQL value instead of a value in the monad. Note `sub_select` returns a single value, so the result needs to be a single value. If you want to check for multiple values, you can use <hoogle search="Database.Esqueleto.subList_select">subList\_select</hoogle> with the <hoogle search="Database.Esqueleto.in_">in\_</hoogle> function instead.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Esqueleto
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (insert)
import Control.Monad.IO.Class (liftIO)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Author
   name     Text
   email    Text
   EmailKey email
   deriving Show
Tutorial
   title    Text
   url      Text
   school   Bool
   author   AuthorId
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    buildDb{-hi-}
    delete $ from $ \t -> do
             where_ $ (t ^. TutorialAuthor) ==. 
                      (sub_select $ from $ \a -> do
                                    where_ (a ^. AuthorEmail ==. val "anne@example.com")
                                    return (a ^. AuthorId)){-/hi-}
    tuts <- select $ from $ \t -> do
            where_ (t ^. TutorialSchool !=. val True)
            return (t ^. TutorialTitle)
    liftIO $ print tuts
-- /show
buildDb = do
    runMigrationSilent migrateTables{-hi-}
    school <- insert $ Author "School of Haskell" "school@example.com"
    anne <- insert $ Author "Ann Author" "anne@example.com"{-/hi-}
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True school
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False anne
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True school
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False anne
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False anne
```
Ok, let's end with something simple. Let's just update the database by giving an author a new email address. We're going to correct Anne Author's address from _ann**e**@example.com_ to _ann**a**@example.com_. We're also going to use this example to show how to extract a value from a returned object. Instead returning the value we want, we'll use <hoogle search="Database.Esqueleto.entityVal">entityVal</hoogle> to extract the value, then `authorEmail` to get the value for that field.

``` active haskell
-- /show
{-# LANGUAGE QuasiQuotes, TemplateHaskell, TypeFamilies #-}
{-# LANGUAGE OverloadedStrings, GADTs, FlexibleContexts #-}
module Main where

import Data.Text (Text)
import Database.Esqueleto
import Database.Persist.Sqlite (runSqlite, runMigrationSilent)
import Database.Persist.TH (mkPersist, mkMigrate, persistLowerCase,
       share, sqlSettings)
import Database.Persist.Sql (insert)
import Control.Monad.IO.Class (liftIO)

share [mkPersist sqlSettings, mkMigrate "migrateTables"] [persistLowerCase|
Author
   name     Text
   email    Text
   EmailKey email
   deriving Show
Tutorial
   title    Text
   url      Text
   school   Bool
   author   AuthorId
   deriving Show
|]
-- show
main = runSqlite ":memory:" $ do
    buildDb{-hi-}
    update $ \a -> do
             set a [AuthorEmail =. val "anna@example.com"]
             where_ (a ^. AuthorEmail ==. val "anne@example.com")
    auths <- select $ from $ \a -> return a
    liftIO $ mapM_ (print . authorEmail . entityVal) auths{-/hi-}
-- /show
buildDb = do
    runMigrationSilent migrateTables{-hi-}
    school <- insert $ Author "School of Haskell" "school@example.com"
    anne <- insert $ Author "Ann Author" "anne@example.com"{-/hi-}
    insert $ Tutorial "Basic Haskell" "https://fpcomplete.com/school/basic-haskell-1" True school
    insert $ Tutorial "A monad tutorial" "https://fpcomplete.com/user/anne/monads" False anne
    insert $ Tutorial "Yesod usage" "https://fpcomplete.com/school/basic-yesod" True school
    insert $ Tutorial "Putting the FUN in functors" "https://fpcomplete.com/user/anne/functors" False anne
    insert $ Tutorial "Basic Haskell" "https://fpcomplete/user/anne/basics" False anne
```

# There's more

This has just been an introduction to using `persistent` and `esqueleto` for accessing a database from Haskell. `persistent` has it's own update and delete functions, as well as the ability to use NoSQL back end. `esqueleto` has the SQL operators for computing values, including other cumulative functions. More information about them can be found in the documentation at <hoogle results="1">Database.Persist</hoogle> and <hoogle results="1">Database.Esqueleto</hoogle>.