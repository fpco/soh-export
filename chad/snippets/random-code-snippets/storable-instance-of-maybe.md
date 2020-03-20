# Storable instance of Maybe

```haskell active
import           Data.Typeable             (Typeable, typeOf)
import qualified Data.Vector.Storable      as VS
import           Data.Word                 (Word8)
import           Foreign.Ptr               (Ptr, castPtr)
import           Foreign.Storable          (Storable, alignment, peek,
                                            peekByteOff, poke, pokeByteOff,
                                            sizeOf)
import           Test.Hspec                (Spec, hspec, shouldBe)
import           Test.Hspec.QuickCheck     (prop)
import           Test.QuickCheck.Arbitrary (Arbitrary)

instance Storable a => Storable (Maybe a) where
    sizeOf x = sizeOf (stripMaybe x) + 1
    alignment x = alignment (stripMaybe x)
    peek ptr = do
        filled <- peekByteOff ptr $ sizeOf $ stripMaybe $ stripPtr ptr
        if filled == (1 :: Word8)
            then do
                x <- peek $ stripMaybePtr ptr
                return $ Just x
            else return Nothing
    poke ptr Nothing = pokeByteOff ptr (sizeOf $ stripMaybe $ stripPtr ptr) (0 :: Word8)
    poke ptr (Just a) = do
        poke (stripMaybePtr ptr) a
        pokeByteOff ptr (sizeOf a) (1 :: Word8)

stripMaybe :: Maybe a -> a
stripMaybe _ = error "stripMaybe"

stripMaybePtr :: Ptr (Maybe a) -> Ptr a
stripMaybePtr = castPtr

stripPtr :: Ptr a -> a
stripPtr _ = error "stripPtr"

test :: (Arbitrary a, Typeable a, Show a, Storable a, Eq a) => a -> Spec
test dummy = prop (show $ typeOf dummy) $ \vals ->
    let v = VS.fromList vals `asTypeOf` VS.singleton (Just dummy)
     in VS.toList v `shouldBe` vals

main :: IO ()
main = hspec $ do
    test (undefined :: Int)
    test (undefined :: Char)
    test (undefined :: Double)
    test (undefined :: Bool)
```