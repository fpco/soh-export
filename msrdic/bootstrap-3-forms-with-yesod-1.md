# Bootstrap 3 forms with Yesod

<hr />

*Edit: this utility code has been modified by [Felipe Lessa](https://github.com/meteficha) and merged into [Yesod](https://github.com/yesodweb/yesod/pull/687).*

<hr />

While working with forms in Yesod, I noticed that the `renderBootstrap` functionality doesn't work with Bootstrap 3. There were some breaking changes introduced in version 3, so it got me thinking of implementing some basic support for Bootstrap 3 forms.

## Bootstrap 3 form kinds

As can be seen in [Bootstrap forms documentation](http://getbootstrap.com/css/#forms), there are basically three kinds of forms in Bootstrap.

The first kind of form is the simplest of all: it arranges its elements one per row. Here's some HTML code for a basic form:

```html
<form>
  <div class="form-group">
    <label for="name">Name</label>
    <input type="text" class="form-control" id="name" placeholder="Name">
  </div>
  <div class="form-group">
    <label for="surname">Surname</label>
    <input type="text" class="form-control" id="surname" placeholder="Surname">
  </div>
  <button type="submit" class="btn btn-default">Submit</button>
</form>
```
Labels and input fields are grouped in containers that have a class attribute `form-group` and input fields additionally have a class attribute `form-control`.

The second kind of form is an inline form. This form doesn't have any labels, but they are just rendered hidden because screen readers can still "see" them. The example source code for this form looks like this:

```html
<form class="form-inline">
  <div class="form-group">
    <label class="sr-only" for="name">Name</label>
    <input type="text" class="form-control" id="name" placeholder="Name">
  </div>
  <div class="form-group">
    <label class="sr-only" for="surname">Surname</label>
    <input type="text" class="form-control" id="surname" placeholder="Surname">
  </div>
  <button type="submit" class="btn btn-default">Submit</button>
</form>
```
The third kind of form is a horizontal form. This form has its elements arranged in a grid, where each group of controls (label, field and tooltips container) has its own row. The example form would look like this:

```html
<form class="form-horizontal">
  <div class="form-group">
    <label for="name" class="col-sm-2 control-label">Name</label>
    <div class="col-sm-4">
      <input type="text" class="form-control" id="name" placeholder="Name">
    </div>
  </div>
  <div class="form-group">
    <label for="surname" class="col-sm-2 control-label">Surname</label>
    <div class="col-sm-4">
      <input type="text" class="form-control" id="name" placeholder="Surname">
    </div>
  </div>
  <div class="form-group">
    <div class="col-sm-offset-2 col-sm-4">
      <button type="submit" class="btn btn-default">Submit</button>
    </div>
  </div>
</form>
```

## Forms comparison

From the examples above we can see that these forms are pretty similar, with minor differences that will affect the implementation. The type of form is determined via `class` parameter, or, in the case of a basic form, no `class` parameter. In all of the three cases form controls are contained inside the `div` container with a `class` attribute of `form-group`. In case of inline forms, labels have a `class` attribute of `sr-only`. Input fields of horizontal forms have its own `div` container, and additionally, labels and enclosing field containers have `class` attributes for positioning and sizing. With these in mind, we can slowly begin implementing some basic support for these kinds of forms.

## First steps

As I already said above, there's a Bootstrap support in Yesod, but it's outdated. Here's the `renderBootstrap` function, taken from the `Yesod.Form.Functions` module, with the rendering code marked:

```haskell
renderBootstrap :: Monad m => FormRender m a
renderBootstrap aform fragment = do
    (res, views') <- aFormToForm aform
    let views = views' []
        has (Just _) = True
        has Nothing  = False
    let widget = [whamlet|
                $newline never
                \#{fragment}
          {-hi-}$forall view <- views {-/hi-}
              {-hi-}<div .control-group .clearfix :fvRequired view:.required :not $ fvRequired view:.optional :has $ fvErrors view:.error>{-/hi-}
                  {-hi-}<label .control-label for=#{fvId view}>#{fvLabel view}{-/hi-}
                  {-hi-}<div .controls .input>{-/hi-}
                      {-hi-}^{fvInput view}{-/hi-}
                      {-hi-}$maybe tt <- fvTooltip view{-/hi-}
                          {-hi-}<span .help-block>#{tt}{-/hi-}
                      {-hi-}$maybe err <- fvErrors view{-/hi-}
                          {-hi-}<span .help-block>#{err}{-/hi-}
                |]
    return (res, widget)
```
The rendering code loops through the list of `views` and renders every form element Bootstrap-friendly, so this is the first place for introducing Bootstrap 3 changes. The first thing to change is the `class` attribute of an eclosing container from `.control-group` to `.form-group`. To make other changes in rendering function, we first need to introduce some additional data types and utility functions.

The first data type that's useful is a `BootstrapForm` data type:

```haskell
data BootstrapForm = BootstrapBasicForm | BootstrapInlineForm
    | BootstrapHorizontalForm { containerOffset :: GridOptions, container :: GridOptions, label :: GridOptions }
```

Since a horizontal form has additional parameters for positioning and sizing form elements, we need to introduce a data type to represent these parameters. `GridOptions` is a data type encoding [standard grid sizes](http://getbootstrap.com/css/#grid-options) in Bootstrap 3 and is defined as following:

```haskell
data GridOptions = ColXs Int | ColSm Int | ColMd Int | ColLg Int

instance Show GridOptions where
    show (ColXs 0) = ""
    show (ColXs columns) = "col-xs-" ++ show columns

    show (ColSm 0) = ""
    show (ColSm columns) = "col-sm-" ++ show columns

    show (ColMd 0) = ""
    show (ColMd columns) = "col-md-" ++ show columns

    show (ColLg 0) = ""
    show (ColLg columns) = "col-lg-" ++ show columns

instance ToMarkup GridOptions where
    toMarkup = toMarkup . show
```
The `Show` and `ToMarkup` instances are also useful, since we intend on rendering these options as class attributes.

We use the `GridOptions` and `BootstrapForm` data types and introduce a `BootstrapFormConfig` in the following way:

```haskell
data BootstrapFormConfig = BootstrapFormConfig { form :: BootstrapForm, submit :: String }
```
The `submit` field will be used later in rendering the submit button.

## Constructing forms in Yesod

It is very simple to construct a form with Yesod and the code could look something like this:

```haskell
personForm :: Html -> MForm Handler (FormResult Person, Widget)
personForm = renderBootstrap $ Person
    <$> areq textField "Name" Nothing
    <*> areq textField "Surname" Nothing
```
This is the basic example with the `Person` entity introduced in the form examples above. The string parameters (`"Name"` and `"Surname"`) are automatically converted to `FieldSettings` data types since there's an instance `IsString` defined for `FieldSettings` data type. `FieldSettings` data type (from `Yesod.Form.Types`) looks like this:

```haskell
data FieldSettings master = FieldSettings
    { fsLabel :: SomeMessage master
    , fsTooltip :: Maybe (SomeMessage master)
    , fsId :: Maybe Text
    , fsName :: Maybe Text
    , fsAttrs :: [(Text, Text)]
    }
```

Additionaly, the `textField` function (from `Yesod.Form.Fields`) used in form construction looks like this:

```haskell
textField :: Monad m => RenderMessage (HandlerSite m) FormMessage => Field m Text
textField = Field
    { fieldParse = parseHelper $ Right
    , fieldView = \theId name attrs val isReq ->
        [whamlet|
$newline never
{-hi-}<input id="#{theId}" name="#{name}" *{attrs} type="text" :isReq:required value="#{either id id val}">{-/hi-}
|]
    , fieldEnctype = UrlEncoded
    }
```
The important part is marked: all the parameters are rendered in a `hamlet` block, and the part that's most interesting to us is the `attrs` parameter, since it contains the `class` attribute and all the other attributes that are not one of the defined in the block above (`id`, `name` etc.). We need to somehow alter the `attrs` parameter to include the `placeholder` attribute and to introduce additonal `class` attributes.

## Getting closer

We want to continue using the `areq` (and other applicative variations like `aopt`, `mreq` and `mopt`), but we also want to introduce additional parameters to support Bootstrap 3. The type of `areq` looks like this:

```haskell
Prelude Yesod> :t areq
areq
  :: (MonadHandler m, RenderMessage (HandlerSite m) FormMessage) =>
     Field m a -> FieldSettings (HandlerSite m) -> Maybe a -> AForm m a
```
The last parameter to `areq` is the optional default value.

Let's introduce an additional parameter to the `renderBootstrap` function, namely the `BootstrapFormConfig`. The function (with the changes mentioned in [First steps](#first-steps)) now looks like this:

```haskell
renderBootstrap :: Monad m => BootstrapFormConfig -> FormRender m a
renderBootstrap {-hi-}formConfig{-/hi-} aform fragment = do
    (res, views') <- aFormToForm aform
    let views = views' []
        has (Just _) = True
        has Nothing  = False
    let widget = [whamlet|
                $newline never
                \#{fragment}
                $forall view <- views
                    <div {-hi-}.form-group{-/hi-} .clearfix :fvRequired view:.required :not $ fvRequired view:.optional :has $ fvErrors view:.error>
                        <label .control-label for=#{fvId view}>#{fvLabel view}
                        <div .controls .input>
                            ^{fvInput view}
                            $maybe tt <- fvTooltip view
                                <span .help-block>#{tt}
                            $maybe err <- fvErrors view
                                <span .help-block>#{err}
                |]
    return (res, widget)
```
We're now going to use the newly introduced `formConfig` parameter to render all three kinds of forms. We want to check which kind of form we're working with and render it in a way described in [Bootstrap 3 form kinds](#bootstrap-3-form-kinds) and [Forms comparison](#forms-comparison). Then we want to introduce the submit button as well, since the positioning of submit button depends on the form parameters.

```haskell
renderBootstrap formConfig aform fragment = do
    (res, views') <- aFormToForm aform
    let views = views' []
        has (Just _) = True
        has Nothing  = False
        widget = [whamlet|
                        \#{fragment}
                        $forall view <- views
                          <div .form-group :fvRequired view:.required :not $ fvRequired view:.optional :has $ fvErrors view:.error>
                            {-hi-}$case (form formConfig){-/hi-}
                              {-hi-}$of BootstrapBasicForm{-/hi-}
                                {-hi-}<label for=#{fvId view}>#{fvLabel view}{-/hi-}
                                {-hi-}^{fvInput view}{-/hi-}
                                {-hi-}^{helpWidget view}{-/hi-}
                              {-hi-}$of BootstrapInlineForm{-/hi-}
                                {-hi-}<label .sr-only for=#{fvId view}>#{fvLabel view}{-/hi-}
                                {-hi-}^{fvInput view}{-/hi-}
                                {-hi-}^{helpWidget view}{-/hi-}
                              {-hi-}$of BootstrapHorizontalForm containerOffset containerClass labelClass{-/hi-}
                                {-hi-}<label .control-label .#{labelClass} for=#{fvId view}>#{fvLabel view}{-/hi-}
                                {-hi-}<div .#{containerClass}>{-/hi-}
                                  {-hi-}^{fvInput view}{-/hi-}
                                {-hi-}^{helpWidget view}{-/hi-}

                        {-hi-}^{submitWidget $ formConfig}{-/hi-}
                |]
    return (res, widget)
```

We've also extracted the two functions, just for readability sake:

```haskell
submitWidget (BootstrapFormConfig (BootstrapHorizontalForm containerOffset containerClass labelClass) submit) = [whamlet|
<div .form-group>
    <div .#{toOffset containerOffset} .#{containerClass}>
      <button type=submit .btn .btn-default>#{submit}
|]
submitWidget (BootstrapFormConfig _ submit) = [whamlet|<button type=submit .btn .btn-default>#{submit}|]

helpWidget view = [whamlet|
    $maybe tt <- fvTooltip view
      <span .help-block>#{tt}
    $maybe err <- fvErrors view
      <span .help-block>#{err}
|]

toOffset :: GridOptions -> String
toOffset (ColXs 0) = ""
toOffset (ColSm 0) = ""
toOffset (ColMd 0) = ""
toOffset (ColLg 0) = ""
toOffset (ColXs columns) = "col-xs-offset-" ++ show columns
toOffset (ColSm columns) = "col-sm-offset-" ++ show columns
toOffset (ColMd columns) = "col-md-offset-" ++ show columns
toOffset (ColLg columns) = "col-lg-offset-" ++ show columns
```

And lastly, we introduce the function for building `FieldSettings` values. We also add an additional parameter for the placeholder attribute. We then introduce the placeholder parameter as one of the input attributes with the `attrsFromFormConfig` function.

```haskell
bootstrapFieldSettings :: BootstrapFormConfig -> SomeMessage site -> Maybe (SomeMessage site)
    -> Maybe Text -> Maybe Text -> Maybe Text -> FieldSettings site
bootstrapFieldSettings formConfig msg tooltip placeholder id name =
    FieldSettings msg tooltip id name (attrsFromFormConfig formConfig placeholder)

attrsFromFormConfig :: BootstrapFormConfig -> Maybe Text -> [(Text, Text)]
attrsFromFormConfig _ Nothing = [("class", "form-control")]
attrsFromFormConfig _ (Just placeholder) = [("class", "form-control"), ("placeholder", placeholder)]
```

## Constructing Bootstrap 3 forms

With all these building blocks in place, we can now construct a Bootstrap 3 form. This is the noisy way to construct a basic form since we need to supply all the parameters for `FieldSettings` parameter:

```haskell
personHForm = renderBootstrap hConfig $ Person
    <$> areq textField (bootstrapFieldSettings config "Name" Nothing (Just "Person name") Nothing Nothing) Nothing
    <*> areq textField (bootstrapFieldSettings config "Surname" Nothing (Just "Person surname") Nothing Nothing) Nothing
```

To make things a little bit easier, as we only care about labels and placeholders, we can extract the `bootstrapFieldHelper` function:

```haskell
bootstrapFieldHelper config label placeholder =
    bootstrapFieldSettings config label Nothing placeholder Nothing Nothing
```

The code for constructing the basic form now looks like this:

```haskell
personForm :: Html -> MForm Handler (FormResult Person, Widget)
personForm = renderBootstrap bConfig $ Person
    <$> areq textField (bootstrapFieldHelper bConfig "Name" (Just "Person name")) Nothing
    <*> areq textField (bootstrapFieldHelper bConfig "Surname" (Just "Person surname")) Nothing
```

## The full example

The full example, showing both inline and horizontal forms, is below. It also shows other field types that I checked by now, in a larger example. The embedded browser doesn't render the inline form correctly, so opening it in a new tab should show everything as expected.

```haskell active web

{-# START_FILE Foundation.hs #-}

{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TypeFamilies #-}

module Foundation where

import           Yesod                   hiding (renderBootstrap)
import           Yesod.Form.Jquery

import           Data.Text               (Text)
import           Data.Time               (Day, TimeOfDay (..))

import           Control.Applicative     ((<$>), (<*>))

import           Form.Bootstrap3

data FormApp = FormApp

instance Yesod FormApp

instance YesodJquery FormApp where
    urlJqueryJs _ = Right "//ajax.googleapis.com/ajax/libs/jquery/1.7/jquery.min.js"

instance RenderMessage FormApp FormMessage where
    renderMessage _ _ = defaultFormMessage

mkYesodData "FormApp" $(parseRoutesFile "config/routes")

data Person = Person { name :: Text, surname :: Text }
    deriving Show

data LargeData = LargeData {
    textField1       :: Text,
    intField1        :: Int,
    doubleField1     :: Double,
    textAreaField1   :: Textarea,
    hiddenField1     :: Text,
    passwordField1   :: Text,
    emailField1      :: Text,
    htmlField1       :: Html,
    dayField1        :: Day,
    timeField1       :: TimeOfDay,
    searchField1     :: Text,
    urlField1        :: Text,
    selectField1     :: Bool,
    checkboxField1   :: Bool
    }

hConfig = BootstrapFormConfig { form = BootstrapHorizontalForm (ColXs 2) (ColXs 4) (ColXs 2), submit = "Create user" }
iConfig = BootstrapFormConfig { form = BootstrapInlineForm, submit = "Create user"}
bConfig = BootstrapFormConfig { form = BootstrapBasicForm, submit = "Create user" }
largeFormConfig = BootstrapFormConfig { form = BootstrapHorizontalForm (ColXs 2) (ColXs 4) (ColXs 2), submit = "Submit large data" }

bootstrapFieldHelper config label placeholder = bootstrapFieldSettings config label Nothing placeholder Nothing Nothing

personHForm :: Html -> MForm Handler (FormResult Person, Widget)
personHForm = renderBootstrap hConfig $ Person
    <$> areq textField (bootstrapFieldHelper hConfig "Name" (Just "Person name")) Nothing
    <*> areq textField (bootstrapFieldHelper hConfig "Surname" (Just "Person surname")) Nothing

personIForm :: Html -> MForm Handler (FormResult Person, Widget)
personIForm = renderBootstrap iConfig $ Person
    <$> areq textField (bootstrapFieldHelper iConfig "Name" (Just "Person name")) Nothing
    <*> areq textField (bootstrapFieldHelper iConfig "Surname" (Just "Person surname")) Nothing

personForm :: Html -> MForm Handler (FormResult Person, Widget)
personForm = renderBootstrap bConfig $ Person
    <$> areq textField (bootstrapFieldHelper bConfig "Name" (Just "Person name")) Nothing
    <*> areq textField (bootstrapFieldHelper bConfig "Surname" (Just "Person surname")) Nothing

largeDataForm :: Html -> MForm Handler (FormResult LargeData, Widget)
largeDataForm = renderBootstrap largeFormConfig $ LargeData
    <$> areq textField (bootstrapFieldHelper hConfig "Text" (Just "Some text content")) Nothing
    <*> areq intField (bootstrapFieldHelper hConfig "Int" (Just "Some integer value")) Nothing
    <*> areq doubleField (bootstrapFieldHelper hConfig "Double" (Just "Some double value")) Nothing
    <*> areq textareaField (bootstrapFieldHelper hConfig "Area" (Just "Some text area content")) Nothing
    <*> areq hiddenField (bootstrapFieldHelper hConfig "Hidden" (Just "Hidden field")) Nothing
    <*> areq passwordField (bootstrapFieldHelper hConfig "Password" (Just "Password field")) Nothing
    <*> areq emailField (bootstrapFieldHelper hConfig "Email" (Just "Email field")) Nothing
    <*> areq htmlField (bootstrapFieldHelper hConfig "Html" (Just "Some HTML")) Nothing
    <*> areq dayField (bootstrapFieldHelper hConfig "Day" (Just "Some day")) Nothing
    <*> areq timeField (bootstrapFieldHelper hConfig "Time" (Just "Some time")) Nothing
    <*> areq (searchField False) (bootstrapFieldHelper hConfig "Search" (Just "Some search")) Nothing
    <*> areq urlField (bootstrapFieldHelper hConfig "URL" (Just "Some URL")) Nothing
    <*> areq boolField (bootstrapFieldHelper hConfig "Bool" (Just "Some bool")) Nothing
    <*> areq checkBoxField (bootstrapFieldHelper hConfig "Checkbox" (Just "Some checkbox")) Nothing

{-# START_FILE Dispatch.hs #-}

{-# LANGUAGE TemplateHaskell #-}

module Dispatch where

import Yesod

import Foundation

import Handler.Home

mkYesodDispatch "FormApp" resourcesFormApp

{-# START_FILE Handler/Home.hs #-}

{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE TemplateHaskell #-}

module Handler.Home where

import          Yesod
import          Yesod.Default.Util
import          Yesod.Form.Jquery

import          Foundation

getHomeR :: Handler Html
getHomeR = do
    (basicWidget, enctype) <- generateFormPost personForm
    (inlineWidget, enctype) <- generateFormPost personIForm
    (horizontalWidget, enctype) <- generateFormPost personHForm
    (largeWidget, enctype) <- generateFormPost largeDataForm
    defaultLayout $ do
        addStylesheetRemote "//netdna.bootstrapcdn.com/bootstrap/3.1.0/css/bootstrap.min.css"
        $(widgetFileReload def "home")

{-# START_FILE Main.hs #-}

module Main where

import          Yesod
import          Yesod.Form.Jquery

import          Foundation
import          Dispatch

main :: IO ()
main = warpEnv FormApp

{-# START_FILE Form/Bootstrap3.hs #-}

{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE OverloadedStrings #-}

module Form.Bootstrap3 (renderBootstrap,
                        bootstrapFieldSettings,
                        BootstrapFormConfig (..),
                        GridOptions (..),
                        BootstrapForm (..)
                        ) where

import              Yesod hiding (renderBootstrap)

import              Data.Maybe (listToMaybe, fromMaybe)

import              Data.Text (Text, pack)

import              Text.Blaze.Html

data GridOptions = ColXs Int | ColSm Int | ColMd Int | ColLg Int

instance Show GridOptions where
    show (ColXs 0) = ""
    show (ColXs columns) = "col-xs-" ++ show columns

    show (ColSm 0) = ""
    show (ColSm columns) = "col-sm-" ++ show columns

    show (ColMd 0) = ""
    show (ColMd columns) = "col-md-" ++ show columns

    show (ColLg 0) = ""
    show (ColLg columns) = "col-lg-" ++ show columns

instance ToMarkup GridOptions where
    toMarkup = toMarkup . show

toOffset :: GridOptions -> String
toOffset (ColXs 0) = ""
toOffset (ColSm 0) = ""
toOffset (ColMd 0) = ""
toOffset (ColLg 0) = ""
toOffset (ColXs columns) = "col-xs-offset-" ++ show columns
toOffset (ColSm columns) = "col-sm-offset-" ++ show columns
toOffset (ColMd columns) = "col-md-offset-" ++ show columns
toOffset (ColLg columns) = "col-lg-offset-" ++ show columns

data BootstrapForm = BootstrapBasicForm | BootstrapInlineForm
    | BootstrapHorizontalForm { containerOffset :: GridOptions, container :: GridOptions, label :: GridOptions }

data BootstrapFormConfig = BootstrapFormConfig { form :: BootstrapForm, submit :: String }

bootstrapFieldSettings :: BootstrapFormConfig -> SomeMessage site -> Maybe (SomeMessage site)
    -> Maybe Text -> Maybe Text -> Maybe Text -> FieldSettings site
bootstrapFieldSettings formConfig msg tooltip placeholder id name =
    FieldSettings msg tooltip id name (attrsFromFormConfig formConfig placeholder)

attrsFromFormConfig :: BootstrapFormConfig -> Maybe Text -> [(Text, Text)]
attrsFromFormConfig _ Nothing = [("class", "form-control")]
attrsFromFormConfig _ (Just placeholder) = [("class", "form-control"), ("placeholder", placeholder)]

renderBootstrap :: Monad m => BootstrapFormConfig -> FormRender m a
renderBootstrap formConfig aform fragment = do
    (res, views') <- aFormToForm aform
    let views = views' []
        has (Just _) = True
        has Nothing  = False
        widget = [whamlet|
                        \#{fragment}
                        $forall view <- views
                          <div .form-group :fvRequired view:.required :not $ fvRequired view:.optional :has $ fvErrors view:.error>
                            $case (form formConfig)
                              $of BootstrapBasicForm
                                <label for=#{fvId view}>#{fvLabel view}
                                ^{fvInput view}
                                ^{helpWidget view}
                              $of BootstrapInlineForm
                                <label .sr-only for=#{fvId view}>#{fvLabel view}
                                ^{fvInput view}
                                ^{helpWidget view}
                              $of BootstrapHorizontalForm containerOffset containerClass labelClass
                                <label .control-label .#{labelClass} for=#{fvId view}>#{fvLabel view}
                                <div .#{containerClass}>
                                  ^{fvInput view}
                                ^{helpWidget view}

                        ^{submitWidget $ formConfig}
                |]
    return (res, widget)

submitWidget (BootstrapFormConfig (BootstrapHorizontalForm containerOffset containerClass labelClass) submit) = [whamlet|
<div .form-group>
    <div .#{toOffset containerOffset} .#{containerClass}>
      <button type=submit .btn .btn-default>#{submit}
|]
submitWidget (BootstrapFormConfig _ submit) = [whamlet|<button type=submit .btn .btn-default>#{submit}|]

helpWidget view = [whamlet|
    $maybe tt <- fvTooltip view
      <span .help-block>#{tt}
    $maybe err <- fvErrors view
      <span .help-block>#{err}
|]

{-# START_FILE config/routes #-}

/           HomeR       GET

{-# START_FILE templates/home.hamlet #-}

<div .container>

    <div .jumbotron>
        <h2>An example of a basic form
        <div .container>
            <form method=get action=@{HomeR} enctype=#{enctype}>
                ^{basicWidget}

    <div .jumbotron>
        <h2>An example of an inline form
        <div .container>
            <form .form-inline method=get action=@{HomeR} enctype=#{enctype}>
                ^{inlineWidget}

    <div .jumbotron>
        <h2>An example of a horizontal form
        <div .container>
            <form .form-horizontal method=get action=@{HomeR} enctype=#{enctype}>
                ^{horizontalWidget}

    <div .jumbotron>
        <h2>An example of a large horizontal form
        <div .container>
            <form .form-horizontal method=get action=@{HomeR} enctype=#{enctype}>
                ^{largeWidget}

```

## Things that are missing...
...are numerous:

 - showing tooltips breaks inline forms
 - support for some input controls is missing (groups of checkboxes and radiobuttons, file inputs etc.)
 - placeholder attribute for fields is a `Maybe Text`, but should support i18n
 - other things I probably forgot and don't know

This was, after all, a simple exercise in making some basic functionality that I'll continue to use in the near future, at least until a better solution pops up.

There is a companion project for the tutorial [here](https://www.fpcomplete.com/user/msrdic/bootstrap-3-forms-with-yesod) and the complete source is on [github](https://github.com/msrdic/yesod-forms). Hope you find it useful.
 