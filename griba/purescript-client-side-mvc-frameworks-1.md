# PureScript client-side MVC frameworks

# PureScript as a full language replacement for Elm

[Elm](http://elm-lang.org) is a client side functional programming framework that has made a trend in Web client frameworks. With the aim of making things easier [it has dropped a nice functional reactive programming model](http://elm-lang.org/blog/farewell-to-frp) and has adopted a Model-View-Controller architecture as the functional blocks that the [object-oriented *reactJS*](https://reactjs.org/tutorial/tutorial.html#what-is-react) system offers. But its language has no polymorphism for containers `(* -> *)`, and one for simple types called "type categories" which values *(number, comparable, appendable)* are hardwired in the compiler, making it a rather limited subset of what the language it is inspired in (Haskell) offers.

See [Elm from a Haskell perspective](https://www.schoolofhaskell.com/user/griba/elm-from-a-haskell-perspective).

[PureScript](http://taylor.fausak.me/static/pages/2015-10-22-better-know-a-language-purescript.html#1) offers alternatives for designing Web User Interfaces much nearer to the language power of Haskell and much lighter weight than GHCJS (no GHC RTS emulation).

See also [PureScript introduction and recursively defined lazy lists](https://www.schoolofhaskell.com/user/griba/lazy_lists_on_strict_purescript).


## PureScript Pux

+ [Pux](http://purescript-pux.org/) is the PureScript simpler framework, similar to Elm except for effects. Subscriptions use an [Elm's old FRP](http://elm-lang.org/blog/farewell-to-frp) Signal [analog](http://purescript-pux.org/docs/rendering/). It uses the [smolder pkg](https://pursuit.purescript.org/packages/purescript-smolder) for describing the markup. But it has [performance issues as stated in its web site](http://purescript-pux.org/#Why_is_Pux_slow) as of August 2018. Check their [benchmarks](http://purescript-pux.org/#Benchmarks).

Here is an example from the Pux web, of an uncommon [multi-URL example](http://purescript-pux.org/docs/routing/) that simulates routing. Instead of fetching URL's by HTTP, it decodes them to an Event type with a Route, which changes the State's *currentRoute* in the controller and renders it in the View, pushing the target URL into the navigator's history stack, and also turns history pops (Back button presses) in Events in an Elm's subscription like manner. I have reordered the snippets and indented some parts, to highlight the MVC constituents:

```haskell
{-| MVC highlights of the Pux multi-page (multi-URL) sample app.
    * Here the Language is PureScript !!! 
    * Square bracket literals denote type Array -}

-- the model
data Route = Home | Users String | User Int | NotFound  -- the pages
type State = {currentRoute :: Route}

-- the message
data Event = PageView Route | Navigate String {- the URL -} DOMEvent

-- the view
view :: State -> Html Event
view state = div do
                navigation
                page state.currentRoute
    where
        -- content
        page :: Route -> Html Event
        page Home           = h1 $ text "Home"
        page (Users sortBy) = h1 $ text ("Users sorted by:" <> sortBy)
        page (User id)      = h1 $ text ("User: " <> show id)
        page NotFound       = h1 $ text "Not Found"

        navigation :: Html Event
        navigation =
          nav do
            ul
              li $ a ! href "/" #! onClick (Navigate "/") $ text "Home"
              li $ a ! href "/users" #! onClick (Navigate "/users") $ text "Users"
              li $ a ! href "/users?sortBy=age" #! onClick (Navigate "/users?sortBy=age") $ text "Users sorted by age."
              li $ a ! href "/users/123" #! onClick (Navigate "/users/123") $ text "User 123"
              li $ a ! href "/foobar" #! onClick (Navigate "/foobar") $ text "Not found"
              
-- the URL to (PageView Route) decoder
match :: String -> Event
match url = PageView $ fromMaybe NotFound $ router url $
  Home <$ end
  <|>
  Users <$> (lit "users" *> param "sortBy") <* end
  <|>
  Users "name" <$ (lit "users") <* end
  <|>
  User <$> (lit "users" *> int) <* end    

-- the controller
foldp :: ∀ fx. Event -> State -> EffModel State Event (history :: HISTORY, 
                                                       dom :: DOM | fx)
foldp (PageView route) st =  noEffects $ st { currentRoute = route }
foldp (Navigate url ev) st =
  onlyEffects st [ liftEff do
                     preventDefault ev
                     
                     -- push url and title into the Navigator history
                     h <- history =<< window
                     pushState (toForeign {}) (DocumentTitle "") (URL url) h
                     
                     -- emit a PageView message 
                     -- `match` decodes the URL into a (PageView Route) Event message
                     pure $ Just $ match url
                 ]

main = do
  -- sampleURL emits a Signal on navigator history pops (as a result of the Back button)
  urlSignal <- sampleURL =<< window
  let routeSignal = urlSignal ~> match  -- Signal to PageView Event "natural transf."
  app <- start
    { initialState: { currentRoute: Home }
    , view
    , foldp
    , inputs: [routeSignal]
    }
```

In simpler apps the [rendering setup](http://purescript-pux.org/docs/rendering/) would tell whether to render

+ to a browser DOM Element, with *renderToDOM*
+ to console output, with *renderToString*

## PureScript Thermite

+ [Thermite](https://github.com/paf31/purescript-thermite/) is a Purescript-react framework that offers some component combination possibilities, each with its model state, event messages and controller. 

	1. you may combine several components of equal model and message types into an app, with the component *Semigroup* operator `(<>)`. 

    	+ to combine components with different models, there are combinators that let you focus with a [lens](https://www.schoolofhaskell.com/user/griba/easier_lenses_profunctor_based_with_mezzolens),  the specific model type from the group model product type, or a prism in case of optional components.
        
        + to combine components with different messages (called *actions*), there are combinators that let you focus with a prism the specific message type from the group message sum type, but sharing the same message type enables mutual interaction.
        
    	Combining independent components:
        
            
            spec1 :: Spec _ S1 _ A1
            spec2 :: Spec _ S2 _ A2
            --
            groupSpec :: Spec _ (Tuple S1 S2)  -- group model
                              _ (Either A1 A2) -- group action
                              
                              
            groupSpec = focus _1    -- apply lens _1 to the group model
                              _Left -- apply lens _Left to the group action
                              spec1 -- component to focus
                              
                        <> focus _2 _Right spec2
            
        
        <!-- Combining interactive components:
            -- both components process all sent messages
            spec1 :: Spec _ S1 _ A
            spec2 :: Spec _ S2 _ A
            --
            groupSpec :: Spec _ (Tuple S1 S2) _ A
            groupSpec = focusState _1 spec1 <> focusState _2 spec2
             -->
            

	2. you may also build a group of equally typed components as a List, where the state is a List of the states, and the message (called *action*) is indexed with the index of the originating subcomponent in the List.

## PureScript Halogen

+ [Halogen](https://github.com/slamdata/purescript-halogen/) is a somewhat more complex Purescript framework.

	* A component may include [*slots*](https://pursuit.purescript.org/packages/purescript-halogen/3.1.3/docs/Halogen.HTML#v:slot) for subcomponents in the HTML DSL, enabling bidirectional parent-child communication reflected in its generator parameters:
    
        * A value of type *Slot* (the [*slot address*](https://github.com/slamdata/purescript-halogen/blob/v4.0.0/docs/5%20-%20Parent%20and%20child%20components.md#user-content-slot-address)) is used as slot identifier and makes possible to send synchronous [*request*](https://github.com/slamdata/purescript-halogen/blob/v4.0.0/docs/2%20-%20Defining%20a%20component.md#query-algebra) [*queries*](https://pursuit.purescript.org/packages/purescript-halogen/2.0.1/docs/Halogen.Query#v:query) from within the controller, to the slot subcomponent.
    
    	* Slot "input" values (at the slot's input parameter) are a means of passing values into a child component every time a parent re-renders, to be handled through the subcomponent [*receiver* field](https://pursuit.purescript.org/packages/purescript-halogen/3.1.3/docs/Halogen.Component#t:ComponentSpec) "input" handler where you may tag the incoming message to be processed by the subcomponent controller

		* The slot "output" handler parameter: a subcomponent may [*raise*](https://pursuit.purescript.org/packages/purescript-halogen/3.1.3/docs/Halogen.Query.HalogenM#v:raise) messages to its parent container component to be handled through the slot's "output" handler where you may tag the incoming message to be processed by the component controller

	* The component state, instead of given as parameter to the *eval* controller, it is held in the controller's MonadState monad.
    
    * There is a weird thing in the way messages, called [Queries](https://github.com/slamdata/purescript-halogen/blob/v4.0.0/docs/2%20-%20Defining%20a%20component.md#query-algebra), are defined, because the system enforces the return value of the *eval* controller to be determined by the *query* message content or, in case of *request queries*, the result of its callback component, which has a strange type of `(queryResult -> a)` (see *[Query evaluation](https://github.com/slamdata/purescript-halogen/blob/v4.0.0/docs/2%20-%20Defining%20a%20component.md#query-evaluation)*). This is enforced by defining the controller type as a  [natural transformation](https://pursuit.purescript.org/packages/purescript-prelude/4.0.1/docs/Data.NaturalTransformation): an homomorphism between structures, the query message and the controller monad, threading some obscure value through them.
