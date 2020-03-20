# A Tic-tac-toe board

``` active haskell
import Data.Functor
import Data.List as L
import Data.Stream.Infinite as S

data Cell = X
          | O
          | Empty
   deriving Eq

instance Show Cell where
    show X = "X"
    show O = "O"
    show Empty = "_"

type Board a = Stream (Stream a)

emptyBoard :: Board Cell
emptyBoard = S.repeat $ S.repeat Empty

updateInStream :: Int -> (a -> Maybe a) -> Stream a -> Maybe (Stream a)
updateInStream i f s = 
    let (l, x :> xs) = S.splitAt i s
    in case f x of
        Nothing -> Nothing
        Just x' -> Just $ foldr (:>) (x' :> xs) l

updateInBoard :: Int -> Int -> (a -> Maybe a) -> Board a -> Maybe (Board a)
updateInBoard row col f = updateInStream row (updateInStream col f)

diagonal :: Board a -> Stream a
diagonal board = S.zipWith (S.!!) board (S.iterate succ 1)

extractSquare :: Int -> Board a -> [[a]]
extractSquare n board = S.take n $ fmap (S.take n) board

reverseDiagonal :: Int -> Board a -> [a]
reverseDiagonal n board = L.zipWith (L.!!) (reverse . extractSquare n $ board) [1..]

boardLines :: Int -> Board a -> [[a]]
boardLines n board =
    let rows = extractSquare n board
        cols = extractSquare n $ S.transpose board
    in S.take n (diagonal board) : reverseDiagonal n board : (rows ++ cols)

nInRow :: Int -> Cell -> Board Cell -> Bool
nInRow n cell board =
    let cells = replicate 3 cell
    in not . null $ L.filter (==cells) $ boardLines n board

renderLine :: [Cell] -> String
renderLine = L.concat . fmap show

render :: Int -> Board Cell -> String
render n table =
    L.concat $ L.intersperse "\n" $ fmap renderLine $ extractSquare n table

putPiece :: (Int,Int) -> Cell -> Board Cell -> Maybe (Board Cell)
putPiece (row,col) cell board =
    let setIfEmpty cell previousCell = case previousCell of
           Empty -> Just cell
           _ -> Nothing
    in updateInBoard row col (setIfEmpty cell) board

main :: IO ()
main = do
    let boardMaybe = 
                putPiece (2,2) X emptyBoard
            >>= putPiece (2,1) X
            >>= putPiece (2,0) X
            >>= putPiece (0,1) O
            >>= putPiece (1,1) O
            >>= putPiece (1,2) O
    case boardMaybe of
        Just board -> do
            putStrLn $ render 3 $ board
            putStrLn $ "O won? " ++ (show $ nInRow 3 O board)
            putStrLn $ "X won? " ++ (show $ nInRow 3 X board)
        Nothing -> putStrLn "Wrong piece position."
    putStrLn "Result of inserting into already occupied cell:"
    let boardMaybe' = boardMaybe >>= putPiece (2,1) X
    case boardMaybe' of
        Just board -> do putStrLn $ render 3 $ board
        Nothing -> putStrLn "Wrong piece position."
```