# OTAS Quant 1: Position Keeping

# Introduction

When working in quant finance, you often need to track your current position; what this really means is, given a list of transactions, calculate how much stock and how much cash you have at the end. I'm going to talk about stock here, but the same method should work for contracts of any financial instrument. I'm going to ignore FX, and we're just going to consider the case of a single line of stock. 

A couple of years ago I wrote code to do this in Haskell. It was fairly concise, robust and easy for people to understand. We used it as the reference specification for position tracking functionality in one of our products. However, I always felt that the code could be improved. So I rewrote it using Lens and the State monad, and the result is a huge improvement on the original. In the process, I think I've finally been convinced of the utility of State and Lens. In the past, when I tried either on their own, I just didn't see enough of a gain for the cost of the additional layer. But stick them together, and it's just awesome! 

Caveat: this is the first time I've ever really used Lens and State together, I'm probably missing some better ways of doing some of this. If you see any areas for improvement, please let me know and I'll update the content.  

# Transactions

Let's define the transactions that we need to process

``` haskell
data Transaction = Transaction {transSide::Side, transPrice::Price, transQty::Qty}
                   |Dividend Amount
                   |Split SplitRatio
                    deriving (Eq, Show)
                    
data Side = Buy | Sell deriving (Eq, Show)

```

This should be easy to understand; a transaction is either a regular Buy or Sell of some stock, or we can have a dividend payment which is an amount of money for each share held, or a corporate action like a stock split in which you receive a number of shares for each share you already hold. 


# Our Book

We represent the state of our position like this 

``` haskell
data Book = Book
  { _cash       :: Amount
  , _shareCount :: Qty
  } deriving (Eq, Show)

makeLenses ''Book

```
Note that I'm using Lenses here so I've prefixed the field names with an underscore and called makeLenses. This is all that's needed to generate all the code required to access and update the fields. 

# Keeping Track of Positions

Now we're ready to update the Book based on a Transaction, so we need an update function that takes a Transaction and updates the State of the Book:


``` haskell
update :: Transaction -> State Book ()

update (Transaction Buy p q) = do
  cash -= toAmount p q
  shareCount += q

update (Transaction Sell p q) = do
  cash += toAmount p q
  shareCount -= q

update (Dividend amt) = do
  -- check if there's a way to do this in one line
  count <- use shareCount
  cash += toAmount amt count

update (Split ratio) = do
  shareCount *= MkQty ratio

```

Thanks to State and Lens working together, the code is practically self-documenting and there's almost nothing apart from the business logic: a Buy reduces your cash by the amount of the purchase, and increases your share position by the amount you bought. Sells increase your cash and reduce your position. Dividends increase your cash, while Splits change your position. 

What's amazing here (to me at least!) is the way that the Lens operators like (+=), (-=), work together with State. The first time I tried writing this, I assumed they wouldn't, so it looked more like this:

``` haskell
update (Transaction Buy price quantity) = do
    c <- gets cash
    q <- gets shareCount 
    put (Book (c - (p * q)) (q+quantity))
    
```

By the way, I didn't figure out that (+=) would take care of the getting and setting in State by reading the documentation for Lens. Even though it's probably clear if you follow the types, it wasn't clear to me. I only compiled it because I wanted to see what the error message would be(!). I found most introductions and tutorials on Lens had pages of stuff on toy examples or abstract theory, which is a shame when there's this pot of gold waiting at the end. 

# Putting it together

We have the update function that will handle one Transaction, so we need to turn that into something that can handle a whole bunch of them. 

``` haskell
processTransaction :: Book -> Transaction -> Book
processTransaction b t = b'
  where
    (_, b') = runState (update t) b

position :: [Transaction] -> Book
position = foldl' processTransaction (Book 0 0)

```

The first function takes care of running update in the state monad, and extracting the result, the second gives us the top level function for converting a list of Transactions into a final position Book. (I'm sure there must be a way of doing this in one step in a single runState, I just haven't had time to look into it yet). 



# A Footnote on the basic types 

In Quant.Basic.Types, I've made newtypes for Qty and Price based on Decimal. This is quite important, most beginners would just use Doubles for these types. Using Double is wrong. Prices and quantities are not Doubles, they are Decimals. If you just use Doubles for everything, you will have bugs where you mix prices and quantities incorrectly. You'll also have bugs because you use division on Doubles without thinking what that really means (hint: you almost never want to divide prices, quantities or amounts, and when you think you do, being forced to think about it is a good thing, Decimal doesn't have a / operator, because it's not a member of the Fractional class). 

If you weren't aware of this, you'd probably model your book like this:

``` haskell
data Book = Book { averageEntryPrice:: Double, cash::Double, shareCount :: Double}...
```
because on first consideration, it's reasonable, and common, to summarise a bunch of transactions on the same stock by saying "the average price we paid was X and we now have Y shares". No trader expects you to say how much in total you've paid for the shares, because most of the time they know what the current price of the stock in the market is, and they want to know whether the overall position is profitable. So it's ok to say that to summarise the situation, but using average prices in calculations is always the wrong thing to do. Calculating an average price means you're rounding somewhere, so you're losing information. This makes it more difficult to test your code, because now instead of checking for exact equality, you have to allow for rounding errors. 

In practice, the whole average price issue really does cause problems that cost businesses real money. If your broker uses a different rounding scheme to your custodian, they will disagree on the amount of money that needs to be transferred to settle your trades. These "breaks" as they are known, have consumed thousands of hours of people in the settlement departments. [These firms are the ones who manage your savings, so you end up paying].

# Combining the types

Programmers are sometimes frustrated by the fact that once they're using the correct types, they can't just write amount = price * qty. Since they're different types, the compiler will reject any attempts to combine them using normal numeric operators. 

The fix for this is to recognise that there are actually only a small number of sensible ways to combine these types, and have the module provide functions for combining them. A perfectly workable solution is to provide helper functions in the module that defines these types. 

``` haskell

calcAmount :: Price -> Qty -> Amount
calcAmount (MkPrice p) (MkQty q) = MkAmount (p*q)

multAmount :: Amount -> Qty -> Amount
multAmount (MkAmount a) (MkQty q) = MkAmount (a * q)
```

Here we've unwrapped the Price and Qty to get the underlying Decimal, done the multiplication and wrapped it back into an Amount. This is fine, and you could create other functions for the other cases. NB: these functions would be defined in Quant.Base.Types, we're not suggesting you'd write this code when you use these types. 

While this is fine, I don't like the fact that we end up with a bunch of functions that are just floating around. It starts to get difficult to name them well, and it feels like we should be able to group them together. For this reason, I created a typeclass called AmountMaker. AmountMaker defines one function: toAmount. Now we just define an instance of AmountMaker for any pair of types that we want to be able to create an Amount out of. 

The advantage to the user of the library is pretty clear, instead of needing to remember or look up function names like calcAmount, you just use toAmount. 

``` haskell

class AmountMaker a b where
  toAmount :: a -> b -> Amount

instance AmountMaker Price Qty where
  toAmount (MkPrice p) (MkQty q) = MkAmount (p*q)

instance AmountMaker Amount Qty where
  toAmount (MkAmount p) (MkQty q) = MkAmount (p*q)

```

There's probably a bunch of well-studied mathematical structures for combining types like this. Maybe we'll take advantage of them in future. For now, I'm happy with how much cleaner this code is to read and use than the original version. 







