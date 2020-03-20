# Querying an existing database

# Introduction

While most database programming tutorials walk you through creating, populating, querying and modifying a database, as that covers the subject, a far more common assignment is to extract data from an existing database in order to create a report, load it into some other application, etc.

The <hoogle>persistent</hoogle> database package uses template haskell to translate *entity definitions* into Haskell data declarations that can be mapped to SQL types.  In this tutorial, we'll look at a (relatively simple) real-world schema, and show how to translate it to a persistent entity definition. We will then check the definition to insure that nothing changes, and use it to extract the table in CSV format. Finally, we'll provide a summary of the features available when creating an entity definition.

# The Schema

The schema we're going to work with is a schema from a now obsolete application, but it was in production for over a decade. It was used for providing a list of web pages, noting when they were added, when someone followed them and how many times they were followed, and the last time it was verified that the URL in question actually worked. Of course, it kept the URL and a description to go with it. There are a number of sites on the web for which such a table might be useful, though they would typically want more information than this.

``` sql
create table hotlist (
	id          int not null ,		-- The index counter
	count		int default 0 ,		-- and the click counter
	added		timestamp default now() ,		-- when it was added
	followed	timestamp ,		-- last time it was followed
	checked		timestamp ,		-- last time it was verified
	description	varchar not null,			-- What the link is to
	URL		    varchar not null,			-- And how to get there
	primary key (id)
	) ;
```

# The entity defintion

In order to access this table with persistent, we need an entity definition, which will be used by the template haskell quasiQuoter <hoogle>persistLowerCase</hoogle> to create the appropriate Haskell data types. The code for the above table looks like:

``` haskell
share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
Link sql=hotlist
   count Int default=0 sqltype=int
   added UTCTime default=now()
   followed UTCTime Maybe
   checked UTCTime Maybe
   description Text
   url Text
   UniqueUrl url
   deriving Show
|]
```

The input to <hoogle>persistLowerCase</hoogle> is a set of entity definitions.  An *entity* corresponds to an SQL table.  The various Haskell data types needed to deal with the table will also be created.

 <hoogle>persistLowerCase</hoogle> translates the Haskell names into SQL by turning upper case to lower case and inserting an underscore before them if they aren't the first character, effectively turning CamelCase into camel&#95;case. The alternative is to use <hoogle>persistUpperCase</hoogle>, which leaves the names intact.

 <hoogle>mkPersist</hoogle> creates the Haskell data types from the output of <hoogle>persistLowerCase</hoogle>. The <hoogle>sqlSettings</hoogle> causes it to generate code for an SQL database.  The set of available back ends changes with time, though at least <hoogle>Database.Persist.MongoDB</hoogle> for MongoDB should be available.

 <hoogle>mkMigrate</hoogle> creates a description of the SQL expected on the back end based on the output of <hoogle>persistLowerCase</hoogle>. That can be passed to a number of functions that query the back end, and adjust the database as needed - including creating any missing tables.

Finally, <hoogle>share</hoogle> runs the list of processes in it's first argument over it's second argument.

## Entities

An entity starts with an name that starts with an uppercase letter (`Link` in our example above), and includes all the following lines that are indented. Python programmers will feel right at home with these.

### Attributes

The name  starts with an uppercase letter because it's going to be used as a constructor for a Haskell record composed of values from the following lines. After the name are attributes for the constructor. In this case, the `sql=hotlist` causes the resulting data types to be associated with the table `hotlist`.  The complete list of entity attributes is:

<table border=1 align=center>
<tr><th>Attribute<th>Affect
<tr><td>sql=<i>name</i><td>The name of the table for this entity
<tr><td>id=<i>name</i><td>The name of the id column for this entity 
<tr><td>json<td>create toJSON and fromJSON instances for this entity
</table>

### Id field

Every entity type must have an *id* field. By default, it's name is `id`, it has type `integer` and it will be unique. That's a fairly common thing to find in an SQL table, though the name may be different. In an existing table, if the name isn't `id`, you can use the `id=*name*` attribute to set it to *name*.

As with other fields, you get a constructor called `*Entity*Id`. However, unlike other fields, this one is not part of the `*Entity*` type. That would make it difficult to construct an entity to insert into the database. Instead, `*Entity*Id` values are returned from the database by queries, and when you insert entities into the database. You can then use them to get the actual entity back from the database with <hoogle search="Database.Persist.get">get</hoogle>.

## Fields

The indented lines, down to the one that starts `URL`, create field definitions. A *field* corresponds to a column in the table. These all start with lower case letters. Each field will have two visible names, both constructed from the entity and field name.  Both consist of the field name appended to the entity name. One is the a Haskell field name for this field in the entity, so it starts with lower case (i.e. - `linkCount`). The other is a constructor that will be used in selects, and thus starts with upper case (`LinkCount`).

### Types

The fields in the entity definition are pretty much in one-to-one correspondence with the columns in the table, except that `id` is missing.  Each name is followed by the type for that field. This should match the type in the table column of that name, so that Haskell's type checking will work.

The Haskell types recognized by <hoogle>persistent</hoogle>, and the PostGreSQL types they map to, are:

<table border=1 align=center>
<tr><th>Haskell<th>SQL
<tr><td>Text<td>VARCHAR
<tr><td>ByteString<td>BYTEA
<tr><td>Int<td>INT8
<tr><td>Double<td>DOUBLE PRECISION
<tr><td>Rational<td>NUMERIC(22, 12)
<tr><td>Bool<td>BOOLEAN
<tr><td>Day<td>DATE
<tr><td>TimeOfDay<td>TIME
<tr><td>UTCTime<td>TIMESTAMP
<tr><td>ZonedTime<td>TIMESTAMP+TIMEZONE
</table>

### Attributes

After the type there is an optional list of attributes for each field. Except for `Maybe`, they generally are used to control the connection to the database, adopting it for an existing database, adopting what gets used, or changing the database table.

<table border=1 align=center>
<tr><th>Attribute<th>Affect
<tr><td>sql=<i>name</i><td>The name of the column for this field
<tr><td>sqltype=<i>type</i><td>The sql type of this column
<tr><td>Maybe<td>Wrap this field in a <hoogle>Maybe</hoogle> to handle SQL `NULL`s
<tr><td>MigrationOnly<td>This disabled the creation of the Haskell type for fields needed by other applications but not by Haskell.
<tr><td>SafeToRemove<td>This will cause the column to be removed from the database.
</table>

## And the rest

The next to last line `UniqueUrl url` - adds a uniqueness constraint to the table. It also creates a type constructor `UniqueUrl which takes arguments of the type of `url`. This can be used in queries to fetch the single `Link` from the table that matches the given `url`. You can use multiple fields here, so this could be, for instance, `UniqueCounters count description` which would create a constructor `UniqueCounters` that takes an integer (`count`) and a string (`description`).  They are distinguished by starting with an upper-case letter.

The last line `deriving Show` adds the `Show` typeclass to the list of typeclasses that are automatically derived for the `Link` type.

## Examining it.

So lets go through the fields for the `Link` type.

The `description` and `url` fields are simple fields of `Text` type. We also add the `UniqueUrl` constraint, which will mark `url` as unique.

`followed` and `checked` are `UTCTime` fields, which means they are going to be an sql `TIMESTAMP`. Both of them are also flagged as `Maybe`, so they can have `NULL` values, and the Haskell values will either be `Nothing` (for `NULL`'s), or `Just UTCTime`.

`added` is also a `UTCTime`, so the column should be an sql `TIMESTAMP`. No `Maybe` attribute, so it's a `NOT NULL` column. The `default=now()` attribute means that it will be set to the time of insertion if we don't specify one.

Finally, the `count` field is an Int defaulting to 0. The attribute `sqltype` forces the type to be an sql `int`, as `persist` will otherwise try and make it an sql `INT8`, which is larger than what the table currently uses. So we need the attribute to get it to use the proper type.

## Using it

Now, we can watch how this works with our actual table.

``` active haskell
{-# LANGUAGE QuasiQuotes, TypeFamilies, GeneralizedNewtypeDeriving, TemplateHaskell,
             OverloadedStrings, GADTs, FlexibleContexts, ScopedTypeVariables #-}
import Database.Persist
import Database.Persist.Postgresql
import Database.Persist.TH
import Control.Monad.IO.Class (liftIO)
import Data.Time (UTCTime)
import Data.Text (Text)
import Control.Monad.Logger

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
Link sql=hotlist
   count Int default=0 sqltype=int
   added UTCTime default=now()
   followed UTCTime Maybe
   checked UTCTime Maybe
   description Text
   url Text
   URL url
   deriving Show
|]

connStr = "dbname=tutorial host=tutorial-db.fpcomplete.com user=tutorial password=tutorial port=5432"

main :: IO ()
main = runStdoutLoggingT $ withPostgresqlPool connStr 10 $ \pool ->
     liftIO $ flip runSqlPersistMPool pool $ do
        printMigration migrateAll
        res  :: [Entity Link] <- selectList [] [LimitTo 1] 
        liftIO $ print res
```

We'll get to the connection setup later, but for now, let's talk about the output. You should see a string of SQL DDL statements to modify tables, then a Haskell list with one (messy) element in it. **Don't panic** - we haven't modified the table. <hoogle>printMigration</hoogle> takes the table descriptions created by <hoogle>mkMigrate</hoogle> and tells you what it needs to do to the database in order to make the tables the way it thinks they ought to be.

What you should see are `update`s to the `count` and `added` columns to set `NULL`s to the default value, and then make them `not null`. Those are arguably fixing bugs in the schema, as those two columns shouldn't have `NULL`s in them. We can get rid of these messages by adding the `Maybe` attribute to those fields. The type on `count` is changed to `int` as well, which is a no-op, as it was `int` in the schema.

It also thinks that the `URL` and `description` columns should be of type `VARCHAR`. For PostgreSQL, this is ok - they were `Text`, which is a synonym for `VARCHAR`. To get rid of these, add an `sqltype=text` attribute to the fields.

Finally, it adds the `u_r_l` constraint for uniqueness, which we've added to the type for pedagogical purposes.

# Translating to CSV 

We now want to get CSV output from the table. Since we're going to do this for production, now is a good time to deal with the connection setup.

``` haskell
main :: IO ()
main = withPostgresqlPool connStr 10 $ \pool -> 
     flip runSqlPersistMPool pool $
```

 <hoogle>withPostgresqlPool</hoogle> creates a pool of connections to the the database specified by `connStr`. In this case, 10 of them.  It passes that pool to it's last argument, which is the lambda that runs <hoogle>runSqlPersistMPool</hoogle> using that argument, running it's last argument with the connections take from a pool. In this case - since we're just going to run one query and then exit - the pool probably isn't necessary. If you were going to run a web server or some other application that would be making concurrent requests, then the pool would be the right choice, and you should see how to do that.

For best performance, we're going to use a conduit to process the results of the query. This avoids loading the entire table in memory at once, allowing arbitrary sized tables to be processed and downloaded. If you're not familiar with conduits, read the [overview](https://www.fpcomplete.com/school/advanced-haskell/conduit-overview).

``` haskell
        selectSource [] [] $$
            CL.map toRow =$
            (writeHeaders defCSVSettings >> fromCSV defCSVSettings) =$
            sinkHandle stdout
```

The change to the persist code is the use of <hoogle>selectSource</hoogle> instead of <hoogle>selectList</hoogle>. <hoogle>selectSource</hoogle> has the same arguments, but returns a conduit `source` instead of a list.

We then use <hoogle>Data.Conduit.List.map</hoogle> to process each row with `toRow`, which we'll show you in a bit.  <hoogle>writeHeaders</hoogle> is then used to add a CSV header to the rows, and finally <hoogle>sinkHandle</hoogle> writes the resulting data to `stdout`.

We have to provide our own conversion from SQL results to CSV rows, which is what `toRow` does:

``` haskell
            toRow (Entity _ Link {..}) = fromList
                [("count" :: Text, pack $ show linkCount),
                 ("added", pack $ show linkAdded),
                 ("followed", pack $ show linkFollowed),
                 ("checked", pack $ show linkChecked),
                 ("description", linkDescription),
                 ("url", linkUrl)]

```

`toRow` uses the `RecordWildCards` extension to easiy get the various linkXxx fields from the record. Those are then converted to text, and become the second element of a tuple whose first element is the name for this column in the CSV rows. Each row in the database will be converted into a list of such tuples, and then <hoogle>fromList</hoogle> will convert that into a CSV row.

Putting it all together, we get:

``` active haskell
{-# LANGUAGE QuasiQuotes, TypeFamilies, GeneralizedNewtypeDeriving, TemplateHaskell,
             OverloadedStrings, GADTs, FlexibleContexts, ScopedTypeVariables, RecordWildCards #-}
import qualified Data.Conduit.List as CL
import Data.Conduit
import Data.Conduit.Binary
import Data.CSV.Conduit
import Database.Persist
import Database.Persist.Postgresql
import Database.Persist.TH
import Control.Monad.IO.Class (liftIO)
import Data.Time (UTCTime)
import Data.Text (Text, pack)
import Data.Map (fromList)
import System.IO (stdout)
import Control.Monad.Logger (runStdoutLoggingT)

share [mkPersist sqlSettings] [persistLowerCase|
Link sql=hotlist
   count Int default=0 sqltype=int
   added UTCTime default=now()
   followed UTCTime Maybe
   checked UTCTime Maybe
   description Text
   url Text
   URL url
   deriving Show
|]

connStr = "dbname=tutorial host=tutorial-db.fpcomplete.com user=tutorial password=tutorial port=5432"

main :: IO ()
main = runStdoutLoggingT $ withPostgresqlPool connStr 10 $ \pool ->
     liftIO $ flip runSqlPersistMPool pool $
        selectSource [] [] $$
            CL.map toRow =$
            (writeHeaders defCSVSettings >> fromCSV defCSVSettings) =$
            sinkHandle stdout
        where
            toRow (Entity _ Link {..}) = fromList
                [("count" :: Text, pack $ show linkCount),
                 ("added", pack $ show linkAdded),
                 ("followed", pack $ show linkFollowed),
                 ("checked", pack $ show linkChecked),
                 ("description", linkDescription),
                 ("url", linkUrl)]
```

This version drops the `migration` facilities so we don't get that in our output.

# Feedback

If you have questions about this tutorial, you can discuss it on the [Google+ Forum](https://plus.google.com/100162554869434148021/posts/MuoZMDz1Tv7).


