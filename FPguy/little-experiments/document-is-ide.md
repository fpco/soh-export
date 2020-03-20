# This document is sort of an IDE

Imagine if you could run Haskell code like this right in a document.

 
``` haskell
main = putStrLn "Hello!"
```

You can! Try pressing the Run button in the following box.


``` active haskell
main = putStrLn "Server-side Haskell says hi!"
```


But the text in the box above is editable, with interactive syntax highlighting. 
And whatever's there will really be compiled and run!
To prove this, delete the Haskell code in the box above and paste this in instead:

    main = print $ map (*5) [1..50]

Then press Run. See? And feel free to edit it as much as you want. Have a Haskell
party -- it's real.


You may have also noticed: 
if you click on a Haskell keyword or operator, in a moment help will appear just 
below the box.


What looks like an interactive blog is actually a cloud-based Haskell IDE.

Greetings from FP Complete. This is just the beginning.



[Milewski](http://haskell.fpcomplete.com/user/bartosz) has written more tutorials.
[Snoyman](http://haskell.fpcomplete.com/user/snoyberg) has too.
Please try writing your own!