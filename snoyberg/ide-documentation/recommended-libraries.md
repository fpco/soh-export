# Recommended libraries

Please check out the library list on haskell-lang, available at:

[https://haskell-lang.org/libraries](https://haskell-lang.org/libraries)

This page is __no longer maintained__, and may be out of date.

* * *

## DEPRECATED CONTENT BELOW HERE

FP Complete's development tools provide you with access to many libraries. A full and up-to-date list can be found on [our Haddock documentation index](https://www.fpcomplete.com/hoogle). Many of these libraries are provided as a convenience for users. In general, we recommend using the following well supported libraries:

## General

* [base](https://www.fpcomplete.com/haddocks/base) Base libraries providing common, core functionality
* [process](https://www.fpcomplete.com/haddocks/process) Run and interact with separate system processes
* [resourcet](https://www.fpcomplete.com/haddocks/resourcet) Safe resource allocation with guaranteed resource cleanup, with full exception safety
* [time](https://www.fpcomplete.com/haddocks/time) Time and date handling
* [mwc-random](https://www.fpcomplete.com/haddocks/mwc-random) Random number generation
* [classy-prelude](https://www.fpcomplete.com/haddocks/classy-prelude) Alternate prelude encouraging best practices
    * Built on [mono-traversable](https://www.fpcomplete.com/haddocks/mono-traversable) for data type abstraction

### Core data types

* [bytestring](https://www.fpcomplete.com/haddocks/bytestring) Raw binary data
* [text](https://www.fpcomplete.com/haddocks/text) Textual data
    * [text-icu](https://www.fpcomplete.com/haddocks/text-icu) Bindings to the ICU library, providing many i18n features such as collation
* [vector](https://www.fpcomplete.com/haddocks/vector) Int-indexed arrays
    * [vector-algorithms](https://www.fpcomplete.com/haddocks/vector-algorithms) Nice collection of algorithms working on vectors

### Data structures

* [containers](https://www.fpcomplete.com/haddocks/containers) Basic data structures like Maps and Sets
* [unordered-containers](https://www.fpcomplete.com/haddocks/unordered-containers) Hash-based Map and Set implementation

### Exception handling

* [exceptions](https://www.fpcomplete.com/haddocks/exceptions) Exception handling in transformer stacks
* [enclosed-exceptions](https://www.fpcomplete.com/haddocks/enclosed-exceptions) Proper way to catch all exceptions in an async-exception-friendly manner

### Monad transformers

* [lifted-base](https://www.fpcomplete.com/haddocks/lifted-base) Versions of some core functions, with built-in support for monad transformers
* [mtl](https://www.fpcomplete.com/haddocks/mtl) Monad Transformer Library
    * [transformers](https://www.fpcomplete.com/haddocks/transformers) transformers is a slightly lower-level library providing many of the core features of mtl. In general, you should probably just use mtl, but occassionally you may want some of the transformers modules instead.

### Concurrency and parallelism

* [stm](https://www.fpcomplete.com/haddocks/stm) Software Transactional Memory
* [resource-pool](https://www.fpcomplete.com/haddocks/resource-pool) Resource pools, such as database connections

### Stream processing

* [conduit](https://www.fpcomplete.com/haddocks/conduit) Provides a simple means of producing, transforming, and consuming streams of data with constant memory usage and deterministic resource handling.
    * [conduit-combinators](https://www.fpcomplete.com/haddocks/conduit-combinators) Large collection of combinator functions
    * [classy-prelude-conduit](https://www.fpcomplete.com/haddocks/classy-prelude-conduit) Classy Prelude with Conduit functions included

### Testing

* [hspec](https://www.fpcomplete.com/haddocks/hspec) Test framework, leverage HUnit and QuickCheck

## Filesystem

* [system-filepath](https://www.fpcomplete.com/haddocks/system-filepath) Abstract data type representing paths, and allowing type-safe modifications
    * [system-fileio](https://www.fpcomplete.com/haddocks/system-fileio) A number of convenience functions for operating on the type-safe `FilePath` type

## Data processing

### Structured textual formats

* [aeson](https://www.fpcomplete.com/haddocks/aeson) JSON
* [base16-bytestring](https://www.fpcomplete.com/haddocks/base16-bytestring) Hexadecimal parsing/rendering
* [base64-bytestring](https://www.fpcomplete.com/haddocks/base64-bytestring) Base-64 parsing/rendering
* [html-conduit](https://www.fpcomplete.com/haddocks/html-conduit) Parse HTML data into tree data structure
* [markdown](https://www.fpcomplete.com/haddocks/markdown) Convert Markdown into HTML
* [xml-conduit](https://www.fpcomplete.com/haddocks/xml-conduit) XML processing, with both streaming and tree-based support
* [yaml](https://www.fpcomplete.com/haddocks/yaml) YAML parsing and rendering

### Arbitrary parsing

* [parsec](https://www.fpcomplete.com/haddocks/parsec) Textual parsing library providing user-friendly error messages and position tracking. Can also be used to parse arbitrary token types, not just textual data.
* [attoparsec](https://www.fpcomplete.com/haddocks/attoparsec) High performance binary and textual parsing
* [cereal](https://www.fpcomplete.com/haddocks/cereal) Binary serialization

## Network

### Client

* [aws](https://www.fpcomplete.com/haddocks/aws) Amazon Web Services
* [gravatar](https://www.fpcomplete.com/haddocks/gravatar) Helpers for accessing Gravatar
* [http-client](https://www.fpcomplete.com/haddocks/http-conduit) HTTP client library
    * [http-conduit](https://www.fpcomplete.com/haddocks/http-conduit) conduit adapter for http-client
* [mime-mail](https://www.fpcomplete.com/haddocks/mime-mail) Format and send emails
    * [mime-mail-ses](https://www.fpcomplete.com/haddocks/mime-mail-ses) Special support for Amazon SES

## Web development

* [email-validate](https://www.fpcomplete.com/haddocks/email-validate) Validate email addresses for correctness, and canonicalize into a standard format.
* [hamlet](https://www.fpcomplete.com/haddocks/hamlet) HTML templating
* [hjsmin](https://www.fpcomplete.com/haddocks/hjsmin) Javascript minification
* [yesod](https://www.fpcomplete.com/haddocks/yesod) Yesod Web Framework for type safe, high performance, RESTful applications. For more information, see [the Yesod website](http://www.yesodweb.com)
* [warp](https://www.fpcomplete.com/haddocks/warp) High performance web server (used by Yesod, but also usable on its own)

## Database

* [persistent](https://www.fpcomplete.com/haddocks/persistent) Type safe object relational mapping
    * [persistent-postgresql](https://www.fpcomplete.com/haddocks/persistent-postgresql)
    * [persistent-sqlite](https://www.fpcomplete.com/haddocks/persistent-sqlite)
    * [esqueleto](https://www.fpcomplete.com/haddocks/esqueleto) Support for more complex SQL queries in Persistent
* [postgresql-simple](https://www.fpcomplete.com/haddocks/postgresql-simple) Allows running SQL queries directly against a PostgreSQL database

## Concurrent programming

* [async](https://www.fpcomplete.com/haddocks/async) Well designed async functionality
* [stm](https://www.fpcomplete.com/haddocks/stm) Software Transactional Memory

## Command line processing

* [optparse-applicative](https://www.fpcomplete.com/haddocks/optparse-applicative) Command line argument parsing

## Cryptography

* [cryptohash](https://www.fpcomplete.com/haddocks/cryptohash) Cryptographic hash functions
    * [cryptohash-conduit](https://www.fpcomplete.com/haddocks/cryptohash-conduit) conduit adapter