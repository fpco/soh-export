# How to cabal install

`cabal` is a command-line program for downloading and building software written in [Haskell].
It can install all kinds of fascinating and useful software packages from the [Hackage] repository.
It is excellent and indispensable, but it currently has a troublesome flaw:
it sometimes mysteriously refuses to install things, leading to cries of "*Aaagh! cabal hell!!*".

A little extra know-how prevents this.
This tutorial aims to show you how to install cabal packages with confidence,
especially if you are new to Cabal and Haskell.
Welcome and let's get started!


## Should I use cabal ?

Your system may have a package manager, like apt-get, yum, or macports, and it might offer packages for the Haskell software you want to install.
In this case you may save time by using it instead of cabal. It probably offers more stable, better-integrated packages, and they may be pre-compiled.

Otherwise, yes, use cabal. It is cross-platform and can install the widest range of up-to-date Haskell software.
It is also an essential tool if you want to develop Haskell software.


## cabal, Cabal, cabal-install

Let's clarify these now to avoid confusion later. The [Cabal wiki page][Cabal] says:

> *"Cabal is a package and build system. Cabal is only involved in the creation of packages and the building of their contents. It does not manage packages.*
> *Cabal-Install installs cabal packages. It is distinct from Cabal (the build system). This often confuses new users. Furthermore, Cabal-Install is not a fully featured package manager. For example, it cannot install non cabal packaged dependencies, it cannot uninstall packages, nor can it automatically upgrade installations.*

In short: this tutorial is about using [cabal-install](http://hackage.haskell.org/package/cabal-install), which is `cabal` on the command line.
We'll just say cabal from now on.
It installs cabal packages, can't uninstall them, and can upgrade them only with supervision.[^3]


## Getting cabal

cabal installs software, and can upgrade itself, but first you need get it installed by some other means.
It is often available as a system package, otherwise get it by installing the [Haskell Platform], 
or just [GHC].

To check that it's installed, at a command prompt do:

```
$ cabal --version
cabal-install version 1.16.0.2
using version 1.16.0 of the Cabal library
$
```

You should avoid versions older than 0.14. (After 0.14 the version number jumped to 1.16).
If you do have a very old version, you might be able to upgrade like so:

```
$ cabal update
Downloading the latest package list from hackage.haskell.org
$ cabal install cabal-install
Resolving dependencies...
Configuring cabal-install-1.16.0.2...
Building cabal-install-1.16.0.2...
...
Installed cabal-install-1.16.0.2
$
```


## Installing things

As we just saw, installing usually goes like this:

1. find a package you want to install, eg from
   the [Hackage package list](http://hackage.haskell.org/packages/archive/pkg-list.html)
   or `cabal list`
2. update your local list of packages and dependencies, with `cabal update`
3. download and install the package with `cabal install PACKAGE`.  
   Some useful options:  
   `--dry-run` to see what cabal plans to do (recommended),  
   `-jN` to build N packages in parallel,  
   `-fFLAG` or `-f-FLAG` to turn optional build flags on or off, for packages that have them.  

For example:

```
$ cabal update
Downloading the latest package list from hackage.haskell.org
$ cabal install --dry shelltestrunner
Resolving dependencies...
In order, the following would be installed (use -v for more details):
Diff-0.2.0
ansi-terminal-0.6
ansi-wl-pprint-0.6.6
filemanip-0.3.6.2
hostname-1.0
test-framework-0.8
test-framework-hunit-0.3.0
shelltestrunner-1.3.1
$ cabal install shelltestrunner -j2
Resolving dependencies...
Downloading Diff-0.2.0...
Downloading ansi-terminal-0.6...
Configuring filemanip-0.3.6.2...
..etc..
Downloading shelltestrunner-1.3.1...
Configuring shelltestrunner-1.3.1...
Building shelltestrunner-1.3.1...
Installed shelltestrunner-1.3.1
$
```


## How to check what's installed

Note that cabal knows about only some installed packages:

```
$ cabal list --installed Diff
* Diff
    Synopsis: O(ND) diff algorithm in haskell.
    Default available version: 0.3.0
    Installed versions: 0.2.0
    License:  BSD3

$ cabal list --installed shelltestrunner
No matches found.
$
```

cabal uses a lower-level tool called `ghc-pkg`. It shows the same results:

```
$ ghc-pkg list --simple Diff 
Diff-0.2.0
$ ghc-pkg list --simple shelltestrunner
$
```

Packages can contain libraries (for use by other packages), executables (for use by humans), or both.
cabal and ghc-pkg only keep track of installed library packages.
The shelltestrunner package provides only executables, so cabal and ghc-pkg do not care about it after installation.
To check for installed executables, we can do eg:

```
$ ls -lt ~/.cabal/bin | head -5
total 1852920
-rwxr-xr-x  1 simon  simon  10394608 Mar  2 07:17 shelltest
-rwxr-xr-x  1 simon  simon   4962612 Mar  2 07:10 gearbox
-rwxr-xr-x  1 simon  simon  12279756 Mar  1 06:56 cabal
-rwxr-xr-x  1 simon  simon   1083948 Mar  1 06:54 hakyll-init
$
```


## Could not resolve dependencies: cabal hell

Over time, as you install more packages, and as new versions are released on Hackage,
`cabal install` becomes more likely to fail due to unsatisfiable dependencies.
There are other reasons for install failure (bad dependencies, bad code, incompatible compiler version, missing C libraries),
but unsatisfiable dependencies is the most common.
When it happens, you'll see something horrible like:

```
cabal install hledger-0.18   # an old version to reproduce the problem
Resolving dependencies...
cabal: Could not resolve dependencies:
rejecting: hledger-0.19.4/installed-402..., 0.19.3, 0.19.2, 0.19.1, 0.19,
0.18.2, 0.18.1 (global constraint requires ==0.18)
trying: hledger-0.18
trying: regexpr-0.5.4/installed-0da...
trying: process-1.1.0.2/installed-7b6...
rejecting: haskeline-0.7.0.3/installed-414..., 0.7.0.3, 0.7.0.2, 0.7.0.1,
0.7.0.0 (conflict: hledger => haskeline==0.6.*)
rejecting: haskeline-0.6.4.7 (conflict: process =>
unix==2.6.0.1/installed-ccb..., haskeline => unix>=2.0 && <2.6)
rejecting: haskeline-0.6.4.6 (conflict: regexpr =>
mtl==2.1.2/installed-538..., haskeline => mtl>=1.1 && <2.1)
rejecting: haskeline-0.6.4.5, 0.6.4.4, 0.6.4.3, 0.6.4.2, 0.6.4.1, 0.6.4.0,
0.6.3.2, 0.6.3.1, 0.6.3, 0.6.2.4, 0.6.2.3 (conflict: process =>
filepath==1.3.0.1/installed-a78..., haskeline => filepath>=1.1 && <1.3)
rejecting: haskeline-0.6.2.2, 0.6.2.1, 0.6.2, 0.6.1.6, 0.6.1.5, 0.6.1.3,
0.6.1.2, 0.6.1.1, 0.6.1, 0.6.0.1, 0.6 (conflict: process =>
filepath==1.3.0.1/installed-a78..., haskeline => filepath==1.1.*)
rejecting: haskeline-0.5.0.1, 0.5, 0.4, 0.3.2, 0.3.1, 0.3, 0.2.1, 0.2
(conflict: hledger => haskeline==0.6.*)
```

What has happened is that your installed packages,
plus the new packages cabal thinks should be installed,
plus a single-version-per-build constraint required by GHC[^1],
have formed a network of dependencies that cabal can't satisfy.
And, cabal's explanation of the problem is hard for a human to understand.
You are entering *cabal hell!*


## The easy solution: reset your packages

There is an easy workaround that does not require cabal troubleshooting skills or special tools.
If we clear out all installed libraries, cabal may have to reinstall a few but will have a much better chance of success.
The easiest way to do this (on unix) is:

```
$ rm -rf ~/.ghc ~/.cabal
$ 
```

This is simplest and reclaims most disk space, but it also deletes executables and config files. 
A less crude way is to use this bash function, which you can add to your ~/.bashrc:

```bash
# ghc-pkg-reset
# Removes all installed GHC/cabal packages, but not binaries, docs, etc.
# Use this to get out of dependency hell and start over, at the cost of some rebuilding time.
function ghc-pkg-reset() {
    read -p 'erasing all your user ghc and cabal packages - are you sure (y/n) ? ' ans
    test x$ans == xy && ( \
        echo 'erasing directories under ~/.ghc'; rm -rf `find ~/.ghc -maxdepth 1 -type d`; \
        echo 'erasing ~/.cabal/lib'; rm -rf ~/.cabal/lib; \
        # echo 'erasing ~/.cabal/packages'; rm -rf ~/.cabal/packages; \
        # echo 'erasing ~/.cabal/share'; rm -rf ~/.cabal/share; \
        )
}
```

and:

```
$ source ~/.bashrc
$ ghc-pkg-reset
erasing all your user ghc and cabal packages - are you sure (y/n) ? y
$ ghc-pkg list --user
/Users/simon/.ghc/x86_64-darwin-7.6.1/package.conf.d
$
```

and then run the cabal install command again.


## The careful solution: use a sandbox

Another approach is to keep separate installed package sets, known as sandboxes.
This costs more disk space and build time overall (for me, each package set takes about 0.5G of space).
But when you need to work on multiple projects whose dependencies are incompatible,
or to isolate projects from unrelated upgrades, this is the solution.
cabal does not yet provide this feature natively; for now it means using an additional tool, either [cabal-dev] or [hsenv].


## The harder, quicker solution: clean up your packages

As you get familiar with diagnosing these failures, you will more often be able to see how to adjust your installed package set to give cabal more freedom.
Sometimes just removing an obsolete installed package (with `ghc-pkg unregister PACKAGE`) can get things unstuck.
It's best to have a theory when you try this - random trial and error will often make things worse.


## Dealing with bad dependencies

If, after a reset or in a clean sandbox, there are still unsatisfiable dependencies,
it means the released packages on Hackage have bad (incorrect, out-of-date, too tight, too loose) dependency declarations,
or the packages are not compatible with your GHC version.
In this case identify the problem package(s) and get the maintainer's help,
and/or try fixing the dependencies yourself in a local copy of the package (`cabal unpack` is good for this).


## Summary

- [`cabal`][cabal-install] installs [Haskell] software from [Hackage]. It comes with [GHC] or the [Haskell Platform] or as a system package. Use at least version 0.14, and preferably the newest.
- Use it instead of your system's package manager when you want to install (or develop) the latest Haskell software.
- The main subcommands are `update`, `list` and `install`.
- install becomes more likely to fail over time, as packages are added locally and on Hackage. Understanding these failures can be hard.
- The easy fix is to reset your packages, eg with [`ghc-pkg-reset`](#reset).
- Or, you can install packages in separate sandboxes with [cabal-dev] or [hsenv].
- Or, with experience quicker fixes may be apparent (eg uninstall old package versions, with `ghc-pkg unregister`).
- Packages which fail to install even with a clean slate should be reported to their maintainers.

May you be free from *cabal hell* and enjoy *haskell heaven!*


## About this document

- [Discussion](http://www.reddit.com/r/haskell/comments/19jbz5/how_to_cabal_install_a_new_tutorial/) on reddit, 2013/3/2-...
- [School of Haskell version](https://haskell.fpcomplete.com/user/simonmichael/how-to-cabal-install) at fpcomplete.com, short and to the point
- [Darcs hub version](http://hub.darcs.net/simon/cabal-install-tutorial/README.md) at hub.darcs.net, slightly more in-depth
  ([changes](http://hub.darcs.net/simon/cabal-install-tutorial/changes))



[^1]: GHC requires that no more than one version of each package is used in any build of a library or executable.

[^2]: At least, not since version 0.14. Before then, cabal used to quietly and too eagerly upgrade installed packages, sometimes leaving you with broken local dependencies.

[^3]: More background: [Repeat after me: “Cabal is not a Package Manager”](http://ivanmiljenovic.wordpress.com/2010/03/15/repeat-after-me-cabal-is-not-a-package-manager/).

[Haskell]:          http://haskell.org
[Cabal]:            http://haskell.org/haskellwiki/Cabal
[Hackage]:          http://hackage.haskell.org
[Stackage]:         https://github.com/fpco/stackage
[cabal-install]:    http://www.haskell.org/haskellwiki/Cabal-Install
[GHC]:              http://haskell.org/ghc
[Haskell Platform]: http://haskell.org/platform
[cabal-dev]:        http://hackage.haskell.org/package/cabal-dev
[hsenv]:            http://hackage.haskell.org/package/hsenv
[SICP]:             http://www.vex.net/~trebla/haskell/sicp.xhtml
