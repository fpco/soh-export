# mutable-bytestring

```haskell active
{-# START_FILE Data/ByteString/Generic.hs #-}
{-# LANGUAGE FlexibleInstances     #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE TypeFamilies          #-}
module Data.ByteString.Generic where

import qualified Data.ByteString          as S
import           Data.ByteString.Internal (ByteString (PS))
import qualified Data.ByteString.Mutable  as M
import qualified Data.ByteString.Unsafe   as BU
import           Data.Vector.Generic
import           Data.Word                (Word8)

newtype ByteString' a = ByteString'
    { unByteString' :: S.ByteString
    }

instance a ~ Word8 => Vector ByteString' a where
    basicUnsafeFreeze (M.MByteString fptr off len) = return $! ByteString' $! PS fptr off len

    basicUnsafeThaw (ByteString' (PS fptr off len)) =
        return $! M.MByteString fptr off len

    basicLength = S.length . unByteString'

    basicUnsafeSlice start len (ByteString' bs) = ByteString' $! BU.unsafeTake len $! BU.unsafeDrop start bs

    basicUnsafeIndexM (ByteString' bs) idx = return $! BU.unsafeIndex bs idx

type instance Mutable ByteString' = M.MByteString'

{-# START_FILE Data/ByteString/Mutable.hs #-}
{-# LANGUAGE FlexibleInstances     #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE TypeFamilies          #-}
module Data.ByteString.Mutable where

import           Control.Monad.Primitive     (unsafePrimToPrim)
import           Data.ByteString.Internal    (mallocByteString)
import           Data.Vector.Generic.Mutable
import           Data.Word8                  (Word8)
import           Foreign.ForeignPtr          (ForeignPtr)
import           Foreign.ForeignPtr.Unsafe   (unsafeForeignPtrToPtr)
import           Foreign.Ptr                 (Ptr)
import           Foreign.Storable            (peekByteOff, pokeByteOff)

data MByteString' s a = MByteString
    { mbsFptr   :: !(ForeignPtr Word8)
    , mbsOffset :: {-# UNPACK #-} !Int
    , mbsLen    :: {-# UNPACK #-} !Int
    }

mbsPtr = unsafeForeignPtrToPtr . mbsFptr

instance a ~ Word8 => MVector MByteString' a where
    basicLength = mbsLen

    basicUnsafeSlice start2 len (MByteString fptr start1 _) =
        MByteString fptr (start1 + start2) len

    basicOverlaps x y = mbsPtr x == mbsPtr y

    basicUnsafeNew len = do
        fptr <- unsafePrimToPrim $ mallocByteString len
        return $! MByteString fptr 0 len

    basicUnsafeRead mbs idx = unsafePrimToPrim $ peekByteOff (mbsPtr mbs) (idx + mbsOffset mbs)

    basicUnsafeWrite mbs idx w = unsafePrimToPrim $ pokeByteOff (mbsPtr mbs) (idx + mbsOffset mbs) w

{-# START_FILE src/Main.hs #-}
import Data.ByteString.Generic
import Data.ByteString.Mutable
import qualified Data.Vector.Generic as V
import qualified Data.Vector.Generic.Mutable as VM

main = do
    mbs <- VM.replicate 100 66
    VM.write mbs 12 67
    V.freeze mbs >>= print . unByteString'
```