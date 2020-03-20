# Working with databases using Groundhog

The object-oriented languages have had high-level Object-Relational Mapping (ORM) tools for a long time. ORM is a layer over the relational storage which allows to manipulate DB using code idiomatic for the language. Groundhog brings this idea to Haskell being an ADT-Relational Mapping library. In a type-safe manner it handles data serialization, schema migration, supports complex queries, and increases portability by providing database-independent API, just to name a few. It ensures that your database, datatypes, and generated queries are coherent, making it easier to follow the DRY principle (Don't Repeat Yourself).

One of the Groundhog goals is not to restrict neither datatypes, nor database schema. Your Haskell datatypes may have type parameters, or several constructors. Your relational schema may have composite primary keys, indexes, references across several schemas. Groundhog will bind them together with minimal configuration.

## Let's get started!

After the language extensions and imports there are our data definitions.

``` haskell
{-# LANGUAGE GADTs, TypeFamilies, TemplateHaskell, QuasiQuotes, FlexibleInstances, StandaloneDeriving #-}
import Control.Monad.IO.Class (liftIO)
import Database.Groundhog.TH
import Database.Groundhog.Sqlite

data Customer = Customer {
  customerName :: String,
  phone :: String
} deriving Show
data Product = Product {
  productName :: String,
  quantity :: Int,
  customer :: DefaultKey Customer
}
deriving instance Show Product
```

This schema demonstrates plain relationship. The ordered Product references Customer through its default key. Groundhog supports many types of keys, but by default it chooses autoincremented integer key. Instead of a universal <code>Show (Key v r)</code> instance there are separate instances for each entity, so we have to use standalone deriving instances to create the <code>Show</code> instance for <code>Product</code>.

Then Template Haskell assisted by QuasiQuotation comes into play.

``` yaml
mkPersist defaultCodegenConfig [groundhog|
- entity: Customer               # Name of the datatype
  constructors:
    - name: Customer
      fields:
        - name: customerName
          # Set column name to "name" instead of "customerName"
          dbName: name
      uniques:
        - name: NameConstraint
          fields: [customerName] # Inline format of list
- entity: Product
|]
```

Function <code>mkPersist</code> creates all instances and definitions for you datatypes so that they can be used mapped to the tables. The datatypes you want to map to database are described in the list of entities. Groundhog analyzes the structure of each datatype and automatically creates its description which is used for codegeneration. The defaults in the description can be overridden explicitly through configuration written in YAML. It is a more readable superset of JSON. Each new item in YAML's multiline lists starts with hyphen+space. In the associative arrays keys and values are separated by colon+space.


For example, to change a field column name we access its constructor <code>Customer</code> at first. To do this we  access list with constructors by its "constructors" key. Then we choose an element from the list by its "name" key. In a similar way we access a field by its "name" in the "fields" list. Adding new parts to the description (e.g., unique constraints) follows the same convention as overriding the default values of description. In this example we leave the default values for Product, so a single line is enough.

Now let's connect to Sqlite and see Groundhog in action!

``` active haskell
-- show
-- /show
{-# LANGUAGE GADTs, TypeFamilies, TemplateHaskell, QuasiQuotes, FlexibleInstances, StandaloneDeriving #-}
import Control.Monad.IO.Class (liftIO)
import Database.Groundhog.TH
import Database.Groundhog.Sqlite

data Customer = Customer {
  customerName :: String,
  phone :: String
} deriving Show
data Product = Product {
  productName :: String,
  quantity :: Int,
  customer :: DefaultKey Customer
}
deriving instance Show Product

mkPersist defaultCodegenConfig [groundhog|
- entity: Customer               # Name of the datatype
  constructors:
    - name: Customer
      fields:
        - name: customerName
          # Set column name to "name" instead of "customerName"
          dbName: name
      uniques:
        - name: NameConstraint
          fields: [customerName] # Inline format of list
- entity: Product
|]
-- show
main :: IO ()
main = withSqliteConn ":memory:" $ runDbConn $ do
  runMigration defaultMigrationLogger $ do
    migrate (undefined :: Customer)
    migrate (undefined :: Product)
  johnKey <- insert $ Customer "John Doe" "0123456789"
  get johnKey >>= liftIO . print
  insert $ Product "Oranges" 3 johnKey
  insert $ Product "Apples" 5 johnKey
  janeKey <- insert $ Customer "Jane Doe" "9876543210"
  insert $ Product "Oranges" 4 janeKey
  johnOrders <- select $ (CustomerField ==. johnKey)
    `orderBy` [Asc ProductNameField]
  liftIO $ putStrLn $ "Products for John: " ++ show johnOrders
-- /show
```

The <code>main</code> starts from opening connection using bracket-like function <code>withSqliteConn</code>. The function <code>runDbConn</code> runs the code block within a transaction. If an exception is thrown within transaction, it will be rolled back. The next step is migration that compares the existing schema in DB with expected one and produces a script to alter DB. It can create schema from scratch, or alter it. The list of entities to be migrated is passed to <code>runMigration</code>. Enumerating all entities explicitly is a good style, but Groundhog tracks references and it would behave the same even if there were only <code>Product</code> because it has key of <code>Customer</code>.

The insert returns value of the autoincremented key <code>johnKey</code> for the newly inserted customer John Doe. We use it to <code>get</code> the data back from DB and as a foreign key for Products. Function <code>select</code> receives a combination of condition and other options, namely ordering, limit, and offset.

## Types and classes

There are classes for three main abstractions: mapping datatypes to a tables, mapping datatype field to columns, and backend. 

### PersistEntity

Class <code>PersistEntity</code> is necessary to map an entity to a table. The CRUD operations and migration work only with instances of this class. It has data family <code>Field</code> which is used in queries to refer to columns. <code>ProductNameField</code> and <code>QuantityField</code> we saw above are constructors of its instances. Also it has data family <code>Key</code> that defines what kinds of keys we have.

### Mapping fields to columns

Each database can have its own format of data. It is inconvenient to work with this format or with values of the original field types. So we have an intermediate type <code>PersistValue</code> for uniform data representation. This sum type stores numbers, strings, dates, etc. The information is converted in directions field <-> PersistValue <-> column.

However, conversion between fields and PersistValues is not straightforward because Groundhog has embedded datatypes corresponding to several columns. Some fields may require access to other tables, so we need a monad in the signature. There are four typeclasses whose conversion functions types capture these specifics and they form a hierarchy from the general <code>PersistField</code> to the most restricted <code>PrimitivePersistField</code>.

<table>
	<tr><td></td><td>Single column</td><td>Multiple columns</td></tr> 
	<tr><td>Pure</td><td><code>PrimitivePersistField</code></td><td><code>PurePersistField</code></td></tr>
	<tr><td>Non-pure&nbsp;</td><td><code>SinglePersistField</code></td><td><code>PersistField</code></td></tr>
</table> <br/>

Here we show functions from <code>PersistField</code>. Similar functions from the other typeclasses have simpler signatures without monad or with a single <code>PersistValue</code> instead of a list.

<code>toPersistValues :: PersistBackend m => a -> m ([PersistValue] -> [PersistValue])</code> returns a difference list.
<code>fromPersistValues :: PersistBackend m => [PersistValue] -> m (a, [PersistValue])</code> creates a value taking several elements from the list and returns it together with list leftovers.

Let's define instances for an enum:

``` haskell
data Cutlery = Spoon | Fork | Knife | Spork
  deriving (Enum, Show, Read)

instance PersistField Cutlery where
  persistName _ = "Cutlery"
  -- primToPersistValue are from Database.Groundhog.Generic
  toPersistValues = primToPersistValue
  fromPersistValues = primFromPersistValue
  -- is stored as string column, not nullable, no default value, no reference
  dbType _ = DbTypePrimitive DbString False Nothing Nothing

instance PrimitivePersistField Cutlery where
  toPrimitivePersistValue p a = toPrimitivePersistValue p $ show a
  fromPrimitivePersistValue p x = read $ fromPrimitivePersistValue p x
```

Now <code>WeekDay</code> can be stored in database and queried in expressions like <code>MyField ==. Friday</code>.

These classes are handy to do conversion from one type to another:

``` haskell
myKey <- insert user
myInt <- toSinglePersistValue myKey >>= fromSinglePersistValue
liftIO $ print (myInt :: Int)
```

### Backend

Class <code>PersistBackend</code> defines what operations backends support. Its functions generate SQL, do marshall and unmarshalling, and talk to a database. It is the top-level interface for manipulating data in DB, so the most application calls to Groundhog will be through it. The inserts, selects, migrations and other operations are executed within its instance. There is a separate instance for each backend. This both keeps code simple and gives high performance because we can do database-specific adjustments. In the section <a href="#queries">Queries and data manipulation</a> there is description of <code>PersistBackend</code> functions and how to use them.

## Keys and references

In a database schema the tables may reference each other in many different ways. There can be the integer autoincremented primary key, composite unique keys, etc. All kinds of foreign keys are represented by data family <code>Key</code> which belongs to class <code>PersistEntity</code>. To reference another entity, your field can store one of its keys (preferably) or the entity itself. The first parameter of <code>Key</code> is its entity, the second is a phantom type which describes the key and helps to ensure that the keys match.

There are two other type families for keys which repeat types of certain <code>Key</code> GADT constructors.

- <code>AutoKey</code> is a type for the autoincremented primary key. If an entity does not have such key, it will be <code>()</code>. In particular, this allows to insert entities regardless of their primary key with the same function <code>insert :: PersistEntity v => v -> m (AutoKey v)</code>.
- <code>DefaultKey</code> helps to choose a key in a situation when an entity is stored directly and has multiple keys. Also it may simplify the type signatures.

Consider the following examples:

``` haskell
data LinkedList a = LinkedList a (Maybe (Key (LinkedList a) BackendSpecific))      
```

The type <code>BackendSpecific</code> means that it key is defined by backend. For the current SQL backends we have autoincremented integer primary key. If we had MongoDB it would be ObjectId. For the other types of keys (non-autoincremented primary key, unique constraints and unique indexes) the type looks like <code>Unique SomeUniqueConstraintName</code>. If the mapping does not override the default key, this example can be rewritten as:

``` haskell
data LinkedList a = LinkedList a (Maybe (DefaultKey (LinkedList a))      
```

Alternatively, instead of its Key, your field may store the entity directly:

``` haskell
data LinkedList a = LinkedList a (Maybe (LinkedList a))
```

In this case when you <code>insert</code> your linked list, the list from the field will be inserted with function <code>insertByAll</code>. Using <code>insertByAll</code> prevents creating duplicate records. It returns id of an existing record which has a matching unique constraint, or inserts the entity and returns its id. Note that if an inner entity has matching unique constraint but some other fields differ, it will still be considered the same and no updates or inserts on it will be performed. To have cleaner semantics, it is recommended to store keys.


Here we see the most commonly used configuration parameters for the keys and references:

The datatype <code>Album</code> does not have autoincremented primary key. Instead, we create our own natural primary key - album name. Its definition in YAML consists of two parts. At first, we define a unique set of fields with type primary. Then we put the name of unique set into section <code>keys</code>. This is necessary to use the constraint in the foreign keys. It results in creating a phantom type <code>AlbumName</code> and another constructor for <code>Key</code> with the corresponding instances. If it had more columns, a composite key would be created. Uniques are not treated as keys by default to avoid polluting the namespace with the phantom types. In a similar way we could declare our key AlbumName to be a constraint or a unique index.

We create <code>AlbumName</code> manually because we use it in <code>Track</code> before Template Haskell does its work. If we used it after calling mkPersist, the generated name would be already available.

``` haskell
data Album = Album {
  albumName :: String,
  albumDescr :: String
} deriving (Eq, Show)
data Track = Track {
  trackAlbum :: Key Album (Unique AlbumName),
  trackName :: String
}
deriving instance Eq Track
deriving instance Show Track

-- It is phantom datatype of the AlbumName unique key.
-- Usually they are generated by Template Haskell, but we define
-- it here manually to use in Track datatype
data AlbumName v where
  AlbumName :: AlbumName (UniqueMarker Album)

mkPersist defaultCodegenConfig [groundhog|
definitions:
  - entity: Album
    autoKey: null # Disable creation of the autoincrement integer key
    keys:
      - name: AlbumName # Matches name of one of the uniques
        default: true
    constructors:
      - name: Album
        uniques:
          - name: AlbumName
            type: primary
            fields: [albumName]
  - entity: Track
    autoKey:
      # Optional constructor name in the Key data family instance
      constrName: TrackAutoKey
    constructors:
      - name: Track
        keyDbName: trackId
        fields:
          - name: trackAlbum
            reference:
              onDelete: cascade
              onUpdate: restrict
|]

```

The <code>Track</code> demonstrates how to change constructor name and column name (<code>keyDbName</code>) for the default primary key. It also shows how we can tweak clauses ON DELETE and ON UPDATE. Note that many options are not listed in the examples. The complete configuration format description is available at <a href="http://hackage.haskell.org/packages/archive/groundhog-th/latest/doc/html/Database-Groundhog-TH.html">Database.Groundhog.TH</a>.

``` active haskell
-- show
-- /show
{-# LANGUAGE GADTs, TypeFamilies, TemplateHaskell, QuasiQuotes, FlexibleInstances, StandaloneDeriving #-}
import Control.Monad
import Control.Monad.IO.Class (liftIO)
import Database.Groundhog.Core (UniqueMarker)
import Database.Groundhog.TH
import Database.Groundhog.Sqlite

data Album = Album {
  albumName :: String,
  albumDescr :: String
} deriving (Eq, Show)
data Track = Track {
  trackAlbum :: Key Album (Unique AlbumName),
  trackName :: String
}
deriving instance Eq Track
deriving instance Show Track

-- It is phantom datatype of the AlbumName unique key.
-- Usually they are generated by Template Haskell, but we define
-- it here manually to use in Track datatype
data AlbumName v where
  AlbumName :: AlbumName (UniqueMarker Album)

mkPersist defaultCodegenConfig [groundhog|
definitions:
  - entity: Album
    autoKey: null # Disable creation of the autoincrement integer key
    keys:
      - name: AlbumName # Matches name of one of the uniques
        default: true
    constructors:
      - name: Album
        uniques:
          - name: AlbumName
            type: primary
            fields: [albumName]
  - entity: Track
    autoKey:
      # Optional constructor name in the Key data family instance
      constrName: TrackAutoKey
    constructors:
      - name: Track
        keyDbName: trackId
        fields:
          - name: trackAlbum
            reference:
              onDelete: cascade
              onUpdate: restrict
|]
-- show
main :: IO ()
main = withSqliteConn ":memory:" $ runDbConn $ do
  let imagineAlbum = Album "Imagine" "Recorded and released in 1971. Side two"
  runMigration defaultMigrationLogger $ do
    migrate (undefined :: Album)
    migrate (undefined :: Track)
  insert imagineAlbum
  let tracks = ["Gimme Some Truth", "Oh My Love", "How Do You Sleep?", "How?", "Oh Yoko!"]       
  let imagineKey = extractUnique imagineAlbum
  mapM_ insert $ map (Track imagineKey) tracks
  tracks' <- select $ (TrackAlbumField ==. imagineKey)
    `orderBy` [Asc AutoKeyField]
  liftIO $ mapM_ print tracks'
-- /show
```

After establishing connection and migration we insert album Imagine. To insert the tracks we need a key for this album. Function <code>insert</code> does not return it because Album has a custom primary key. Instead, we use <code>extractUnique</code> from class <code>IsUniqueKey</code> that creates a key by extracting key fields from the entity. Finally, we select the tracks ordering them by their primary key. A datatype AutoKeyField behaves like a regular field and can be used in conditions or ordering clause.


As the unique keys contain columns of a specific constructor, they cannot be created for sum types.

## Queries and data manipulation

We've already inserted some records and queried. Now let's look more closely at the Groundhog API.

### Inserting

There is a family of insert functions. They differ with regard to keys.

``` haskell
insert :: PersistEntity v => v -> m (AutoKey v)
```
inserts a new record to a database and return its autogenerated key or (). If there are any unique constraints violations, it will throw an error.
``` haskell
insert_ :: PersistEntity v => v -> m ()
```
also inserts a new record, but does not return the primary key. It is noticeably faster on Sqlite and MySQL because they need an extra query to retrieve it.
``` haskell
insertBy :: (PersistEntity v, IsUniqueKey (Key v (Unique u)))
         => u (UniqueMarker v)
         -> v
         -> m (Either (AutoKey v) (AutoKey v))
```
checks if the unique key our entity matches already existing one. If there is such record, it returns <code>Left oldKey</code>, otherwise, it inserts our entity and returns <code>Right newKey</code>. For example, having already inserted <code>imagineAlbum</code> from the last example we could safely run <code>insertBy AlbumName imagineAlbum</code>. It would see that there is already an album with this name return <code>Left ()</code> because album table does not have autoincremented primary key.
``` haskell
insertByAll :: PersistEntity v
            => v
            -> m (Either (AutoKey v) (AutoKey v))
```
checks all existing constraints before inserting the entity. If there are any matches, it returns <code>Left oldKey</code>. Note that if our entity matches several records on different constraints, it will return a key for an arbitrary record.

### Modification

There are two functions for modifying a record in a database:

``` haskell
replace :: (PersistEntity v, PrimitivePersistField (Key v BackendSpecific))    
        => Key v BackendSpecific -> v -> m ()
```
completely replaces the record. It receives an autogenerated id. You can replace values created by the same or different constructors.

``` haskell
update :: (PersistEntity v, EntityConstr v c)
       => [Update (PhantomDb m) (RestrictionHolder v c)]
       -> Cond (PhantomDb m) (RestrictionHolder v c)
       -> m ()
```
Update receives the list of fields to be updated and condition and the chosen fields. An <code>Update</code> is created with <code>=.</code> operator

``` haskell
-- double quantity column for record with the given id
update [QuantityField =. toArith QuantityField * 2]
  $ AutoKeyField ==. k
-- rename customer
update [CustomerNameField =. "John Doe"]
  $ CustomerNameField ==. "John" ||.
    CustomerNameField `like` "%Smith%"
```

### Selecting data

Groundhog has many functions to query a database. Some of them are more specialized, but simpler to use, while others offer high flexibility.

``` haskell
get :: (PersistEntity v, PrimitivePersistField (Key v BackendSpecific))
    => Key v BackendSpecific -> m (Maybe v)
```
is one of the simplest functions for querying. It gets an entity by its primary autogenerated key. As an entity may be absent, it returns Maybe.
``` haskell
getBy :: (PersistEntity v, IsUniqueKey (Key v (Unique u)))
      => Key v (Unique u) -> m (Maybe v)
```
is similar to <code>get</code>, but accepts a unique key.
``` haskell
count :: (PersistEntity v, Constructor c)
      => Cond (PhantomDb m) (RestrictionHolder v c) -> m Int
```
returns total number of records satisfying the condition. The parameter <code>PhantomDb m</code> of <code>Cond</code> ensures that for example, a PostgreSQL-specific operator which occurs in condition can be run only in PostgreSQL database.
``` haskell
countAll :: PersistEntity v => v -> m Int
```
returns total number of records of all constructors. We cannot use <code>count</code> for this as the condition can refer to fields of only one constructor, The parameter <code>v</code> is used only to infer type of the record.
``` haskell
select :: (PersistEntity v, Constructor c, HasSelectOptions opts (PhantomDb m) (RestrictionHolder v c))
       => opts -> m [v]
```
has more complicated type, but it is simple to use. It returns a list of entities which satisfy a condition. The parameter <code>opts</code> is condition with optional ordering, limit, and offset. The convenience class <code>HasSelectOptions</code> helps to make the the clauses optional and ensure that we don't duplicate them.

``` haskell
select $ QuantityField >=. (5 :: Int)
select $ (10 :: Int) <. toArith QuantityField * 2
     &&. (ProductNameField ==. "Apples"
     ||. ProductNameField ==. "Melons")
select $ (CustomerField ==. johnKey)
  `orderBy` [Asc ProductNameField] `offsetBy` 20 `limitTo` 10
```
The condition operators are similar to Haskell ones, but with a dot at the end. They work both with fields, constants, and complex expressions, e.g., arithmetics. A downside of this flexibility is that the arguments may need an explicit type annotations. If you want to run a query without condition, just pass <code>CondEmpty</code>.

``` haskell
selectAll :: PersistEntity v => m [(AutoKey v, v)]
```
is to <code>select</code> as <code>count</code> is to <code>countAll</code>. It selects from all constructor tables and does not have a condition.
``` haskell
project :: (PersistEntity v, Constructor c,
            Projection p (PhantomDb m) (RestrictionHolder v c) a,
            HasSelectOptions opts (PhantomDb m) (RestrictionHolder v c))    
        => p -> opts -> m [a]
```
is the most powerful query function. While <code>select</code> returns a list of entities, with <code>project</code> you choose which columns and expressions to select. Parameter <code>p</code> describes what things you would like to get from a table and defines type of result <code>a</code>.
						<br/><br/>
<table> 
	<tr><td width="20%">Projection</td><td width="20%">Result</td><td width="60%">Description</td></tr> 
	<tr><td><code>Field v c a</code></td><td><code>a</code></td><td>Regular entity field</td></tr>
	<tr><td><code>SubField v c a</code></td><td><code>a</code></td><td>Field of embedded entity. It is created by <code>~></code> operator.</td></tr>
	<tr><td><code>Expr db r a</code></td><td><code>a</code></td><td>Arbitrary expression (see section below)</td></tr>
	<tr><td><code>AutoKeyField v c</code></td><td><code>AutoKey v</code></td><td>Field for autogenerated key</td></tr>
	<tr><td><code>c (ConstructorMarker v)</code></td><td><code>v</code></td><td>Constructor phantom type gives its entity</td></tr>
	<tr><td><code>u (UniqueMarker v)</code></td><td><code>Key v (Unique u)</code></td><td>Unique key phantom type can be used as an shortcut for fields it consists of</td></tr>
</table>
						<br/>
						A tuple of projections is also a projection and will give you a tuple of corresponding results. Now there are instances for tuples of arity up to five. If you need to select more fields, you can nest the tuples.
``` haskell
project (AutoKeyField, (NameField, MyTupleField ~> Tuple2_1Selector))    
  $ someCond :: m [(AutoKey SomeEntity, (String, Double))]
```
It may an alternative to <code>select</code> if you want to get a key too.
``` haskell
project (AutoKeyField, UserConstructor)
  $ AutoKeyField ==. k :: m [(AutoKey SomeEntity, SomeEntity)]
```

### Expressions

You may have noticed that the parameters on the left and right sides of <code>==.</code> and other operators are very different. They are fields, constants, arithmetics, and more. This is possible because all they are instances of the class <code>Expression</code>. With its power we can build a DSL that closely matches capabilities of SQL expressions in a type-safe way.

``` haskell
-- | Instances of this type can be converted to 'UntypedExpr'.
-- It is useful for uniform manipulation over fields, constant values, etc.
class Expression db r a where
  toExpr :: a -> UntypedExpr db r
```
The parameter <code>a</code> is for the value that can be lifted. The <code>db</code> constraints the databases where this expression can be used. For example, if expression is rendered into SQL with an operator which exists only in PostgreSQL, we would be able to use it only for PostgreSQL connections. Parameter <code>r</code> is used to restrict other properties. Usually it is <code>RestrictionHolder v c</code> - container for value type and its constructor. Together they uniquely define a table we use. It is important to prevent querying one table using columns from others.

The expressions are used in WHERE conditions, projections and updates. They are rendered into a part of SQL query (or several parts for embedded datatypes) with placeholders and a list of <code>PersistValue</code>. The functions and infix operators can be created as follows:

``` haskell
like :: (SqlDb db, QueryRaw db ~ Snippet db, ExpressionOf db r a String)    
     => a -> String -> Cond db r
-- operator priority, operator string, parameters
like a b = CondRaw $ operator 40 " LIKE " a b

upper :: (SqlDb db, QueryRaw db ~ Snippet db, ExpressionOf db r a String)
      => a -> Expr db r String
upper a = Expr $ function "upper" [toExpr a]
```

The constructor <code>CondRaw</code> brings expressions with SQL to conditions. The operators priority parameter is used to decide when to put parentheses. The <code>Expr</code> is just a wrapper around <code>UntypedExpr</code>. It carries a type parameter which keeps the type of expressions.


The <code>ExpressionOf db r a String</code> is a shorthand for <code>(Expression db r a, Unifiable a String)</code>. Class <code>Unifiable</code> ensures type safety of the expressions. It unifies the values of similar types. Expression <code>Field v c String</code> matches with <code>String</code> or <code>SubField v c String</code>. But it would not match <code>Field v c Int</code>.


``` haskell
project ("username: " `append` upper UserNameField)
  $ lower UserNameField `like` "%smith%"
project ((toArith NumberField + 1) * 2)
  $   upper FirstNameField ==. upper UserNameField
  ||. (toArith NumberField * 5) >. (25 :: Int)
update (EmailField =. upper EmailField) (AutoKeyField ==. k)
```

would be rendered into the following SQL queries (actually they would have placeholders instead of constants).

``` SQL
SELECT 'username: ' || upper(username) FROM mytable WHERE lower(username) LIKE '%smith%'
SELECT (number + 1) * 2 FROM mytable WHERE upper(firstname) = upper(username) OR number * 5 > 25
UPDATE mytable SET email = upper(email) WHERE id = k
```

## Embedded data types

The embedded data types are a way to map a single field to multiple columns. Entity table does not reference them but embeds their columns. A tuple is a characteristic example of such type. Fields of embedded types may include other embedded types or references, pretty much like entity fields. Their subfields can be accessed individually or as a whole.

The same embedded type may occur several times in one record. We would like both to avoid column name clashes and keep the name configuration optional. To achieve this, by default the column names are prefixed by the name of the outer field. For example, <code>myField :: (Int, (Int, Int))</code> will be mapped to columns <code>myField#val0</code>, <code>myField#val1#val0</code>, <code>myField#val1#val1</code>. The names of the inner columns can be overridden. If any of the embedded field names is changed, its other fields will lose their prefixes too.

We define entity <code>Company</code> and an embedded type <code>Address</code> which is kept inside a tuple along with the address coordinates.

``` haskell
data Company = Company {
  name :: String,
  location :: ((Double, Double), Address)
} deriving (Eq, Show)
data Address = Address {
  city :: String,
  zipCode :: String,
  street :: String
} deriving (Eq, Show)
```

The embeddeds may have only a single constructor, so unlike entities they don't have <code>constructors</code> parameter. To change the default values of column name and column type we use field key <code>embeddedType</code>. There are slight differences between configuring regular fields in the definition itself and modifying them through <code>embeddedType</code>. In the first case changes are applied at compile time. But at the second case since we cannot call function <code>dbType</code> for a field due to Template Haskell phase restrictions, the changes are applied at runtime. Also because of this we have to use default dbName column name to access a subfield as with <code>zip_code</code> below.

``` yaml
mkPersist defaultCodegenConfig [groundhog|
definitions:
  - entity: Company
    constructors:
      - name: Company
        fields:
          - name: location
            embeddedType:               # If a field has an embedded type you can access its subfields.
              - name: val1
                embeddedType:
                  - name: city          # Just a regular list of fields. However, note that you should use default dbNames of embedded
                    dbName: hq_city
                    type: varchar       # Change type of column
                  - name: zip_code      # Here we use embedded dbName (zip_code) which differs from the name used in Address definition (zipCode)
                    dbName: hq_zipcode
                  - name: street
                    dbName: hq_street
  - embedded: Address
    fields:                             # The syntax is the same as for constructor fields. Nested embedded types are allowed.
      - name: zipCode
        dbName: zip_code                # Change column name.
 |]
```

After all instances are created by TH, we can run a program.

``` active haskell
-- show
-- /show
{-# LANGUAGE GADTs, TypeFamilies, TemplateHaskell, QuasiQuotes, FlexibleInstances, StandaloneDeriving #-}
import Control.Monad.IO.Class (liftIO)
import Database.Groundhog.TH
import Database.Groundhog.Sqlite

data Company = Company {
  name :: String,
  location :: ((Double, Double), Address)
} deriving (Eq, Show)
data Address = Address {
  city :: String,
  zipCode :: String,
  street :: String
} deriving (Eq, Show)

mkPersist defaultCodegenConfig [groundhog|
definitions:
  - entity: Company
    constructors:
      - name: Company
        fields:
          - name: location
            embeddedType:               # If a field has an embedded type you can access its subfields.
              - name: val1
                embeddedType:
                  - name: city          # Just a regular list of fields. However, note that you should use default dbNames of embedded
                    dbName: hq_city
                    type: varchar       # Change type of column
                  - name: zip_code      # Here we use embedded dbName (zip_code) which differs from the name used in Address definition (zipCode)
                    dbName: hq_zipcode
                  - name: street
                    dbName: hq_street
  - embedded: Address
    fields:                             # The syntax is the same as for constructor fields. Nested embedded types are allowed.
      - name: zipCode
        dbName: zip_code                # Change column name.
 |]
-- show
main = withSqliteConn ":memory:" $ runDbConn $ do
  let address = Address "Sunnyvale" "18144" "El Camino Real"
      coords = (37.355362, -122.016633)
      company = Company "Cyberdyne Systems" (coords, address)
  runMigration defaultMigrationLogger $ migrate company
  insert company
  cs <- select
    $   LocationField ~> Tuple2_0Selector ==. coords
    &&. LocationField ~> Tuple2_1Selector ~> ZipCodeSelector ==. "18144"    
  liftIO $ print cs
-- /show
```

In condition of select we both access a tuple as a whole for coordinates, and drill down to a string zip code value. The subfields created by <code>~></code> operator are expressions and can be used everywhere. The tuple fields have names like <code>val0</code>, <code>val1</code>, and so on. Their selectors have corresponding names of form TupleN_MSelector where N is number of elements in a tuple, and M is number of the selected element. This flexibilty allows high-level and concise statements that affect multiple columns like 
``` haskell
update [MyAddress =. newAddress] $ MyAddress ==. oldAddress
```