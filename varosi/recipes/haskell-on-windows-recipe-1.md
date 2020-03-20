# Haskell on Windows recipe #1

OUTDATED! Currently this tutorial is not up to date, because EcplipseFP plug-in is abandoned!

This tutorial explains what to do exactly to setup Haskell working environment under Windows using Eclipse as a IDE and MinGHC tool chain in a few steps. Steps that just work.

### Using MinGHC Haskell tool chain
1. Install [MinGHC 7.8.3 installer](http://www.stackage.org/install#windows)
2. Ensure those paths are first: set PATH=%USERPROFILE%\AppData\Roaming\cabal\bin;%PATH%
3. cabal update & cabal install alex happy cabal-install-1.20.0.6

Note:

- cabal 1.20.x is currently supported by latest buildwrapper. [1.22 is not working now](https://github.com/JPMoresmau/BuildWrapper/issues/52).

### Installing IDE and supporting tools

1. wget http://www.stackage.org/lts/cabal.config
2. cabal sandbox init & cabal install **buildwrapper hoogle hlint stylish-haskell scion-browser**
4. copy .cabal-sandbox/bin/*.exe %USERPROFILE%/AppData/Roaming/cabal/bin/
5. cabal sandbox delete
6. hoogle data
7. Install [Eclipse Luna](https://eclipse.org/home/index.php)
8. Install [EclipseFP](http://eclipsefp.github.io/install.html) (From inside Eclipse do only plug-in installation. Tools are already installed.)
9. cd **where_my_cabal_file_is**
10. wget http://www.stackage.org/lts/cabal.config
11. cabal update & cabal install

### If you want to Cleanup/deinstall old Haskell installations

1. Deinstall Platform Haskell (from Control Panel | Programs and Features)
2. Check if PATH env. variable is not containing anything about haskell, cabal or GHC
3. Delete %USERPROFILE%\AppData\Roaming\cabal folder
4. Delete %USERPROFILE%\AppData\Roaming\ghc folder
