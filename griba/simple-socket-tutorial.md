# Simple socket tutorial

## Simple socket tutorial

Client send messages, server receive and print them. 

I use an old comm. style convention of sending length before the message. If you know the types involved, the length prefix should not be needed.

### Encoding / decoding msg length

ByteString lengths are returned as Int64. Since message lengths are not going to be that big, I constraint length encoding to 2 bytes, validating lengths < 2^16.

```haskell
{-| Validate and encode/decode msg length
       -}
module XmitLenCodec (
  encodeXmitLen,
  decodeXmitLen,
  MsgLen,
  validateMsgLen,
  msgLenEOT,
  isLenEOT,
  lenLen,
  ) where

import           Control.Exception    (assert)
import           Data.ByteString.Lazy (ByteString, pack, unpack)
import qualified Data.ByteString.Lazy as BS
import           Data.Char            (chr, ord)
import           Data.Int             (Int64)
import           Data.Word            (Word16, Word8)
import           Data.Binary          (Binary, encode, decode) 

-- | validated msg length
newtype MsgLen = MsgLen Word16

maxlen = 0x0FFFF :: Int64

lenLen = 2 :: Int64

msgLenEOT = MsgLen 0

isLenEOT :: Int64 -> Bool
isLenEOT = (== 0)

validateMsgLen :: Int64 -> Maybe MsgLen
validateMsgLen len = if len <= maxlen && len >= 0
                        then Just $ MsgLen $ fromIntegral len
                        else Nothing

-- | encode with Binary                        
encodeXmitLen :: MsgLen -> ByteString
encodeXmitLen (MsgLen w) = encode w

-- | decode with Binary                        
decodeXmitLen :: ByteString -> Int64
decodeXmitLen bs =
      assert (BS.length bs == lenLen) $
          let w = decode bs :: Word16
          in fromIntegral w 
```


### Sending / recieving a sequence of messages

To support a sequence of messages of unknown length, I give the receive function a type suitable to be used with a monadic `unfoldrM`. 

```haskell
{-| send / receive msgs / EOT -}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE PackageImports   #-}
{-# LANGUAGE Rank2Types       #-}
module SendReceive (sendAsBS, sendEOT, receiveMay) where

import           Control.Monad
import           Data.Binary              (Binary, decode, encode)
import           Data.ByteString.Lazy     (ByteString)
import qualified Data.ByteString.Lazy     as BS
import           Debug.Trace              (traceIO)
import           "network" Network.Socket
import qualified "network" Network.Socket.ByteString.Lazy as NSBS
import           Text.Printf

-- app modules
import           XmitLenCodec      (decodeXmitLen, encodeXmitLen,
                                    isLenEOT, lenLen, msgLenEOT,
                                    validateMsgLen)

-- | send msg
sendAsBS :: Binary a => Socket -> a -> IO ()
sendAsBS skt myData =
    case validateMsgLen len of
      Nothing -> traceIO $ -- error as trace msg
                    printf "sendAsBS: msg length %d out of range" len

      Just msgLen -> do
        NSBS.send skt $ encodeXmitLen msgLen
        bytes <- NSBS.send skt bsMsg
        when (bytes /= len) $ traceIO "sendAsBS: bytes sent disagreement"
        -- return ()
  where
    bsMsg = encode myData
    len = BS.length bsMsg

-- | send EOT
sendEOT :: Socket -> IO ()
sendEOT skt = do
        NSBS.send skt $ encodeXmitLen msgLenEOT
        return ()

-- | type alias for a monadic unfoldrM
--   adding a context here requires the language extensions 
--   Rank2Types, FlexibleContexts
type UnfoldableM m b a = Monad m => b -> m (Maybe (a, b))

-- | receive EOT or msg
receiveMay :: Binary a => Socket -> UnfoldableM IO b a
receiveMay skt st = do
        len <- fmap decodeXmitLen $ NSBS.recv skt lenLen
        if isLenEOT len
           then return Nothing
           else do
             bs <- NSBS.recv skt len
             return $ Just (decode bs, st)
```

### Setting up client, server and connections

Wrapping `receiveMay` with unfoldrM for an unknown number of messages to receive, gathering its results in a list.

It would have been easier to send the list as a whole in a unique message, since there is a Binary instance for lists, but I wanted to test sending a sequence of messages. 

```haskell
{-| file Sockets.hs -}
{-# LANGUAGE PackageImports #-}
module Sockets (client, server) where

-- import           Prelude                        as P
import           Text.Printf                    (printf)
import           System.IO                      (stdout, hFlush) 
import           Control.Exception              (bracket)
import           Control.Monad
import           Debug.Trace                    (traceIO)
import           Numeric                        (showHex)
import           "monad-loops" Control.Monad.Loops  (unfoldrM)
import qualified "network" Network.BSD                    as BSD
import           "network" Network.Socket
import qualified "network" Network.Socket.ByteString.Lazy as NSBS
import           Data.Int

-- | app modules
import           SendReceive       (receiveMay, sendAsBS, sendEOT)

type XmitType = Int32

withSocket ::  Family -> SocketType -> ProtocolNumber -> 
                  (Socket -> IO a) -> IO a
                  
withSocket family sktType protocol = 
    bracket (socket family sktType protocol)  -- acquire resource
            (\skt -> shutdown skt ShutdownBoth >> close skt) -- release

withAccept :: Socket -> ((Socket, SockAddr) -> IO a) -> IO a
withAccept skt = bracket (accept skt)          -- acquire
                         (\(connSkt, _) -> do  -- release
                                shutdown connSkt ShutdownBoth
                                close connSkt) 

        
-- | server accept connection, receive some msgs, gather results and print
acceptConn :: Socket -> HostAddress -> IO ()
acceptConn skt expectedHost = do
   res <- withAccept skt $ \(connSkt, connSktAddr) ->
            case connSktAddr of
               SockAddrInet _ originHost 
                 | originHost == expectedHost ->
                    -- read msgs until EOT
                    -- wrap receiveMay with unfoldrM 
                    -- for an unknown number of msgs to receive
                    unfoldrM (receiveMay connSkt) 0 :: IO [XmitType]
               _ -> return [] 
   ------------------------
   -- print received data
   putStrLn "received:"    
   forM_ res print


-- | client start / send a few data msgs, then EOT
-- withSocketsDo required in Windows platform for initialisation, 
-- no-op otherwise
client :: PortNumber -> IO ()
client port = withSocketsDo $ do  
   protocol <- fmap BSD.protoNumber $ BSD.getProtocolByName "TCP"
   withSocket AF_INET Stream protocol $ \skt -> do

       localhost <- inet_addr "127.0.0.1"
       -- traceIO $ "localhost: " ++ showHex localhost ""
       let sktAddr = SockAddrInet port localhost
       connect skt sktAddr

       -- send msgs, EOT
       let testList = [1..4] :: [XmitType]
       putStrLn "to send:"    
       forM_ testList print   -- print test msgs
       hFlush stdout
       forM_ testList $ \n -> sendAsBS skt n
       sendEOT skt

-- | server side params
maxConnToListenTo = 5

-- | server start / shutdown
-- withSocketsDo required in Windows platform for initialisation, 
-- no-op otherwise
server :: PortNumber -> IO ()
server port = withSocketsDo $ do  
   protocol <- fmap BSD.protoNumber $ BSD.getProtocolByName "TCP"
   withSocket AF_INET Stream protocol $ \skt -> do

       localhost <- inet_addr "127.0.0.1"
       let sktAddr = SockAddrInet port localhost
       bind skt sktAddr
       listen skt maxConnToListenTo
       ---------------------
       acceptConn skt localhost -- accept once from LocalHost
```

### Starting client and server

Using port 3000 for tests in FPComplete server.

Server and Client threads controlled with *async* library.

```haskell
{-| file Main.hs -}
{-# LANGUAGE PackageImports #-}
module Main where

import           "async" Control.Concurrent.Async (wait, withAsync)
import           Control.Monad
import           "network" Network.Socket           (PortNumber (PortNum))
import           "safe" Safe                     (readMay)
import           System.Environment       (getArgs, getProgName)
import           System.Exit              (ExitCode (..), exitWith)
import           Text.Printf              (printf)

-- | app modules
import           Sockets                  (client, server)


-- | thread control
process :: PortNumber -> IO ()
process nPort =
  withAsync (server nPort) $ \asyncServer ->
    withAsync (client nPort) $ \asyncClient -> do
      wait asyncClient
      wait asyncServer
      putStrLn "end of program"

-- | usage info
usage :: IO ()
usage = do
        prog <- getProgName
        printf "%s port (e.g.: 8007)\n" prog

-- | The main entry point.
main :: IO ()
main = do
   args <- getArgs
   case args of
      [sPort] ->
         case readMay sPort of
             Just nPort -> process $ PortNum nPort
             Nothing -> do
                 printf "unreadable port number %s\n" sPort
                 usage
                 exitWith (ExitFailure 1)

      _ -> -- using fixed port number 3000, 
           -- accepted for test in FPComplete projects 
           process $ PortNum 3000
```

