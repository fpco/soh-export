# Azúcar sintáctico y otros artilugios

En este tutorial conoceremos algunas formas de azúcar sintáctico y otros mecanismos que nos harán escribir código más corto. Aunque no nos darán más poder de cómputo, nos darán mayor poder de expresividad; algo dulce, algo que nos haga la vida más fácil[.](http://img4.wikia.nocookie.net/__cb20131008205703/adventuretimewithfinnandjake/images/8/83/S5_e38_Jake_eating_a_cookie.PNG)

Ya hemos visto un poco de azúcar sintáctico cuando hablamos de las listas y de como podríamos expresarlas con tipos de datos algebraicos recursivos en vez de su azúcar sintáctica: `[`,`]` y `,`; con eso qudó claro que el azucar sintáctico nos ofrece una alternativa menos tediosa para alguna expresión. Técnicamente este tutorial no es sobre programación funcional, sino de azúcar sintáctico y otras expresiones útiles para Haskell.

## Operador de aplicación ($)
El operador de aplicaicón `$` es redundante en Haskell, pues lo mismo es `f p` que `f $ p`, donde `f` es una función y `p` es un argumento. Por ejemplo:
```active haskell
main =
  do
    print (even 2)
    print $ even 2
```
Sin embargo, aunque ambas formas producen el mismo resultado, la segunda utiliza menos paréntesis; estar cerrando paréntesis es tedioso, consume tiempo e interrumpe el flujo del pensamiento.

`$` es un *operador infijo asociativo a la derecha con prioridad baja*.

*Operador infijo* significa que `$` no se utiliza como función, sino como operador que toma dos argumentos, los cuales se colocan a los lados; que sea *asociativo a la derecha* lo diferencía de los operadores infijos asociativos a la izquierda, como el operador `/`; mientras `1/2/3 = ((1/2)/3)` (asociativo a la izquierda), `f1 $ f2 $ p = (f1 $ (f2 $ p))` (asociativo a la derecha). Que tenga *prioridad baja* significa que en caso de haber más operadores en una misma expresión, primero se asociarán los de mayor prioridad. Por ejemplo, `even $ 4 / 2 + 1` se asocia de esta manera: `even $ ((4 / 2) + 1)`, pues las prioridades de `$`, `+` y `/` son tal que *pr($) < pr(+) < pr(/)*, donde *pr(op)* es la prioridad de un operador *op*, o en otras palabras, primero se asocia la división: `(4 / 2)`, después la suma: `(4 / 2) + 1` y al final `$`: `even $ ((4 / 2) + 1)`.

> ([Esta](https://www.haskell.org/onlinereport/haskell2010/haskellch4.html#x10-820004.4.2) es una lista de la prioridad de los operadores de Haskell)

`f1 $ f2 $ p` se traduce a `($) f1 (($) f2 p)` cuando `$` se utiliza como función `($)` en vez de como parámetro. Lo menciono para que quede claro que no hay nada excepcional en `$`; incluso Haskell le permite al usuario definir sus propios operadores con la asociatividad y precedencia que uno escoja.

Ejemplos del uso de `$`:
```active haskell
main =
  do
    print $ even $ mod 6 4
-- equals to:
    ($) print (($) even (($) mod 6 4))
-- that since ($) is redundant is equal to:
    print (even (mod 6 4))
    
-- testing precedence of "$" versus "+"
    print $ even $ 6 + 3
```

A partir de ahora, lo usaremos mucho[.](http://t.qkme.me/3qlrv3.jpg)

## Instancias derivadas (derived instances)
Esto es de lo más *mágico* que tiene Haskell y sólo veremos la intuición en esta sección. En el tutorial [Clases de tipos](https://www.fpcomplete.com/user/XookDo/introducci-n-a-la-programaci-n-funcional/parte-5/tutorial) daremos algunas explicaciones más detalladas.

### Derivación de Eq
Supongamos que tenemos un tipo de dato para los días de la semana:
```haskell
data Day = Monday | Tuesday | Wednesday | Thursday | Friday | Saturday | Sunday
```
Si los necesitaras comparar, podrías hacer una función algo así:
```haskell
compareDays d1 d2 =
  case (d1, d2) of
  (Monday, Monday) -> True
  (Monday, _) -> False
  (Tuesday, Tuesday) -> True
  (Tuesday, _) -> False
  ...
```
Una manera más corta sería algo como esto:
```haskell
compareDays d1 d2 =
  dayToInt d1 == dayToInt d2
  where dayToInt d = case d of
                     Monday -> 1
                     Tuesday -> 2
                     ...
```
Pero afortunadamente podemos "derivar la instancia de `Eq`" para `Day` y obtener los operadores `==` y `/=` gratis:
```active haskell
data Day = Monday | Tuesday | Wednesday | Thursday | Friday | Saturday | Sunday {-hi-}deriving Eq{-/hi-}
main = do
  print $ Saturday {-hi-}=={-/hi-} Saturday
  print $ Friday {-hi-}/={-/hi-} Wednesday
```

### Derivación de Show
También se puede derivar una instancia de la "clase" `Show`; esto nos permite usar la función `show` sobre nuestros propios tipos; no es la única manera de usar `show` sobre nuestros tipos, pero esta es la más fácil y especialmente útil para escribir tutoriales.

```active haskell
data Color = Black | White | Gray {-hi-}deriving Show{-/hi-}

main = putStrLn.{-hi-}show{-/hi-} $ White
```

## Tuplas
Las tuplas son azucar sintáctico para la multiplicación de tipos. Por ejemplo, si necesitamos una estructura de datos con tres valores, en vez de escribir `data ThreeValues = ThreeValues a b c` (multiplicación de los tipos `a`, `b` y `c`) podemos no escribir nada y simplemente usar las tuplas que Haskell nos proporciona. E.g. `(1,2) :: (Int, Int)`, `(1,'♥',"sugar") :: (Int, Char, [Char])`, etcétera.

La solución al ejercicio 2 del tutorial [Algo sobre listas y todo sobre funciones]()

> 2. Write a function "average" that gets the average from a list of Doubles using "foldl".
     For the empty list, let the average be zero.
     Write a function "increaseAndSum" that helps you with the folding.

luce así:
```active haskell
data Tuple a b = Tuple a b
zeroes = Tuple 0 0

increaseAndSum (Tuple currentCount currentSum) x = Tuple (currentCount + 1) (currentSum + x)

average [] = 0
average ls = sum/count
  where Tuple count sum = foldl increaseAndSum zeroes ls

main = print (average (take 100 [1 ..]))
```
Pero usando ***tuplas*** pudo haber sido más corto:
```active haskell
{-hi-}increaseAndSum (currentCount, currentSum) x = (currentCount + 1, currentSum + x){-/hi-}

average [] = 0
average ls = sum/count
  where {-hi-}(count, sum){-/hi-} = foldl increaseAndSum {-hi-}(0,0){-/hi-} ls

main = print (average (take 100 [1 ..]))
```
Y ya sólo estamos entonces a un pequeño paso de que finalmente quede así (usándo una función anónima):
```active haskell
average [] = 0
average ls = sum/count
  where (count, sum) = foldl {-hi-}(\(cc,cs) x -> (cc+1,cs+x)){-/hi-} (0,0) ls

main = print (average (take 100 [1 ..]))
```
Si vas a usar funciones anónimas, asegúrate de que sean fáciles de entender; en general, [no abuses](http://vignette2.wikia.nocookie.net/adventuretimewithfinnandjake/images/3/32/S2e24_princess_bubblegum_eating_ice_cream.png/revision/latest?cb=20120506192530).

## Registros
Supongamos que quieres modelar una partida de [gato](https://en.wikipedia.org/wiki/Tic-tac-toe). Usando solo tipos algebráicos, muy probablemente modelarías el estado de esta manera:
```active haskell
data Mark = O | X | Empty
data Player = PO | PX
data Score = Score Int Int
data GameState = GameState [[Mark]] Player Player Score
```
Si hubiésemos usado registros, nos hubiera quedado de esta forma:
```active haskell
data Mark = O | X | Empty
data Player = PO | PX
data Score = {-hi-}Score { oVictories::Int, xVictories::Int }{-/hi-}
data GameState = {-hi-}GameState
                 {
                     boardState     :: [[Mark]]
                   , nextToMove     :: Player
                   , startedTheGame :: Player
                   , score          :: Score
                 }{-/hi-}
```
Así queda un poco más clara la intención de los tipos; ahora que podemos usar "etiquetas" para los valores, debe ser fácil entender que el primer `Player` (`nextToMove`) representa el siguiente jugador en tirar y el segundo `Player` (`startedTheGame`) indica quien tiró al inicio del juego (para saber a quien le tocará iniciar en el siguiente juego).

Pero en realidad no son etiquetas, son funciones; por ejemplo, la etiqueta `oVictories` en realidad es una función de tipo `Score -> Int`, o sea que recibe un score y regresa el número de victorias del jugador *círculo*. A continuación, usaremos `oVictories` como función:
```active haskell
data Score = Score { oVictories::Int, xVictories::Int }
someScore = Score { oVictories = 2, xVictories = 1 }
main = (print.{-hi-}oVictories{-/hi-}) someScore
```
Esto nos evita tener que hacer búsqueda de patrones. Si no hubiésemos usado registros, hubiésemos tenido que hacer algo como esto:
```active haskell
data Score = Score Int Int
someScore = Score 2 2

main = print oVictories
       where oVictories = case someScore of Score oVictories _ -> oVictories
```
Todo eso para hacer búsqueda de patrones sobre `someScore`... not cool.

Y siempre podemos construir un registro como si fuera un tipo de dato algebraico:
```active haskell
data Score = Score { oVictories::Int, xVictories::Int }
someScore = {-hi-}Score 2 1{-/hi-}
main = (print.oVictories) someScore
```
[C0Ool](https://31.media.tumblr.com/5e829faafb14759c51d20dae54525c29/tumblr_inline_nb7626MtH61rodyy6.gif).

### Actualización de registros

> Técnicamente, ningún dato se "actualiza" en Haskell, pues eso le quitaría la pureza; cuando nos referimos a "actualizar" un dato, nos referimos a duplicar un dato con cierta variación.

Supongamos que terminó un juego de gato y ganó `PO` y debemos crear un nuevo `Score` que refleje esto. Sin usar *registros*, se podría modelar así:
```active haskell
import Text.Printf

data Score = Score Int Int
data Player = PO | PX deriving Eq

updateScore (Score oVictories xVictories) winner
  | winner == PO = Score (oVictories + 1) xVictories
  | otherwise = Score oVictories (xVictories + 1)

scoreToStr (Score oVictories xVictories) =
  "O: " ++ (show oVictories) ++
  ", X: " ++ (show xVictories)
  
someScore = Score 2 2

main = do
  putStrLn $ "old score: " ++ (scoreToStr someScore)
  putStrLn $ "new score: " ++ (scoreToStr $ updateScore someScore PX)
```
Pero utilizando registros, obtenemos una sintaxis más conveniente:
```active haskell
data Player = PO | PX deriving Eq
data Score = Score { oVictories::Int, xVictories::Int }

updateScore {-hi-}s{-/hi-} winner
  | winner == PO = {-hi-}s { oVictories = oVictories s + 1 }{-/hi-} -- "updates" oVictories, xVictories remains the same
  | otherwise = {-hi-}s { xVictories = xVictories s + 1 }{-/hi-} -- "updates" xVictories, oVictories remains the same

scoreToStr s =
  "O: "   ++ (show.oVictories) s ++
  ", X: " ++ (show.xVictories) s

someScore = Score { oVictories = 2, xVictories = 1 }

main = do
  putStrLn $ "old score: " ++ (scoreToStr someScore)
  putStrLn $ "new score: " ++ (scoreToStr $ updateScore someScore PX)
  putStrLn $ "old score remains unchanded: " ++ (scoreToStr someScore) -- important!
```
En este ejemplo, `s { oVictories = oVictories s + 1 }` actualiza el dato `oVictories` de `s` y deja `xVictories` sin modificar. En `s { xVictories = xVictories s + 1}`, se actualiza el dato `xVictories` de `s` y deja `oVictories` sin modificar. Es importante recalcar que en realidad **no se actualiza** nada, sino que se crea un segundo objeto.

### Búsqueda de patrones sobre registros
Si utilizas registros para tus estructuras, debes saber que también puedes realizar búsqueda de patrones sobre estos:
```active haskell
data Mark = O | X | Empty
data Player = PO | PX deriving Eq
data Score = Score { oVictories::Int, xVictories::Int }
data GameState
  = GameState
  { boardState     :: [[Mark]]
  , nextToMove     :: Player
  , startedTheGame :: Player
  , score          :: Score
  }

updateScore s winner
  | winner == PO = s { oVictories = oVictories s + 1 }
  | winner == PX = s { xVictories = xVictories s + 1 }

scoreToStr s =
  "O: "   ++ (show.oVictories) s ++
  ", X: " ++ (show.xVictories) s

someGameState
  = GameState
  { boardState = [[Empty, Empty, Empty], [Empty, Empty, Empty], [Empty, Empty, Empty]]
  , nextToMove = PX
  , startedTheGame = PX
  , score = Score { oVictories = 1, xVictories = 0 }
  }

printScore {-hi-}(GameState { score = s }){-/hi-} = putStrLn.scoreToStr $ s

main = printScore someGameState
```

## Captura de argumentos
Cuando hacemos búsqueda de patrones sobre una estructura de datos, podemos acceder a sus valores internos, pero perdemos la capacidad de hacer referencia a la estructura completa.

Por ejemplo supongamos que queremos definir una función `toAdult` que dada la información de una persona, regresa `Just p` si `p` es mayor de 17 años y `Nothing` de lo contrario.

```active haskell
data Color = Red | Orange | Yellow | Green | Blue | Indigo | Violet | Pink deriving Show
data Person = P String Int Color deriving Show -- Name, age, favorite color

toAdult (P name age color)
  | age > 17 = Just $ P name age color
  | otherwise = Nothing

main = print $ map toAdult [P "Lay" 24 Blue, P "Jenny" 17 Pink, P "Bill" 59 Blue]
```
Y no está tan mal, pero con la **captura de argumentos**, recobramos la habilidad de hacer referencia a la estrucutra completa a la vez que podemos hacer búsqueda de patrones.

A continuación veremos el mismo ejemplo pero implementado usando captura de patrones tanto para tipos de datos algebraicos como para registros.

### Captura de argumentos sobre un tipo de dato algebraico
```active haskell
data Color = Red | Orange | Yellow | Green | Blue | Indigo | Violet | Pink deriving Show
data Person = P String Int Color deriving Show -- Name, age, favorite color

toAdult {-hi-}p@(P _ age _){-/hi-}
  | age > 17 = Just {-hi-}p{-/hi-}
  | otherwise = Nothing

main = print $ map toAdult [P "Lay" 24 Blue, P "Jenny" 17 Pink, P "Bill" 59 Blue]
```
### Captura de argumentos sobre un registro
```active haskell
data Color = Red | Orange | Yellow | Green | Blue | Indigo | Violet | Pink deriving Show
data Person = P { name::String, age::Int, favColor::Color } deriving Show

toAdult {-hi-}p@(P {age = a}){-/hi-}
  | a > 17 = Just {-hi-}p{-/hi-}
  | otherwise = Nothing

main = print $ map toAdult [P "Lay" 24 Blue, P "Jenny" 17 Pink, P "Bill" 59 Blue]
```

## Módulos
Modularizar el código nos permite

- tener más de un ***espacio de nombres*** (***namespaces***)
- agrupar el código por funcionalidad
- esconder información de un módulo a otro

En esta sección veremos brevemente como crear un programa usando tres módulos, cada uno en su propio archivo y como beneficiarnos de tener más de un *espacio de nombres*.

Empecemos directamente con un ejemplo, con lo que has aprendido hasta ahora, deberías de poder entender el significado del código con sólo leerlo.

```active haskell
{-# START_FILE Color.hs #-}
module Color where -- We declare a module named "Color"
data Color = Red | Orange | Yellow | Green | Blue | Indigo | Violet | Pink deriving Show
data RGB = RGB { red::Float, green::Float, blue::Float }

rgbToStr (RGB r g b) =
    "R:" ++ (show r) ++
  ", G:" ++ (show g) ++
  ", B:" ++ (show b)

colorToRGB c = RGB r g b
  where (r,g,b) = case c of
                  Red    -> (1, 0, 0)
                  Orange -> (1, 0.647, 0)
                  Yellow -> (1, 0.843, 0)
                  Green  -> (0, 1, 0)
                  Blue   -> (0, 0, 1)
                  Indigo -> (0.294, 0, 130)
                  Violet -> (0.933, 0.509, 0.933)
                  Pink   -> (1, 0.752, 0.796)

-- blends two colors using this formula: http://stackoverflow.com/a/29321264
blendColors (RGB r1 g1 b1) (RGB r2 g2 b2) t =
  RGB r g b
  where
    r = sqrt $ ((1-t)*r1)^2 + (t*r2)^2
    g = sqrt $ ((1-t)*g1)^2 + (t*g2)^2
    b = sqrt $ ((1-t)*b1)^2 + (t*b2)^2

{-# START_FILE Person.hs #-}
module Person where -- We declare a module named "Person"
import Color
data Person = P { name::String, age::Int, favColor::Color }

ana = P "Ana" 25 Red
bob = P "Bob" 25 Pink

blendFavColors p1 p2 t =
  blendColors (colorToRGB $ favColor p1) (colorToRGB $ favColor p2) t

{-# START_FILE Main.hs #-}
module Main where -- declares a module named "Main"
import Color      -- imports the Color module
import Person     -- imports the Person module

main =
  putStrLn $ "If we blend evenly ana's favorite color with bob's favorite color, we get "
             ++ (rgbToStr $ blendFavColors ana bob 0.5)
```

### Resolviendo name clashes
En el módulo `Color` está definida una función `blendColors` y en el módulo `Person` está definida una función `blendFavColors`; la función `blendFavColors` bien pudo haberse llamado también `blendColors`, pero eso hubiese causado un "*name clash*" en el espacio de nombres del módulo `Main` pues habrían dos elementos con el mismo nombre.

La ambigüedad creada por dos elementos con el mísmo nombre se puede resolver si antecedemos el uso de dichos elementos con el nombre del módulo al que pertenecen y un punto: `{-hi-}Module.{-/hi-}element`.

Entonces, sí podemos llamar ambas funciones `blendColors` y el ejemplo pasado quedaría así:
```active haskell
{-# START_FILE Color.hs #-}
module Color where -- We declare a module named "Color"
data Color = Red | Orange | Yellow | Green | Blue | Indigo | Violet | Pink deriving Show
data RGB = RGB { red::Float, green::Float, blue::Float }

rgbToStr (RGB r g b) =
    "R:" ++ (show r) ++
  ", G:" ++ (show g) ++
  ", B:" ++ (show b)

colorToRGB c = RGB r g b
  where (r,g,b) = case c of
                  Red    -> (1, 0, 0)
                  Orange -> (1, 0.647, 0)
                  Yellow -> (1, 0.843, 0)
                  Green  -> (0, 1, 0)
                  Blue   -> (0, 0, 1)
                  Indigo -> (0.294, 0, 130)
                  Violet -> (0.933, 0.509, 0.933)
                  Pink   -> (1, 0.752, 0.796)

-- blends to colors using this formula: http://stackoverflow.com/a/29321264
blendColors (RGB r1 g1 b1) (RGB r2 g2 b2) t =
  RGB r g b
  where
    r = sqrt $ ((1-t)*r1)^2 + (t*r2)^2
    g = sqrt $ ((1-t)*g1)^2 + (t*g2)^2
    b = sqrt $ ((1-t)*b1)^2 + (t*b2)^2

{-# START_FILE Person.hs #-}
module Person where -- We declare a module named "Person"
import Color
data Person = P { name::String, age::Int, favColor::Color }

ana = P "Ana" 25 Red
bob = P "Bob" 25 Pink

{-hi-}blendColors{-/hi-} p1 p2 t =
  {-hi-}Color.{-/hi-}blendColors (colorToRGB $ favColor p1) (colorToRGB $ favColor p2) t

{-# START_FILE Main.hs #-}
module Main where -- declares a module named "Main"
import Color      -- imports the Color module
import Person     -- imports the Person module

main =
  putStrLn $ "If we blend evenly ana's favorite color with bob's favorite color, we get "
             ++ (rgbToStr $ {-hi-}Person.{-/hi-}blendColors ana bob 0.5)
```

Si escribir el nombre completo del módulo es muy tedioso, se puede declarar un alias en la importación del módulo, por ejemplo:
```haskell
{-# START_FILE Main.hs #-}
module Main where
import Color
import Person {-hi-}as P{-/hi-}    -- imports the Person module

main =
  putStrLn $ "If we blend evenly ana's favorite color with bob's favorite color, we get "
             ++ (rgbToStr $ {-hi-}P.{-/hi-}blendColors ana bob 0.5)
```
Para una información más completa sobre la importación de módulos, visita [Modules - Haskell, The Wikibook](https://en.wikibooks.org/wiki/Haskell/Modules)
## Interpolación de cadenas (string interpolation)
¿Recuerdas este ejemplo?
```active haskell
data Color = Black | White | Gray deriving Show

introduction :: String -> (Int -> (Color -> String))
introduction name age color =
    "Hi, I'm " ++ name ++ ". "
    ++ "I'm " ++ (show age) ++ " years old "
    ++ "and my favorite color is " ++ (show color)

main = putStrLn $ introduction "Pluto" 3 Gray
```

Se puede evitar mezclar tantos `String`s con `++` y patrones utilizando [interpolación de cadenas](https://en.wikipedia.org/wiki/String_interpolation#C.23.NET). Existen varias formas, pero hay una que será familiar para muchos, [`Text.Printf`](http://hackage.haskell.org/package/base-4.2.0.1/docs/Text-Printf.html) pues está inspirada en [printf](https://es.wikipedia.org/wiki/Printf) del lenguaje de programación C.

Usando interpolación de cadenas, queda más limpio:
```active haskell
{-hi-}import Text.Printf{-/hi-}

data Color = Black | White | Gray deriving Show

introduction :: String -> (Int -> (Color -> String))
introduction name age color =
  {-hi-}printf{-/hi-} "Hi, I'm {-hi-}%s{-/hi-}. I'm {-hi-}%i{-/hi-} years old and my favorite color is {-hi-}%s{-/hi-}" name age (show color)

main = putStrLn $ introduction "Pluto" 3 Gray
```
En el formato declarado, indicamos que interpolaremos tres variables, la primera siendo un `String` (`%s`), la segunda siendo un `Int` (`%i`) y la tercera un `String` (`%s`) nuevamente.

**Pero ten cuidado**, pues el formato producido por `printf` no es muy fuertemente tipado y puede arrojar excepciones en tiempo de ejecución si es mal utilizado:

```active haskell
import Text.Print
format = printf "%i"           -- expects an int
main = putStrLn $ format ">:)" -- raises an exception {-hi-}in runtime{-/hi-} with an evil type error
```
Por esta razón, algunos sugieren que nos conformemos con `++`.

En la documentación de [`Text.Printf`](http://hackage.haskell.org/package/base-4.2.0.1/docs/Text-Printf.html) hay una tabla con todas las conversiones soportadas.

## Ejercicios

[Ejercicios](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-4/ejercicios)

[Soluciones](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-4/soluciones)

## Siguiente tutorial
Y eso es todo por este tutorial. Si has seguido todos los tutoriales y hecho todos los ejercicios, te mereces un postre [:)](http://mentalfloss.com/article/65394/how-do-you-punctuate-around-emoticons-and-emoji) Cuando quieras, puedes continuar con el [siguiente tutorial](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-5/tutorial).