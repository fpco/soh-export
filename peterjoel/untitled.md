# Untitled

import Data.Monoid;

     mempty :: a -> m a
     mempty = return

     mappend :: (a -> m b) -> (b -> m c) -> (a -> m c)
     mappend = (>=>)