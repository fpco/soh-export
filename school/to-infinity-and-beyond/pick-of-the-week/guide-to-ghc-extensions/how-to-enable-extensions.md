# How to Enable Extensions

This post is designed to help you if there is a GHC extension that you want to use, but you’re not sure how to go about enabling it.

There are (at least) three ways to enable an extension in GHC, and you can use any combination of the three.  For the rest of this post, we will talk about a hypothetical extension called *`ExtensionName`*.

# `LANGUAGE` Pragma

You can enable an extension for a single file by placing `{-# LANGUAGE `*`ExtensionName `*`#-}` on its own line at or near the top of the file (anywhere before the `module` header, or before the first `import` or definition if there is no `module` header, should be fine).

You can enable multiple extensions by using multiple `LANGUAGE` pragmas, by mentioning multiple extension names seperated by commas within one `LANGUAGE` pragma, or by using any combination of the two.  For example, if you wanted to enable the extensions `ScopedTypeVariables`, `LiberalTypeSynonyms`, and `MultiWayIf`, all of the following are perfectly fine and do exactly the same thing:

``` haskell
{-# LANGUAGE ScopedTypeVariables, LiberalTypeSynonyms, MultiWayIf #-}
```

``` haskell
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE LiberalTypeSynonyms #-}
{-# LANGUAGE MultiWayIf #-}
```

``` haskell
{-# LANGUAGE ScopedTypeVariables, LiberalTypeSynonyms #-}
{-# LANGUAGE MultiWayIf #-}
```

# `.cabal` File

If your project is managed by `cabal` and/or `stack`, then your `.cabal` file can include extensions that will apply to the whole project.  Simply place **`extensions: `*`ExtensionName`*** on it’s own line in the `.cabal` file’s `library` and/or `executable` section(s) (depending on which target(s) you want to enable the extension for).

You can enable multiple extensions at once by placing multiple extension names, seperated by commas, on the `extensions:` line; you can even seperate them by whitespace, including newlines, as long as the result respects [Haskell’s usual notion of “layout.”](https://www.haskell.org/onlinereport/haskell2010/haskellch2.html#x7-210002.7)

# `-X` Options

For a single compilation, REPL session, or interpretation, you can enable an extension by passing `ghc`, `ghci`, or `runghc` (respectively) the command-line flag `-X`*`ExtensionName`*.  This works no matter where the flag comes from, so long as it ends up being passed to the program in question via its command-line arguments.

You can enable multiple extensions by passing multiple `-X` flags.

# Disabling an Extension

In any of the three ways discussed, you can always disable an extension by enabling the extension name `No`*`ExtensionName`*.  If the extension’s name already follows that pattern, then simply remove the `No` prefix instead.

The only extensions for which there are no `No`-variants are:

- [`Haskell98`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/language-standards#haskell98)
- [`Haskell2010`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/language-standards#haskell2010)
- `Unsafe`
- `Trustworthy`
- `Safe`

The first two of those are sort of like extension bundles whose only purpose is to set a whole bunch of other extension flags with the goal of being compliant with a particular Haskell standard.  The last three act together as a three-way switch, in a similar way to how all other extensions act as a two-way switch with their `No`-variants.

# Mentioning an Extension Multiple Times

If an extension is mentioned multiple times, it is still enabled as usual.  If *`ExtensionName`* and `No`*`ExtensionName`* are both mentioned, the last one given wins.

# Checking Which Extensions are Supported in a particular GHC version

The [GHCanIUse table](https://damianfral.github.io/ghcaniuse/) is a good way to check which GHC versions support which language extensions.

You can also pass the command-line option `--supported-extensions` (or its synonym, `--supported-languages`) to `ghc`, which will output a list of extension names.  Unlike the [GHCanIUse table](https://damianfral.github.io/ghcaniuse/), this list will include all the [`No`-variants](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/how-to-enable-extensions#disabling-an-extension) of all the extensions available to your version of ghc.