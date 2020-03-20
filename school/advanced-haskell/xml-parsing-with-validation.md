# XML Processing with Error checking

# Introduction

One of the most basic XML parsing tasks in web applications is reformatting XML data for display on a web page - basically using it as a markup language. For such chores, silently ignoring an invalid element is a minor problem. You get a missing or garbled row of data in output that's being read by a human, who can figure out what it should say, or ignore it, or even get the data from another source if it's the bit they're looking for.

If that describes your application, you might consider using `xml-conduit` package, which has less complicated combinators. It is described in in our [HTML parsing tutorial](https://www.fpcomplete.com/school/starting-with-haskell/libraries-and-frameworks/text-manipulation/tagsoup), which uses the same set of functions and operators, parsing the document with the HTML parser instead of an XML parser.

Other applications of XML, like using it to hold program configuration information or data being collected in the field, are less tolerant of errors. A program misconfiguration can cause no end of problems, over and above missing data. If your application is processing financial data, that error could turn into a loss of money. If it's handling medical data - or worse yet, controlling medical equipment - then those errors could turn into a loss of life.

XML provides tools for dealing with such conditions, and this tutorial is going to explore some of those.

## The application

Let's build a simple financial application. It will process transactions made at a single ATM during the course of the day. A transaction consists of an account, the type of transaction (either a deposit or a withdrawal), and the amount. A real-world version of this would have a database back end, and update the accounts in the database as it processed them, but we're going to skip that. We'll just print the Haskell data structures in the document so we can verify that the XML was processed correctly.

## The software

We're going to be using the Haskell XML library `HXT`, as it provides the validation and processing features we need. The processing paradigm is <hoogle>arrows</hoogle>, which are very powerful, but unfortunately very complicated. This means we're going to be writing a lot of functions that are arrows, but we'll introduce the features we need.

## A note on DTD's, schemas and XML

If you're not familiar with schemas at all, then the nickel tour is that they describe the structure of a valid document. For each tag, they list the valid attributes, contents and types for the tag. An XML document is said to be _well-formed_ if it is syntactically correct XML, and the open and close tags pair up properly. An XML document is _valid_ if it conforms to a schema. A document that isn't well-formed is called _ill-formed_, though _not XML_ is also correct. An XML document that doesn't conform to a schema is _invalid_.

The oldest format for schemas - predating XML - is the [Document Type Definition](http://www.w3schools.com/dtd/), or _DTD_. A modern, more expressive format is the [W3C XML Schema](http://www.w3.org/TR/2001/REC-xmlschema-0-20010502/), which is XML. We're working with [RelaxNG](http://relaxng.org/tutorial-20011203.html), because it has a [compact form](http://relaxng.org/compact-tutorial-20030326.html) based on regular expressions that is easy for humans to read and edit as well as an XML form that is simpler than a W3C XML Schema, while having similar expressive power.

The tool [trang](http://www.thaiopensource.com/relaxng/trang.html) can be used to convert between these formats, and will warn you when you try using features of one that won't work in the other. It also has the ability to read XML files and create a schema for them.

# The schema

In this case, we're fortunate enough to get to define schema we want to use. It's not at all unusual to be either given the schema, or worse, not be given the schema, and expected to figure out what is and isn't valid.

The XML file will pretty much directly reflect our data structures. The RNC schema is:

```
start = element transactions {
   (element deposit { info } | element withdrawal { info })+
}

info = ( attribute account { xsd:positiveInteger },
         attribute amount { xsd:decimal { pattern = "\d+.\d\d" }},
         text )
```

The root element is `transactions`. It contains a list of one or more (as denoted by the `+`) elements, each of which is either a `withdrawal` or (the `|`) a `deposit` element. Both those elements contain `info`, meaning they have `account` and `amount` atributes, and allow arbitrary text as content. The `account` attribute is a positive integer, and the `amount` attribute is a decimal number of dollars and cents.

## In XML

As mentioned, this can be translated into a RelaxNG XML schema, and in fact that is what is required by `HXT`. It looks like:

```
<?xml version="1.0" encoding="UTF-8"?>
<grammar xmlns="http://relaxng.org/ns/structure/1.0" datatypeLibrary="http://www.w3.org/2001/XMLSchema-datatypes">
  <start>
    <element name="transactions">
      <oneOrMore>
        <choice>
          <element name="deposit">
            <ref name="info"/>
          </element>
          <element name="withdrawal">
            <ref name="info"/>
          </element>
        </choice>
      </oneOrMore>
    </element>
  </start>
  <define name="info">
    <attribute name="account">
      <data type="positiveInteger"/>
    </attribute>
    <attribute name="amount">
      <data type="decimal">
        <param name="pattern">\d+.\d\d</param>
      </data>
    </attribute>
    <text/>
  </define>
</grammar>
```

## W3C XML Version

If your software requires it, you can also translate it into a W3C XML Schema. While we won't use it, this looks like:

```
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema" elementFormDefault="qualified">
  <xs:element name="transactions">
    <xs:complexType>
      <xs:choice maxOccurs="unbounded">
        <xs:element ref="deposit"/>
        <xs:element ref="withdrawal"/>
      </xs:choice>
    </xs:complexType>
  </xs:element>
  <xs:element name="deposit">
    <xs:complexType mixed="true">
      <xs:attributeGroup ref="info"/>
    </xs:complexType>
  </xs:element>
  <xs:element name="withdrawal">
    <xs:complexType mixed="true">
      <xs:attributeGroup ref="info"/>
    </xs:complexType>
  </xs:element>
  <xs:attributeGroup name="info">
    <xs:attribute name="account" use="required" type="xs:positiveInteger"/>
    <xs:attribute name="amount" use="required">
      <xs:simpleType>
        <xs:restriction base="xs:decimal">
          <xs:pattern value="\d+.\d\d"/>
        </xs:restriction>
      </xs:simpleType>
    </xs:attribute>
  </xs:attributeGroup>
</xs:schema>
```

As you can see, those are both harder to read than the RNC format. However, this is the last time we'll be using them. The RelaxNG schema will be in each example, but it won't change, so you can ignore it.

# Processing

The pieces are almost in place to start processing the data. But first, let's look at the sample data:

```
<?xml version="1.0" encoding="utf-8"?>
<transactions>
  <deposit account="1" amount="156.83">Payday!</deposit>
  <withdrawal account="1" amount="50.00" />
  <deposit account="2" amount="218.12" />
  <withdrawal account="2" amount="20.00" />
  <deposit account="3" amount="123.45" />
  <withdrawal account="3" amount="100.00" />
  <deposit account="4" amount="1022.51" />
</transactions>
```

The first example should give us back this input - give or take some formatting.

## Reading the file

Parsing the XML - with or without validation - is done by the <hoogle>readDocument</hoogle> function. It expects two arguments. One is a list of options for parsing XML, and the other is a file name to read. Since we can't provide command line arguments, we hardcode the file name:

``` haskell
readDocument [withRelaxNG "transactions.rng"] "transactions.xml"
```

For now, we'll just write the data out in XML again. That uses <hoogle>writeDocument</hoogle>, which also takes two arguments, a list of options and a file name. In this case, we're going to use the file name `""` to cause output to go to standard out:

``` haskell
writeDocument [withIndent True, withOutputEncoding utf8] ""
```

The two options give us nicely formatted output in UTF-8.

Both of these functions return arrows, so can be combined with the <hoogle>>>></hoogle> operator to produce a third arrow. This is just function composition (<hoogle>(.)</hoogle>) with arrows, so input values are passed to the first arrow, and the output of that to the second.


``` active haskell
{-# START_FILE Main.hs #-}
import Text.XML.HXT.Core
import Text.XML.HXT.RelaxNG

main :: IO ()
main = do
  runX $ readDocument [withRelaxNG "transactions.rng"] "transactions.xml"
         >>> writeDocument [withIndent True, withOutputEncoding utf8] ""
  return ()

{-# START_FILE transactions.rng #-}
<?xml version="1.0" encoding="UTF-8"?>
<grammar xmlns="http://relaxng.org/ns/structure/1.0" datatypeLibrary="http://www.w3.org/2001/XMLSchema-datatypes">
  <start>
    <element name="transactions">
      <zeroOrMore>
        <choice>
          <element name="deposit">
            <ref name="info"/>
          </element>
          <element name="withdrawal">
            <ref name="info"/>
          </element>
        </choice>
      </zeroOrMore>
    </element>
  </start>
  <define name="info">
    <attribute name="account">
      <data type="positiveInteger"/>
    </attribute>
    <attribute name="amount">
      <data type="decimal">
        <param name="pattern">\d+.\d\d</param>
      </data>
    </attribute>
    <text/>
  </define>
</grammar>
{-# START_FILE transactions.xml #-}
<?xml version="1.0" encoding="utf-8"?>
<transactions>
  <deposit account="1" amount="156.83">Payday!</deposit>
  <withdrawal account="1" amount="50.00" />
  <deposit account="2" amount="218.12" />
  <withdrawal account="2" amount="20.00" />
  <deposit account="3" amount="123.45" />
  <withdrawal account="3" amount="100.00" />
  <deposit account="4" amount="1022.51" />
</transactions>
```

## A list of `Transaction`'s

Now that we can parse the XML transaction list, let's turn it into a list of Haskell `Transaction`'s. A transaction is:


``` haskell
import Data.Fixed (Centi)
import Data.Text (Text, pack)

type Comment = Maybe Text
data Transaction = Deposit !Text !Centi !Comment
                 | Withdrawal !Text !Centi !Comment
                 deriving Show
```

We introduce a type alias for `Comment`, which may or may not be present.

For the actual data type, we use `Centi` instead of `Float` or `Double` because  dollars and cents are a `Centi` values, and an obvious choice if we want to avoid the various issues with floating point representations. The account number could be an `Integer`, but making it `Text` leaves room for future changes. We also force all the values in the record to be strict with the `!` prefix.

## Selecting our elements

HXT processes XML documents as lists of nodes. It provides a set of functions - specifically arrows - that transform an input list of nodes into an output list of nodes. Our select function will use those functions to produce a list of just the elements we want to process.

The document element of an XML document has as it's children information about the document as well as the parsed XML document, which we can extract with <hoogle search="Control.Arrow.ArrowTree.getChildren">getChildren</hoogle>. We want only the transactions element from those, so we use <hoogle search="Text.XML.HXT.Arrow.XmlArrow.hasName">hasName</hoogle> to select that. The `transactions` children are the elements of interest. However, it also has text children from the whitespace between the elements, so we use <hoogle search="Text.XML.HXT.Arrow.XmlArrow.isElem">isElem</hoogle> to get only the XML elements from the childre. So we select only the elements:

``` haskell
select = getChildren >>> hasName "transactions" >>> getChildren >>> isElem
```
Since we're using a validated document, we could replace the `hasName "transactions"` with `isElem` and get the same result. We could also leave it off entirely, but that would then process all the children of the document root through the second `getChildren`, rather than just the element we want.

## Constructing the data type we want

Since `HXT` is going to provide a string for the tag name, we need a function to map those to the appropriate constructors:

``` haskell
make "deposit" = Deposit
make "withdrawal" = Withdrawal
make trans = error $ "Invalid transaction type: " ++ trans
```

The final case isn't strictly necessary, as the validation will insure that it's never invoked. However, leaving the function definition partial is a sufficiently bad idea that we'll provide it anyway.

## Getting Haskell data

In order to turn the elements into Haskell data, we're going to use the features enabled by the `Arrows` extension:

``` haskell
transform = proc el -> do
  trans <- getName -< el
  account <- getAttrValue "account" -< el
  amount <- getAttrValue "amount" -< el
  comment <- withDefault (Just . pack ^<< getText <<< getChildren) Nothing -< el
  returnA -< (make trans) (pack account) (read amount) comment
```

Here the Arrows keyword `proc`, like `do`, provides syntactic sugar but builds a function that is an arrow. It takes arguments simular to a lambda, in this case one, `el`, and passes it to the defined arrow. Inside the arrow, the Arrows syntax `-<` can be used to process values through arrows.  If one of the internal arrows should fail for some reason, then processing stops.

This arrow uses <hoogle search="Text.XML.HXT.Arrow.XmlArrow.getName">getName</hoogle> to get the name of the transaction, then <hoogle search="Text.XML.HXT.Arrow.XmlArrow.getAttrValue">getAttrValue</hoogle> to get the account and amount attribues.

To get the comment, we use <hoogle search="Text.XML.HXT.Arrow.XmlArrow.getText">getText</hoogle> on the nodes children (provided by <hoogle search="Control.Arrow.ArrowTree.getChildren">getChildren</hoogle>) to extract the text. That is then passed to <hoogle search="Data.Text.pack">pack</hoogle>  and <hoogle>Just</hoogle> to get a `Maybe Text` value, using the <hoogle>^&lt;&lt;</hoogle> combinator to combine `Just . pack` with an arrow. The <hoogle>withDefault</hoogle> function is used to return the text if available, or `Nothing` if there is no text available.

The last step is to use the previously defined `make` function to create a transaction from that data, and pass it to <hoogle search="Control.Arrow.returnA">returnA</hoogle> to get it back to our arrow monad.

## Putting it all together

With all that in place, all we need to do is modify `main` to pass the data through `select` and then `transform` to get a list of `Transaction`s, which will will them <hoogle>mapM_</hoogle> <hoogle>print</hoogle> over to demonstrate the results.

``` active haskell
{-# START_FILE Main.hs #-}
{-# LANGUAGE Arrows #-}

import Text.XML.HXT.Core
import Text.XML.HXT.RelaxNG
import Data.Text (Text, pack)
import Data.Fixed (Centi)

type Comment = Maybe Text
data Transaction = Deposit !Text !Centi !Comment
                 | Withdrawal !Text !Centi !Comment
                 deriving Show

make "deposit" = Deposit
make "withdrawal" = Withdrawal
make trans = error $ "Invalid transaction type: " ++ trans

select = getChildren >>>  hasName "transactions" >>> getChildren >>> isElem

transform = proc el -> do
  trans <- getName -< el
  account <- getAttrValue "account" -< el
  amount <- getAttrValue "amount" -< el
  comment <- withDefault (Just . pack ^<< getText <<< getChildren) Nothing -< el
  returnA -< (make trans) (pack account) (read amount) comment

main :: IO ()
main = do
  tl <- runX $ readDocument [withRelaxNG "transactions.rng"] "transactions.xml"
               >>> select >>> transform
  mapM_ print tl

{-# START_FILE transactions.rng #-}
<?xml version="1.0" encoding="UTF-8"?>
<grammar xmlns="http://relaxng.org/ns/structure/1.0" datatypeLibrary="http://www.w3.org/2001/XMLSchema-datatypes">
  <start>
    <element name="transactions">
      <zeroOrMore>
        <choice>
          <element name="deposit">
            <ref name="info"/>
          </element>
          <element name="withdrawal">
            <ref name="info"/>
          </element>
        </choice>
      </zeroOrMore>
    </element>
  </start>
  <define name="info">
    <attribute name="account">
      <data type="positiveInteger"/>
    </attribute>
    <attribute name="amount">
      <data type="decimal">
        <param name="pattern">\d+.\d\d</param>
      </data>
    </attribute>
    <text/>
  </define>
</grammar>
{-# START_FILE transactions.xml #-}
<?xml version="1.0" encoding="utf-8"?>
<transactions>
  <deposit account="1" amount="156.83">Payday!</deposit>
  <withdrawal account="1" amount="50.00" />
  <deposit account="2" amount="218.12" />
  <withdrawal account="2" amount="20.00" />
  <deposit account="3" amount="123.45" />
  <withdrawal account="3" amount="100.00" />
  <deposit account="4" amount="1022.51" />
</transactions>
```
# Handling errors

So now let's see what happens if we have an error in the data. For example, what happoens if a  `withdrawal` tag has been shorted to `withdraw`, possibly from a data transmission or entry error.

## With validation

Here's the exact same code as above, with the change in the data:

``` active haskell
{-# START_FILE Main.hs #-}
{-# LANGUAGE Arrows #-}

import Text.XML.HXT.Core
import Text.XML.HXT.RelaxNG
import Data.Text (Text, pack)
import Data.Fixed (Centi)

type Comment = Maybe Text
data Transaction = Deposit !Text !Centi !Comment
                 | Withdrawal !Text !Centi !Comment
                 deriving Show

make "deposit" = Deposit
make "withdrawal" = Withdrawal
make trans = error $ "Invalid transaction type: " ++ trans

select = getChildren >>>  hasName "transactions" >>> getChildren >>> isElem

transform = proc el -> do
  trans <- getName -< el
  account <- getAttrValue "account" -< el
  amount <- getAttrValue "amount" -< el
  comment <- withDefault (Just . pack ^<< getText <<< getChildren) Nothing -< el
  returnA -< (make trans) (pack account) (read amount) comment

main :: IO ()
main = do
  tl <- runX $ readDocument [withRelaxNG "transactions.rng"] "transactions.xml"
               >>> select >>> transform
  mapM_ print tl

{-# START_FILE transactions.rng #-}
<?xml version="1.0" encoding="UTF-8"?>
<grammar xmlns="http://relaxng.org/ns/structure/1.0" datatypeLibrary="http://www.w3.org/2001/XMLSchema-datatypes">
  <start>
    <element name="transactions">
      <zeroOrMore>
        <choice>
          <element name="deposit">
            <ref name="info"/>
          </element>
          <element name="withdrawal">
            <ref name="info"/>
          </element>
        </choice>
      </zeroOrMore>
    </element>
  </start>
  <define name="info">
    <attribute name="account">
      <data type="positiveInteger"/>
    </attribute>
    <attribute name="amount">
      <data type="decimal">
        <param name="pattern">\d+.\d\d</param>
      </data>
    </attribute>
    <text/>
  </define>
</grammar>
{-# START_FILE transactions.xml #-}
<?xml version="1.0" encoding="utf-8"?>
<transactions>
  <deposit account="1" amount="156.83">Payday!</deposit>
  <withdrawal account="1" amount="50.00" />
  <deposit account="2" amount="218.12" />
  <withdrawal account="2" amount="20.00" />
  <deposit account="3" amount="123.45" />
  <withdraw account="3" amount="100.00" />
  <deposit account="4" amount="1022.51" />
</transactions>
```

As expected, the validation caught the error, and printed it for us with enough information to identify the problem.

## Without validation

So what happens if we don't validate? Well, this version of the code disables validation (highlighted in the code) in the `readDocument` options list:

``` active haskell
{-# START_FILE Main.hs #-}
{-# LANGUAGE Arrows #-}

import Text.XML.HXT.Core
import Text.XML.HXT.RelaxNG
import Data.Text (Text, pack)
import Data.Fixed (Centi)

type Comment = Maybe Text
data Transaction = Deposit !Text !Centi !Comment
                 | Withdrawal !Text !Centi !Comment
                 deriving Show

make "deposit" = Deposit
make "withdrawal" = Withdrawal
make trans = error $ "Invalid transaction type: " ++ trans

select = getChildren >>>  hasName "transactions" >>> getChildren >>> isElem

transform = proc el -> do
  trans <- getName -< el
  account <- getAttrValue "account" -< el
  amount <- getAttrValue "amount" -< el
  comment <- withDefault (Just . pack ^<< getText <<< getChildren) Nothing -< el
  returnA -< (make trans) (pack account) (read amount) comment

main :: IO ()
main = do
  tl <- runX $ readDocument [{-hi-}withValidate no{-/hi-}] "transactions.xml"
               >>> select >>> transform
  mapM_ print tl
{-# START_FILE transactions.xml #-}
<?xml version="1.0" encoding="utf-8"?>
<transactions>
  <deposit account="1" amount="156.83">Payday!</deposit>
  <withdraw account="1" amount="50.00" />
  <deposit account="2" amount="218.12" />
  <withdrawal account="2" amount="20.00" />
  <deposit account="3" amount="123.45" />
  <withdrawal account="3" amount="100.00" />
  <deposit account="4" amount="1022.51" />
</transactions>

```

In this case, the XML parser passed the XML through just fine, and the error caused processing to stop with no error message. While we could catch this case - and  others - by adding code similar to the handling of comments to every arrow in `proc`, that would result in noticably longer, more opaque code. The ad hoc nature of such additions would lead to more errors and more fragile code.

I suggest you play with the above two examples, feeding them various errors to see how the two handle the different cases. Errors that make the XML ill-formed as well as invalid, and errors in the data - non-numeric `account` values, or non-dollar types for `amount` - are all worth investigating.

## Catching errors

For use cases where errors **must** be handled, we normally want to provide special handling for errors - for instance, logging them so the correct data can be entered, and the source of the errors corrected.

The first thing we need is to decide how to handle errors. Passing the XML document instance created by `HXT` to the handler provides lots of flexibility. For our case, we're just going to print the file name:

``` haskell
handleError = getAttrValue a_source >>> arrIO (putStrLn . ("Error in document: " ++))
```

This is another arrow. We just extract the file name, and then print a message. We use <hoogle>arrIO</hoogle> to invoke an IO function as part of an arrow.

Since we want run either the error handler, **or** the standard processing, we need to convert processing - printing the transactions - to an arrow. Since it's in an arrow, this function wil be called on each transaction, so all we have to do is invoke <hoogle>print</hoogle> as an arrow. <hoogle>arrIO</hoogle> comes in handy there as well.

``` haskell
process = arrIO print
```

The last step is to fix the arrow in main to use the two new arrows:

``` haskell
main = do
  runX $ readDocument [withRelaxNG "transactions.rng"] "transactions.xml"
         >>> ((select >>> transform >>> process) `orElse` handleError)
  return ()
```

This  uses the <hoogle search="Control.Arrow.ArrowIf.orElse">orElse</hoogle> combinator to run the error handler should the document processing fail. We need a `return ()` because the type of the arrow is `IO [()]`.

So, tying that together:

``` active haskell
{-# START_FILE Main.hs #-}
{-# LANGUAGE Arrows #-}
import Text.XML.HXT.Core
import Text.XML.HXT.RelaxNG
import Data.Fixed (Centi)
import Data.Text (Text, pack)


type Comment = Maybe Text
data Transaction = Deposit !Text !Centi !Comment
                 | Withdrawal !Text !Centi !Comment
                 deriving Show

make "deposit" = Deposit
make "withdrawal" = Withdrawal
make trans = error $ "Invalid transaction type: " ++ trans

select = getChildren >>>  hasName "transactions" >>> getChildren >>> isElem

transform = proc el -> do
  trans <- getName -< el
  account <- getAttrValue "account" -< el
  amount <- getAttrValue "amount" -< el
  comment <- withDefault (Just . pack ^<< getText <<< getChildren) Nothing -< el
  returnA -< (make trans) (pack account) (read amount) comment

process = arrIO print

handleError = getAttrValue a_source >>> arrIO (putStrLn . ("Error in document: " ++))

main :: IO ()
main = do
  runX $ readDocument [withRelaxNG "transactions.rng"] "transactions.xml"
         >>> ((select >>> transform >>> process) `orElse` handleError)
  return ()

{-# START_FILE transactions.rng #-}
<?xml version="1.0" encoding="UTF-8"?>
<grammar xmlns="http://relaxng.org/ns/structure/1.0" datatypeLibrary="http://www.w3.org/2001/XMLSchema-datatypes">
  <start>
    <element name="transactions">
      <zeroOrMore>
        <choice>
          <element name="deposit">
            <ref name="info"/>
          </element>
          <element name="withdrawal">
            <ref name="info"/>
          </element>
        </choice>
      </zeroOrMore>
    </element>
  </start>
  <define name="info">
    <attribute name="account">
      <data type="positiveInteger"/>
    </attribute>
    <attribute name="amount">
      <data type="decimal">
        <param name="pattern">\d+.\d\d</param>
      </data>
    </attribute>
    <text/>
  </define>
</grammar>
{-# START_FILE transactions.xml #-}
<?xml version="1.0" encoding="utf-8"?>
<transactions>
  <deposit account="1" amount="156.83">Payday!</deposit>
  <withdrawal account="1" amount="50.00" />
  <deposit account="2" amount="218.12" />
  <withdrawal account="2" amount="20.00" />
  <deposit account="3" amount="123.45" />
  <withdraw account="3" amount="100.00" />
  <deposit account="4" amount="1022.51" />
</transactions>
```

As you can see, `errorHandler` gets called with the document instance and prints the file name. Since the file name is fixed, it won't ever do anything else here. You might verify that you get that behavior with all the various errors you tried above.

# Bidirectional conversion

If you're converting to/from XML, then the <hoogle>HXT</hoogle> toolbox includes tools for building *pickler/unpickler* pairs for converting to and from XML with about as much code as it takes to write one of the pair. This is documented [on the Haskell wiki](http://www.haskell.org/haskellwiki/HXT/Conversion_of_Haskell_data_from/to_XML)

# Feedback

[Discuss](https://plus.google.com/100162554869434148021/posts/QLaGuo8Sy9m) this tutorial in the FP Complete Google+ Community.


