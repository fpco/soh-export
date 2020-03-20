# Java

```haskell active
{-# START_FILE Test.java #-}
public class Test {
    public static void main(String[] args) {
        System.out.println("Hello from Java!");
    }
}
{-# START_FILE main.hs #-}
import System.Cmd
main = do
    putStrLn "Hello from Haskell!"
    rawSystem "javac" ["Test.java"]
    rawSystem "java" ["Test"]
```