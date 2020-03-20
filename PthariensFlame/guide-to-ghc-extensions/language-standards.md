# Language Standards

These extensions are not really extensions, but instead control which Haskell standard GHC attempts to be faithful to.

Conceptually, GHC starts with a kind of “null language,” in which no extensions are enabled, and then enables a select number of extensions “by default.”  Which extensions count as “default extensions” in this sense is controlled by which of the following flags, if either, is present (if they are both present, the last one “wins”).

# `Haskell98`

**Available in:  All recent GHC versions**

The `Haskell98` “language” attempts to follow [the Haskell 98 Revised Report](http://www.haskell.org/onlinereport/) as closely as it can; the extensions it enables are currently:

- [`DatatypeContexts`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/data-type-extensions#datatypecontexts)
- `ImplicitPrelude`
- `MonomorphismRestriction`
- [`NondecreasingIndentation`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/basic-syntax-extensions#nondecreasingindentation)
- [`NPlusKPatterns`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/pattern-and-guard-extensions#npluskpatterns)
- `TraditionalRecordSyntax`

# `Haskell2010`

**Available in:  All recent GHC versions**

The `Haskell2010` “language”  attempts to follow [the Haskell 2010 Report](http://www.haskell.org/onlinereport/haskell2010/) as closely as it can; the extensions it enables are currently:

- [`DatatypeContexts`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/data-type-extensions#datatypecontexts)
- [`DoAndIfThenElse`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/basic-syntax-extensions#doandifthenelse)
- [`EmptyDataDecls`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/data-type-extensions#emptydatadecls)
- `ForeignFunctionInterface`
- `ImplicitPrelude`
- `MonomorphismRestriction`
- [`PatternGuards`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/pattern-and-guard-extensions#patternguards)
- `RelaxedPolyRec`
- `TraditionalRecordSyntax`

# The Default Language

If neither of the above two “languages” are used, then GHC enables a short list of extensions as a “default language;” these extensions are currently:

- [`DoAndIfThenElse`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/basic-syntax-extensions#doandifthenelse)
- [`EmptyDataDecls`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/data-type-extensions#emptydatadecls)
- `ForeignFunctionInterface`
- `ImplicitPrelude`
- `MonomorphismRestriction`
- [`NondecreasingIndentation`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/basic-syntax-extensions#nondecreasingindentation)
- [`PatternGuards`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/pattern-and-guard-extensions#patternguards)
- `RelaxedPolyRec`
- `TraditionalRecordSyntax`