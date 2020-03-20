# Code snippets that might be useful

Following the steps of this <a href="https://www.fpcomplete.com/school/starting-with-haskell/libraries-and-frameworks/basics-of-yesod/yesod-1"> tutorial</a> I ran into an issue as to how to enable login in an fp ide application. In fpcomplete we don't have access to the url for our server to paste it directly in the source file.

This is how I approached this: it is ugly and unsafe all over. I am certain that there is a better way, but this one-liner seems to work:

``` active haskell
instance Yesod Piggies where    
    -- Need to use the get env function to set this value.
    -- approot should get picked up from the settings.yml.    
approot =  ApprootStatic $ pack $ unsafePerformIO $ getEnv "APPROOT"

```


So, as it turns out, there is indeed a better way: use <a href="http://stackoverflow.com/a/20897235/2976249">scaffolding</a>. Here is the relevant solution

```active haskell
data Piggies = Piggies
    {
    connPool    :: ConnectionPool,
    httpManager:: Manager,
    staticURL :: Text
    }

instance Yesod Piggies where
    -- This is the real one-liner. ApprootMaster knows to 
    -- setup the root in terms of the staticURL function call
    -- 
    approot = ApprootMaster staticURL
```

Here is the <a href="http://hackage.haskell.org/package/yesod-core-1.2.6.2/docs/Yesod-Core.html"link</a> to the ApprootMaster type

> ApprootMaster !(master -> Text)

In this case staticURL is simply that. 




