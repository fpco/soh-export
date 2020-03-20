# Algo sobre listas y todo sobre funciones

En el tutorial pasado vimos algunos puntos importantes sobre las funciones de Haskell:

- Sólo reciben un valor y producen un valor
- Su firma tiene la forma `functionName :: T1 -> T2 -> ... -> Tn -> R` que es "azúcar sintáctico" ("syntactic sugar") para `functionName :: T1 -> (T2 -> (... -> (Tn -> R)))`. De esto surge el fenómeno llamado "aplicación parcial" ("partial application"), que no es más que aplicar un argumento a una función para que pase de ser `functionName :: T1 -> (T2 -> (... -> (Tn -> R)))` a `functionName :: T2 -> (... -> (Tn -> R))`.

En este tutorial iremos más allá y exploraremos los conceptos restantes sobre las funciones en Haskell, los cuales no quedan muchos en realidad. Al final de este tutorial, conocerás todo lo que necesitas saber sobre las funciones de Haskell, pero primero, exploraremos el concepto de listas en Haskell.

# Listas
Ya habrás notado que las listas se expresan en Haskell utilizando `[` y `]` para denotar el inicio y el fin de la lista y `,` para separar los elementos de la lista siendo definida. Pero la verdad es que esto no es más que azúcar sintáctico, pues la lista podría definirse con la siguiente estructura:

```haskell
List a = Cons a (List a) | Nil
```
Algunos ejemplos de listas que podriamos representar de esta manera:

- `[]              = Nil`
- `[1]             = Cons 1 Nil`
- `[1, 2, 3]       = Cons 1 (Cons 2 (Cons 3 Nil))`
- `[[1], [2], [3]] = Cons (Cons 1 Nil) (Cons (Cons 2 Nil) (Cons (Cons 3 Nil) Nil))`
- `[[1], [2, 3]]     = Cons (Cons 1 Nil) (Cons (Cons 2 (Cons 3 Nil)) Nil)`

Pero la verdad es que **no hay razón de lidiar con esto, el azúcar sintáctico de Haskell es suficente**. Lo único importante es no olvidar que cuando veamos `[]` o `[x]` o `[x, ...]` no es más que eso, azúcar sintáctico para una estructura de datos que se podría definir con los tipos algebraicos que ya conocemos.

## Anexar y concatenar (append and concatenate) (`:` and `++`)
Para añadir un elemento al inicio de una lista, existe el operador infijo `(:) :: a -> [a] -> [a]`. E.g. `1:[2,3] = [1,2,3]`.
Para concatenar listas, existe el operador `++`. E.g. `[1]++[2,3] = [1,2,3]`. Dado que `String` es un sinónimo de `[Char]`, también se utiliza el operador `++` para concatenar strings. E.g. `"Hello," ++ " World!" = "Hello, World!".

## Búsqueda de patrones sobre el azúcar sintáctico de las listas
No se podría usar el azúcar sintáctico para las listas si no hubiera también un método para hacer búsqueda de patrones sobre este. A continuación veremos ejemplificado en una función `f` todos los casos de búsqueda de patrones sobre una lista.
```haskell
f :: [a] -> String
f [] = "The list is empty (n = 0)"
f [x] = "The list contains exactly one element (n = 1)"
f [x1, x2] = "The list contains exactly two elements (n = 2)"
-- etc.
f (x1:xs) = "The list contains at least one element. xs could be [] (n >= 1)"
f (x1:x2:xs) = "The list contains at least two elements. xs could be [] (n >= 2)"
-- etc.
```
Pero esta definición tiene varios problemas:

- `[x]` no dejaría que `(x1:xs)` haga match en el caso de una lista con un elemento simplemente porque `[x]` aparece primero en la definición de `f`; se dice que los patrones se "empalman".
- Lo mismo sucede con `[x1, x2]` y `(x1:x2:xs)`; `[x1, x2]` no dejaría que `(x1:x2:xs)` haga match en el caso de una lista con dos elementos simplemente porque `[x1, x2]` aparece primero en la definición de `f`.
- Además, todas las listas que hagan match con `(x1:xs)` hacen match con `(x1:x2:xs)`, por lo que `(x1:xs)` inutiliza a `(x1:x2:xs)`. En general, una vez que vemos `:xs`, ya estamos capturando cualquier lista, sea vacia (`Nil`) o no (`Cons _ _`).

Así que para ver dichos patrones realmente en acción, lo haremos en dos funciones, `f1` y `f2`, para evitar empalmes.

```active haskell
f1 :: [a] -> String
f1 [] = "The list is empty (n = 0)"
f1 [x] = "The list contains exactly one element (n = 1)"
f1 [x1, x2] = "The list contains exactly two elements (n = 2)"
f1 (x1:x2:xs) = "The list contains at least two elements (n >= 2)"

f2 :: [a] -> String
f2 [] = "The list is empty (n = 0)"
f2 [x] = "The list contains exactly one element (n = 1)"
f2 (x1:x2:x3:xs) = "The list contains at least three elements (n >= 3)"
f2 _ = "The list contains 2 elements (n = 2)" -- by discard

main = do
  putStrLn ("f1 [] = "        ++ (f1 []))
  putStrLn ("f1 [1] = "       ++ (f1 [1]))
  putStrLn ("f1 [1,2] = "     ++ (f1 [1,2]))
  putStrLn ("f1 [1,2,3] = "   ++ (f1 [1,2,3]))
  putStrLn ("f1 [1,2,3,4] = " ++ (f1 [1,2,3,4]))

  putStrLn ("f2 [] = "        ++ (f2 []))
  putStrLn ("f2 [1] = "       ++ (f2 [1]))
  putStrLn ("f2 [1,2] = "     ++ (f2 [1,2]))
  putStrLn ("f2 [1,2,3] = "   ++ (f2 [1,2,3]))
  putStrLn ("f2 [1,2,3,4] = " ++ (f2 [1,2,3,4]))
```
# El resto sobre funciones
Ahora que ya conocemos más sobre las listas en Haskell, será más fácil hablar sobre los siguientes temas.

## Composición de funciones
La ***composición de funciones*** (***function composition***) es una manera de expresar que el resultado de una función `f` es la entrada de otra función `g`. En Haskell tiene esta sintaxis: `g.f`.
A continuación, un ejemplo.

```active haskell
f s = "Hello, " ++ s
g s = s ++ "!"
gf  = g.f

main =
  do
    putStrLn (g (f "World"))
    putStrLn (gf "World")
```
Para poder componer una función con otra, es importante que el tipo de salida de la primer función, sea igual al tipo de entrada de la segunda.
Otro ejemplo.
```active haskell

increaseOne :: Int -> Int
increaseOne n = n + 1

intToStr :: Int -> String
intToStr i = show i

g :: String -> String
g s = s ++ "!"

f :: Int -> String
f = g.intToStr.increaseOne

main =
  do
    putStrLn (f 9)
    putStrLn (g (intToStr (increaseOne 9)))
```
En este caso, estamos componiendo *tres* funciones; nuevamente, la composicióne es posible por que se respetó la regla de que el tipo de salida de una función debe ser igual al tipo de entrada de la siguiente:

```haskell
increaseOne :: Int -> Int
intToStr    ::        Int -> String
g           ::               String -> Int
```

Para demostrar un caso en el que la composición no se logra por no respetar los tipos de las funciones, cambiaremos el orden de composición del ejemplo pasado por este (`g.increaseOne.intToStr`):
```haskell
intToStr    :: Int -> String
increaseOne ::        Int    -> Int            -- Type mismatch! String != Int
g           ::                  String -> Int  -- Type mismatch! Int    != String

```
Demostración del error al compilar por no respetar los tipos:
```active haskell

increaseOne :: Int -> Int
increaseOne n = n + 1

intToStr :: Int -> String
intToStr i = show i

g :: String -> String
g s = s ++ "!"

f :: Int -> String -- not really (type error)
f = {-hi-}g.increaseOne.intToStr{-/hi-}

main =
  do
    putStrLn (f 9)
```

## Funciones anónimas
Ponerle nombre a los objetos es uno de los métodos de abstracción mencionados en el primer tutorial, recordaremos que su función es poder hacer referencia a dicho objeto en otras partes. Cuando una función es simple (no necesita un nombre que nos recuerdo su funcionamiento) y sólo se va a utilizar una vez, podemos evitarnos usar un nombre y podemos expreserla directamente mediante una ***funcion anónima*** (***anonymous function***). Su sintaxis es sencilla; una función anónima de **n** parámetros tiene la siguiente forma `\p1 p2 ... pn -> haskellExpression`. En el siguiente tema, veremos ejemplos donde las funciones anónimas son de utilidad.

## Funciones de orden superior
Una función de orden superior es aquella que:

- Produce una función como su salida
- Alguno de sus parámetros es una función

### Funciones que producen funciones como su salida
En Haskell, todas las funciones que toman más de un parámetro forman parte de este grupo. Como ya habíamos dicho, una función de **n** parámetros que produce un valor de tipo **R** tiene esta signatura:
`:: P1 -> (P2 -> (... -> R))`; es sólo por el azúcar sintáctico que podemos escribir su firma de esta manera: `:: P1 -> P2 -> ... -> R`. Esto nos permite 2 cosas:

- ***Aplicación parcial*** (***partial application***)
- ***Eta reduction***

#### Aplicación parcial (partial application)
La aplicación parcial consiste en aplicar sólo los primeros **k** parámetros de **n** que tome una función, donde ***k < n***. Es decir, no aplicar todos los parámetros. Ejemplo:
```active haskell
data Color = Black | White | Gray

colorToStr c = case c of
               Black -> "Black"
               White -> "White"
               Gray  -> "Gray"

introduction :: String -> (Int -> (Color -> String))
introduction name age color =
    "Hi, I'm " ++ name ++ ". "
    ++ "I'm " ++ (show age) ++ " years old "
    ++ "and my favorite color is " ++ (colorToStr color)

introduction1 :: Int -> (Color -> String)
introduction1 = introduction "Pluto"

introduction2 :: Color -> String
introduction2 = introduction1 3

introduction3 = introduction2 Gray

main = putStrLn introduction3
```
Podemos ver como pudimos hacer *aplicación parcial* de sus parámetros uno por uno; cada vez que aplicabamos un parámetro, el tipo se reducía:
```haskel
introduction  :: String -> (Int -> (Color -> String))
introduction1 ::            Int -> (Color -> String)  -- after applying a  String
introduction2 ::                    Color -> String   -- after applying an Int
introduction3 ::                             String   -- after applying a  Color
```

La *aplicación parcial* es especialmente útil para reutilizar alguna de las funciones intermedias:
```active haskell
data Color = Black | White | Gray

colorToStr c = case c of
               Black -> "Black"
               White -> "White"
               Gray  -> "Gray"
               
introduction :: String -> (Int -> (Color -> String))
introduction name age color =
    "Hi, I'm " ++ name ++ ". "
    ++ "I'm " ++ (show age) ++ " years old "
    ++ "and my favorite color is " ++ (colorToStr color)

introduction1 :: Int -> (Color -> String)
introduction1 = introduction "Pluto"

introduction2 :: Color -> String
introduction2 = introduction1 3

-- reuse introduction2:
introduction31 = {-hi-}introduction2{-/hi-} Gray
introduction32 = {-hi-}introduction2{-/hi-} White

main =
  do
    putStrLn introduction31
    putStrLn introduction32
```


#### Eta reduction
Omitir un parámetro en la definición de una función se llama **eta reduction**; esto nos permite escribir código con menos ruido al no tener que estar escribiendo los parametros ni del lado izquierdo ni del lado derecho de una función. Por ejemplo, `plusOne n = (+) 1 n` y `plusOne = (+) 1` son equivalentes.

Ahora, otro caso más desarrollado.
Supongamos que tenemos una función `lengthIs` que recibe una lista (`[a]`) y un entero (`Int`) y produce un booleano (`Bool`) que indique si el tamaño de la lista es igual al entero recibido:

```active haskell
lengthIs :: [a] -> Int -> Bool
lengthIs l n = length l == n

main = putStrLn (show (lengthIs [1, 2, 3] 3))
```
Lo cual es igual a (usando `==` como función en vez de como operador infijo):
```active haskell
lengthIs :: [a] -> Int -> Bool
{-hi-}lengthIs l n = (==) (length l) n{-/hi-}

main = putStrLn (show (lengthIs [1, 2, 3] 3))
```
Ahora podemos realizar **eta reduction** y eliminar el último parámetro de `lengthIs`.
```active haskell
lengthIs :: [a] -> Int -> Bool
{-hi-}lengthIs l = (==) (length l){-/hi-}

main = putStrLn (show (lengthIs [1, 2, 3] 3))
```
Lo cual es igual a (usando composición de funciones):
```active haskell
lengthIs :: [a] -> Int -> Bool
{-hi-}lengthIs = (==) . length{-/hi-}

main = putStrLn (show (lengthIs [1, 2, 3] 3))
```
Nótese como al aplicar la transformación *composición de funciones*, nuevamente desaparece un parámetro. Con esto, ya tenemos una definición muy sucinta de `lengthIs`; el nombre de `lengthIs` junto con su tipo nos da una idea bastante clara de lo que hace: `lengthIs :: [a] -> Int -> Bool`.

> Es muy importante recordar que *eta reduction* se aplica siempre al último parámetro de una función, el cual es el primer argumento en ser aplicado. Se debe tener esto en mente al decidir el órden de los parámetros de una función

### Funciones que al menos uno de sus parámetros es una función
Poder explicar este tipo de funciones es la razón principal por la que introdujimos "algo sobre listas", pues las funciones de orden superior de este tipo son particularmente útil para trabajar con datos recursivos, como lo son las listas.

A continuación veremos los conceptos de ***map*** y ***fold*** para listas.

> En el tutorial siguiente (Clases de tipos), veremos que *fold* y una versión más general de `map` (`fmap`) aplican no sólo para listas, sino para toda una clase de tipos.

#### map
`map` se utiliza para aplicar una función a todos los elementos de una lista; el resultado de `map f [x, y, ..., z]` sería `[f x, f y, ..., f z]`. La definición de map es:

```haskell
map :: (a -> b) -> [a] -> [b]
map f [] = []
map f (x:xs) = f x : fmap f xs
```
Y algunas ejemplificaciones de sus usos:
```active haskell
import GHC.Unicode
main = do
  putStrLn (show (map (+ 1) [1,-2,3]))
  putStrLn (show (map (> 0) [1,-2,3]))
  putStrLn (map toUpper "hello, world!") -- toUpper converts a char to its upper case form
```
> Recuerda que un `String` es una lista de `Char`s (`[Char]`), por lo que podemos utilizar `map` para `String`s.

#### fold
La operación *fold* se refiere a obtener un valor resumen dado un conjunto de valores. Para listas existen dos versiones de *fold*: `foldl` y `foldr`. Para obtener el valor resumen, el usuario de `foldl` o `foldr` debe de proporcionar cual es el valor inicial y una función que los vaya acumulando. Por ejemplo:
```active haskell
main = putStrLn (show (foldl (+) 0 [1,2,3]))
```
Aquí, le indicamos a `foldl` que utilizaremos a la suma (`(+)`) para "acumular" los valores y que el `0` será nuestro valor inicial.

La definición de `foldl` para listas es la siguiente:
```haskell
foldl :: (b -> a -> b) -> b -> [a] -> b
foldl _ acum [] = acum
foldl f acum (x:xs) = foldl f (f acum x) xs
```
Entonces, se puede ver que `main = putStrLn (show (foldl (+) 0 [1,2,3]))` se ejecuta como `(((0 + 1) + 2) + 3)`.

`foldr` es como `foldl` pero invertida tanto la asociatividad como el orden de los operandos, entonces, `main = putStrLn (show (foldr (+) 0 [1,2,3]))` se ejecutaría como `(0 + (1 + (2 + 3)))`. En este caso, el resultado es nuevamente `6`, pero no siempre el resultado es el mismo, depende de que función utilizemos para "doblar" los valores; por ejemplo, la división (`(/)`) no es comutativa, por lo tanto, usar `foldl (/)` es diferente a `foldr (/)`.

```active haskell
main =
  do
    putStrLn (show (foldl (/) 1 [2,3,4])) -- ((1/2)/3)/4
    putStrLn (show (foldr (/) 1 [2,3,4])) -- 2/(3/(4/1))
    
    --  ______ foldl ______  |  ______ foldr ______
    --   1     / 2  = 0.50   |    4 / 1    = 4
    --   0.50  / 3  = 0.16…  |    3 / 4    = .75
    --   0.16… / 4  = 0.416… |    2 / 0.75 = 2.66…
```
Un ejemplo más de `foldr`, esta vez aprovechando que el tipo del valor de resumen no tiene que ser el mismo que el de los valores de la lista.
```active haskell
andGTZero = (&&).(> 0)
main =
  do
    putStrLn (show (foldr andGTZero True [1,2,4]))
    putStrLn (show (foldr andGTZero True [1,2,-4]))
```
Aunque la lista que "doblamos" es de enteros (`[Int]`), el tipo del resumen es `Bool`.


Aun cuando la función que utilizemos sea commutativa (y por lo tanto no importa si se utiliza asociatividad por la izquierda o por la derecha), se debe tomar en cuenta la eficiencia al decidir usar `foldl` o `foldr`. Como ejemplo, tomemos la operación de concatenación (`(++)`); No es lo mismo `[1]++[2,3,4,5]` que `[1,2,3,4]++[5]` en cuanto al tiempo que toma calcularse, pues para concatenar una lista, se requiere recorrer toda la primer lista hasta encontrar donde se conectará con la segunda. 


```active haskell
import Criterion.Measurement
-- show
import Data.Functor
import Data.List.Split

manyLists = chunksOf 1 (take 10000 [1,2 ..])

f1  = length (fold{-hi-}l{-/hi-} (++) [0] manyLists)
f2  = length (fold{-hi-}r{-/hi-} (++) [0] manyLists)

-- Some trickery not shown for measuring time using Criterion.Measurement
-- /show
main =
  do
    initializeTime
    t0  <- getTime
    putStrLn ("f1 = " ++ show f1)
    f1r <- return.show $ f1
    t1  <- getTime
    putStrLn ("f2 = " ++ show f2)
    t2  <- getTime

    putStrLn ("f1 took ~" ++ (secs (t1 - t0)))
    putStrLn ("f2 took ~" ++ (secs (t2 - t1)))
```

## Guía para escribir funciones

- Una función debe de producir una salida válida para toda entrada [1](http://math.stackexchange.com/questions/1429972/why-dont-we-have-an-analogous-of-rangecodomain-for-domain/1429979#comment2913299_1429972).
- Encontrar el balance adecuado entre eficiencia e inteligibilidad. Es importante considerar la etapa del producto; ¿es un prototipo? ¿es crítico no cometer errores? ¿cuál es el tiempo de vida estimado de la función?
- El orden de los parámetros importa. El orden de los parámetros determina como se hará la *aplicación parcial* y la composición.

# Ejercicios

[Ejercicios](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-3/ejercicios)

[Soluciones](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-3/soluciones)

# Siguiente parte
[Cuarta parte](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-4/tutorial)

