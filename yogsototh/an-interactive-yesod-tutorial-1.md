# An Interactive Yesod Tutorial

The following code run a server which listen on `/` and `/echo/*`

If you go on `/` you are given a list of links pointing to url of the kind `/echo/*`.
Click on them and you get a fading echo. Try it to see what I mean.


``` active haskell web
-- show
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod
data MyApp = MyApp
instance Yesod MyApp -- We Declare MyApp to be a Yesod App

-- We declare the active URLs
mkYesod "MyApp" [parseRoutes|
  /             HomeR GET
  /echo/#String EchoR GET
|]

-- getHomeR doesn't take any parameter
-- return an HTML page
-- with two links to the same local URL
-- /echo/Le%20Big%20Mac
getHomeR = defaultLayout [whamlet|
  <ul>
   <li><a href=/echo/Yesod>to /echo/Yesod
   <li><a href=@{EchoR "Yesod"}>to @{EchoR "Yesod"}
|]

-- getEchoR take as parameter the content of the first
-- url variable declared in the route part
-- Then it write it many time fading :)
getEchoR theText = defaultLayout [whamlet|
    #{theText}
      <sup style="opacity:0.9">#{theText}
        <sup style="opacity:0.8">#{theText}
          <sup style="opacity:0.7">#{theText}
            <sup style="opacity:0.6">#{theText}
              <sup style="opacity:0.5">#{theText}
    <br>
    <a href=@{HomeR}>go Back
    |]

main = warpEnv MyApp
```

**ex 1:** Modify the links such that, when clicking on them you get the echo of "Hello"

@@@
``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod
data MyApp = MyApp
instance Yesod MyApp -- We Declare MyApp to be a Yesod App

mkYesod "MyApp" [parseRoutes|
  /         HomeR GET
  /echo/#String EchoR GET
|]

-- show
getHomeR = defaultLayout [whamlet|
  <ul>
   <li><a href=/echo/Hello>to /echo/Hello
   <li><a href=@{EchoR "Hello"}>to @{EchoR "Hello"}
|]
-- /show

getEchoR theText = defaultLayout [whamlet|
    #{theText}
      <sup style="opacity:0.9">#{theText}
        <sup style="opacity:0.8">#{theText}
          <sup style="opacity:0.7">#{theText}
            <sup style="opacity:0.6">#{theText}
              <sup style="opacity:0.5">#{theText}
    <br>
    <a href=@{HomeR}>go Back
    |]

main = warpEnv MyApp
```

This one was just for warming you up.

@@@

**ex 2:** Modify the links such that, when clicking on them you get the echo of "Le Big Mac"

@@@

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod
data MyApp = MyApp
instance Yesod MyApp -- We Declare MyApp to be a Yesod App

mkYesod "MyApp" [parseRoutes|
  /         HomeR GET
  /echo/#String EchoR GET
|]

-- show
getHomeR = defaultLayout [whamlet|
  <ul>
   <li><a href=/echo/Le%20Big%20Mac>to /echo/Le%20Big%20Mac
   <li><a href=@{EchoR "Le Big Mac"}>to @{EchoR "Le Big Mac"}
|]
-- /show

getEchoR theText = defaultLayout [whamlet|
    #{theText}
      <sup style="opacity:0.9">#{theText}
        <sup style="opacity:0.8">#{theText}
          <sup style="opacity:0.7">#{theText}
            <sup style="opacity:0.6">#{theText}
              <sup style="opacity:0.5">#{theText}
    <br>
    <a href=@{HomeR}>go Back
    |]

main = warpEnv MyApp
```

Using accessor is easier no? Yesod does the tedious escaping work for you.

@@@

**ex 3:** Modify the code such that, the active route is no more `/echo/*` but `/le/*`

@@@

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod
data MyApp = MyApp
instance Yesod MyApp -- We Declare MyApp to be a Yesod App

-- show
mkYesod "MyApp" [parseRoutes|
  /         HomeR GET
  /le/#String EchoR GET
|]

-- show the first link change while the second remain identical
getHomeR = defaultLayout [whamlet|
  <ul>
   <li><a href=/le/Le%20Big%20Mac>to /le/Le%20Big%20Mac
   <li><a href=@{EchoR "Le Big Mac"}>to @{EchoR "Le Big Mac"}
|]
-- /show

getEchoR theText = defaultLayout [whamlet|
    #{theText}
      <sup style="opacity:0.9">#{theText}
        <sup style="opacity:0.8">#{theText}
          <sup style="opacity:0.7">#{theText}
            <sup style="opacity:0.6">#{theText}
              <sup style="opacity:0.5">#{theText}
    <br>
    <a href=@{HomeR}>go Back
    |]

main = warpEnv MyApp
```

Using accessor means you don't have to change the code.
While you had to manually change the url when writting it manually.
URL accessors are clearly superior.
Now we should get rid of the first manually entered link.

@@@

**ex 4:** Modify the code such that, it will add "le " before the repeating text.

@@@

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod
data MyApp = MyApp
instance Yesod MyApp -- We Declare MyApp to be a Yesod App

mkYesod "MyApp" [parseRoutes|
  /         HomeR GET
  /le/#String EchoR GET
|]

-- show Changed the link to remove "le"
getHomeR = defaultLayout [whamlet|
  <ul>
   <li><a href=@{EchoR "Big Mac"}>to @{EchoR "Big Mac"}
|]

-- show
getEchoR theText = defaultLayout [whamlet|
    Le #{theText}
      <sup style="opacity:0.9">le #{theText}
        <sup style="opacity:0.8">le #{theText}
          <sup style="opacity:0.7">le #{theText}
            <sup style="opacity:0.6">le #{theText}
              <sup style="opacity:0.5">le #{theText}
    <br>
    <a href=@{HomeR}>go Back
    |]
-- /show    

main = warpEnv MyApp
```

I manually entered le before each entry, that was tedious...

@@@

**ex 5:** Modify the code such that, the active route is now `/le/<some number>/*`

@@@

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod
data MyApp = MyApp
instance Yesod MyApp -- We Declare MyApp to be a Yesod App

-- show
mkYesod "MyApp" [parseRoutes|
  /                    HomeR GET
  /le/#Integer/#String EchoR GET
|]

-- show added a parameter
getHomeR = defaultLayout [whamlet|
  <ul>
   <li><a href=@{EchoR 8 "Big Mac"}>to @{EchoR 8 "Big Mac"}
|]
-- /show

getEchoR nb theText = defaultLayout [whamlet|
    Le #{theText}
      <sup style="opacity:0.9">le #{theText}
        <sup style="opacity:0.8">le #{theText}
          <sup style="opacity:0.7">le #{theText}
            <sup style="opacity:0.6">le #{theText}
              <sup style="opacity:0.5">le #{theText}
    <br>
    <a href=@{HomeR}>go Back
    |]
-- /show    

main = warpEnv MyApp
```

Adding a new parameter imply more changes in the code.

@@@

**ex 6:** Modify the code such that, the number of echo is given by the number in the URL parameter.

_Hint_: You can call an Handler function using ^{function} in hamlet.

@@@

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod
import Text.Printf
data MyApp = MyApp
instance Yesod MyApp -- We Declare MyApp to be a Yesod App

mkYesod "MyApp" [parseRoutes|
  /                    HomeR GET
  /le/#Integer/#String EchoR GET
|]

getHomeR = defaultLayout [whamlet|
  <ul>
   <li><a href=@{EchoR 8 "Big Mac"}>to @{EchoR 8 "Big Mac"}
   <li><a href=@{EchoR 20 "Big Mac"}>to @{EchoR 20 "Big Mac"}
   <li><a href=@{EchoR 200 "Ia! Ia! Cthulhu fatgn!"}>to Cthulhu!
|]

-- show
itofdiv :: Integer -> Integer -> Float
itofdiv x y = (fromIntegral x) / (fromIntegral y)

showEcho _ 0 _ = [whamlet| |]
showEcho m nb theText = do
  minval <- return $ div m 10
  opacityratio <- return $ itofdiv (nb + minval) (m + minval)
  opacity <- return $ (printf "%.4f" opacityratio:: String)
  [whamlet|
      <sup style=opacity:#{opacity}>le #{theText}
        ^{showEcho m (nb - 1) theText}|]
        
getEchoR nb theText = defaultLayout [whamlet|
    Le #{theText}
      ^{showEcho (nb - 1) (nb - 1) theText}
    <br>
    <a href=@{HomeR}>go Back
    |]
-- /show    

main = warpEnv MyApp
```

Adding a new parameter imply more changes in the code.

@@@

Hope you had fun!
