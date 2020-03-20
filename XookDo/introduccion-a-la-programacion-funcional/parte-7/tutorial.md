# Mónadas

En este tutorial, nos enfocaremos en el estudio de la clase `Monad`; esta clase es especial en Haskell por dos razones:

- Existe azúcar sintáctico que nos facilita su uso, la notación `do`, que de hecho ya hemos utilizando sin una buena explicación.
- Se utiliza para interactuar con el mundo real, cosa que ya hemos estado haciendo cada vez que imprimimos en consola igual sin una buena explicación.

En este tutorial daremos esas buenas explicaciones.

# Mónadas
De la definición del funtor applicativo a la del mónada sólo hay una función adicional de diferencia, `join :: m (m a) -> m a`; su función es la de "eliminar" un contenedor cuando hay dos anidados.

Para la lista, la definición de `join` es simplemente `concat` (concatenación de listas):
```active haskell
join = concat

aListOfLists :: [[Int]]
aListOfLists = [[1,2],[3,4]]

joinedLists :: [Int]
joinedLists = join aListOfLists

main =
  do
    print aListOfLists
    print $ join joinedLists
```

La definición de `join` para `Maybe` sería esta:
```haskell
join (Just (Just x)) = Just x -- From two anidated containers to one
join (Just Nothing) = Nothing -- From two anidated containers to one
```

La definición completa de la clase mónada podría ser entonces:
```haskell
class Applicative m => Monad m where
  join  :: m (m a) -> m a

  (>>=) :: m a -> (a -> m b) -> m b
  m >>= f = join (fmap f m)

  (>>)  :: m a -> m b -> m b
  m >> n = m >>= \_ -> n
```
`>>=` es un operador llamado "bind" y podemos ver que su definición se deriva de `join` y de `fmap` por lo que es correcto afirmar que un mónada es un funtor aplicativo con la operación adicional de `join`.

Sin embargo, por razones históricas, la verdadera definición de la clase `Monad` es esta otra:
```haskell
class Applicative m => Monad m where
  return :: a -> m a
  return = pure -- pure comes with m being also an applicative functor. return and pure do exactly the same thing!

  (>>=) :: m a -> (a -> m b) -> m b

  (>>)  :: m a -> m b -> m b
  m >> n = m >>= \_ -> n
```
En esta definición no se nos pide implementar `join` y en su lugar, tenemos que implementar `>>=`. Teniendo `>>=`, podríamos derivar para todo mónada la definición de `join` pues así como `m >>= f = join (fmap f m)`, `join m = m >>= id` (`id` es la función identidad' `id x = x`). Dado `>>=`, igual podríamos derivar para todo mónada la definición de `fmap`, pues `fmap f x = x >>= (return.f)`.

## Leyes de los mónadas
Hay dos maneras de expresar las leyes de los mónadas:

- En términos de `>>=` y `return`. Al ser estas funciones parte de la definición de la clase `Monad`, me referiré a esta manera como "en términos simples".
- En términos de `>=>`. La función `>=>` (llamada "composición de Kleisli" o "pescado") está definida en términos de `>>=` y `return` por lo que esta manera de de expresar las leyes de los mónadas es teórica mente más compleja, pero visualmente e intuitivamente más sencilla de comprender.

En las secciones **Leyes de los mónadas en términos de (>=>)** y **Leyes de los mónadas en términos simples** veremos las leyes en términos de (`>=>`) y `>>= & return` respectivamente. Entender solo una sección es suficiente. En la sección **Relación entre >=>, >>= y return** comprobaremos que ambas maneras son equivalentes; está sección podría ser ignorada siempre y cuando se entienda alguna de las dos maneras de expresar las leyes de los mónadas.


### Leyes de los mónadas en términos de (>=>)
`(>=>)` es como composición de funciones, `(.)`, pero para funciones de este tipo: `a -> m b`, donde `m` es el contenedor de un mónada; el tipo de `(>=>)` es `(a -> m b) -> (b -> m c) -> (a -> m c)`. Usando `(>=>)` se pueden definir las 3 leyes de los mónadas de esta manera:

- **Identidad por la izquierda:**

`(return >=> f) = f`
Que significa que `return` a la izquierda de `>=>` forma la función identidad.

- **Identidad por la derecha:**

`(f >=> return) = f`
Que significa que `return` a la derecha de `>=>` forma la función identidad.

- **Asociatividad:**

`(f >=> g) >=> h = f >=> (g >=> h)`
Que simplemnete significa que el operador `>=>` es asociativo.

### Leyes de los mónadas en términos simples
A continuación formularemos las leyes de los mónadas en términos de `return` y `>>=`; es un poco más verbosa que la forma de expresar las leyes en términos de `>=>`, pero también es más directa.

- **Identidad por la izquierda:**
`return a >>= f = f a`

- **Identidad por la derecha:**
`m >>= return = m`

- **Asociatividad:**
`(m >>= f) >>= g = m >>= (\x -> f x >>= g)`

### Relación entre >=>, >>= y return

Para poder argumentar que basta con seguir las leyes basadas en `>=>`, es necesario comprobar su relación con la formulación de las leyes basadas en `>>=` y `return`.

Primero veamos la definición de `(>=>)`:
```haskell
(>=>) :: (a -> m b) -> (b -> m c) -> a -> m c
-- or as I find clearer that (>=>) is a kind of function composition:
(>=>) :: (a -> m b) -> (b -> m c) -> {-hi-}({-/hi-}a -> m c{-hi-}){-/hi-}
f >=> g = \x -> f x >>= g
```
Con esto queda claro que `(>=>)` está definida en términos de `>>=`. A continuación transformaremos cada una de las leyes en términos de `(>=>)` a términos de `>>=` y `return` hasta llegar a la formulación de las leyes en términos de `>>=` y `return`; así quedará demostrado que las dos maneras de expresar las leyes son equivalentes.

Iniciemos con la primer ley: `return >=> f = f`
```haskell
-- Substituting >=> by its definition:
-- before:
(return {-hi-}>=>{-/hi-} f) = f

-- after:
\x -> return x {-hi-}>>={-/hi-} f = f
-- and applying an x to both sides:
return x >>= f = f x -- which equals to the first rule in simple terms
```
Similar con la segunda ley:
```haskell
-- before:
(f {-hi-}>=>{-/hi-} return) x = f x

-- after:
\x -> f x {-hi-}>>={-/hi-} return = f
-- and applying an x to both sides:
f x >>= return = f x -- which equals to the second rule in simple terms
```

Y por último, la tercera ley:
```haskell
-- let's do first the left hand side of the law: `(f >=> g) >=> h`
-- substituting `(f >=> g)` by the definition of >=>: 
\x -> (f x >>= g) >=> h

-- then, substituting (\x -> f x >>= g) {-hi-}>=>{-/hi-} h by the definition of >=>:
\y -> \x -> (f x >>= g) y >>= h

-- which is obviously equal to simply:
\x -> (f x >>= g) >>= h

-- now the right hand side of the law: f >=> (g >=> h)
-- substituting `(g >=> h)` by the definition of >=>:
f >=> (\y -> g y >>= h)

-- then, substituting `f {-hi-}>=>{-/hi-} (\y -> g x >>= h)` by the definition of >=>:
\x -> f x >>= (\y -> g y >>= h)

-- now we can put result1 and result2 side by side in an equation:
\x -> (f x >>= g) >>= h   =   \x -> f x >>= (\y -> g y >>= h)

-- and applying an x to both sides:
(f x >>= g) >>= h   =   f x >>= (\y -> g y >>= h)
-- which equals to the third law in simple terms
```

## Maybe como mónada

`Maybe` además de ser miembro de la clase `Functor` y de la clase `Applicative`, también es miembro de la clase `Monad`. Veamos algunos ejemplos y luego los explicaremos con detalle.

```active haskell
import Control.Applicative
main =
  do
    print ((Just 1) >>= (\x -> Just (x + 1))) -- case 1.1
    print ((Just 1) >>= (return.(+ 1)))       -- case 1.2
    print ((Just 1) >>= (pure.(+ 1)))         -- case 1.3

    print (Nothing >>= return.(+ 1))          -- case 2
```

Los casos 1.1, 1.2 y 1.3 son equivalentes, así que sólo expandiremos el caso 1.1:
```haskell
(Just 1) >>= (\x -> Just (x + 1))
-- using this definition: m >>= f = join (fmap f m), it equals to:
join (fmap (\x -> Just (x + 1)) (Just 1))
-- by the definition of fmap for Maybe
join (Just (Just 2))
-- which by the definition of join, it equals to:
Just 2
```

Ahora el caso 2:
```active haskell
Nothing >>= return.(+ 1)
-- by the definition of >>= for Maybe
join (fmap (\x -> Just (x + 1)) Nothing)
-- by the definition of fmap for Maybe
join (Just Nothing)
-- by the definition of join for Maybe
Nothing
```

Con suerte, tu pensamiento crítico se esté preguntando: ¿Por qué `join (Just (Just x)) = Just x`? o ¿Por qué `join (Just (Nothing))` no es igual a `Just 0` en vez de `Nothing`?, pero la respuesta no es mas que: porque así es la definición del "contexto" de `Maybe`. Por otro lado, esta implementación de `Monad` es bastante útil, pues ataca el error de [mil millones de dólares](http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare); a continuación un ejemplo:

```active haskell
add m1 m2 =
  m1 >>= (\x1 -> m2 >>= (\x2 -> return (x1 + x2)))

main =
  do
    print $ add (Just 1) (Just 2)
    print $ add {-hi-}Nothing{-/hi-} (Just 2)             -- first value is "absent"
    print $ add (Just 1) {-hi-}Nothing{-/hi-}             -- second value is "absent"
    print $ add {-hi-}Nothing{-/hi-} {-hi-}Nothing{-/hi-} -- both values are "absent"
```

En otros lenguajes, la ausencia de un valor se trata con un `null` (o `nil`) y eso nos obliga a checar que cada argumento no sea `null`; en Haskell, todos esos errores se detectan en tiempo de compilación, pues si un valor puede no estar presente, se tiene que declarar de forma explícita rodeando su tipo de un `Maybe`.

## Notación do
Como ya habíamos mencionado, una de las razones por la que los mónadas son especiales en Haskell es porque ofrecen azúcar sintáctico para el operador `>>=`. Antes de explicarlo con detenimiento, veámoslo en acción:
```active haskell
add m1 m2 =
-- previously: m1 >>= (\x1 -> m2 >>= (\x2 -> return (x1 + x2)))
  do
    x1 <- m1
    x2 <- m2
    return $ x1 + x2

main =
  do
    print $ add (Just 1) (Just 2)
    print $ add Nothing (Just 2)
    print $ add (Just 1) Nothing
    print $ add Nothing Nothing
```

Después de estar usando la notación `do` por todos lados, ahora sí va una explicación detallada. Cada bloque `do` corresponde a esta expresión: `m1 >>= (\x1 -> (m2 >>= (\x2 -> (... (mn >>= (\xn -> exp))))))`, donde `x1` representa un valor extraído de su contexto `m1`, `x2` representa un valor extraído de su contexto `m2`, `xn` algún valor extraido de su contexto `mn` y `exp` una expresión que regresa algún valor dentro de un mónada.

Poniéndolos lado a lado:
```haskell
add m1 m2 =          |add m1 m2 = 
  do                 |
    x1 <- m1         |  m1 >>= (\x1 ->
    x2 <- m2         |    m2 >>= (\x2 ->
    return $ x1 + x2 |      return $ x1 + x2))
```
Así queda claro que aunque la notación `do` puede parecer programación imperativa, cada línea dentro de un bloque `do` corresponde a una parte de una sola expresión. 

El operador `>>` se utiliza para realizar los efectos secundarios de extraer un valor de su contexto sin importar realmente el valor extraido, de ahí que su definición sea esta: `m >> n = m >>= \{-hi-}_{-/hi-} -> n` donde el guión bajo significa que se ignora el valor extraido de `m`. Imprimir dos veces en consola es un caso de uso simple para este operador:
```active haskell
main = putStr "Hello" >> putStr ", world!"
```
El tipo de `putStr` es `:: String -> IO ()`, donde `IO` es un mónada y `()` es ***unit***, el único valor carente de información en Haskell. Al `putStr` producir un `IO ()` es obvio que no nos importa lo que contiene el contenedor de `IO`, sino sólo el efecto secundario de extraer `()` de `IO` el cual es imprimir en consola. Para el operador `>>` la notación `do` también ofrece azúcar sintáctico. Dado que `>>` ignora el valor extraido de su contenedor, no hay necesidad de capturarlo en un patrón, por lo que simplemente se omite `xn <- ` de `xn <- mn` quedando de esta manera:

```active haskell
main =
  do
    putStr "Hello"
    putStr ", world!"
```
Podríamos capturar los valores extraidos de `IO ()`, pero no podríamos hacer mucho con ellos:
```active haskell
main =
  do
    x1 <- putStr "Hello"
    x2 <- putStrLn ", world!"
    print x1
    print x2
```
Hubiésemos podido simplemente hacer:
```active haskell
main =
  do
    putStr "Hello"
    putStrLn ", world!"
    print ()
    print ()
```

## Listas como mónadas
Las listas también son mónadas en Haskell y su implementación de `join` es simplemente `concat`. Veámoslo en acción:
```active haskell
x = do
  x1 <- [1,2,3]
  x2 <- ["a","b","c"]
  return (show x1 ++ x2)

main = print x
```
Analicemos con detalle que sucede en `x`.
Primero, eliminemos el azúcar sintáctico de la notación `do`.
```active haskell
x =
  [1,2,3] >>= (\x1 ->
    ["a","b","c"] >>= (\x2 ->
      return (show x1 ++ x2)))

main = print x
```
Después separemos `x` en tres expresiones `e1`, `e2` y `e3` para mantener la claridad. `e1` representa la subexpresión más anidada en `x` y `e3` la subexpresión menos anidada.
```active haskell
e1 x1 x2 = return (show x1 ++ x2)
e2 x1 = ["a","b","c"] >>= e1 x1
e3 = [1,2,3] >>= e2


main = print e3
```
Y finalmente, sustituyamos el operador `>>=`. Utilizaré esta definición de bind: `m >>= f = (join.fmap f) m`, recordando que la implementación de `join` para las listas es simplemente `join = concat`.

```active haskell
join = concat

e1 x1 x2 = return (show x1 ++ x2)
e2 x1 = (join.fmap (e1 x1)) ["a","b","c"]
e3 = (join.fmap e2) [1,2,3]

main = print e3
```

O si es más facil de entender en una sola expresión:
```active haskell
join = concat

e1 x1 x2 = return (show x1 ++ x2)
e2 x1 = (join.fmap (e1 x1)) ["a","b","c"]
e3 = (join.fmap e2) [1,2,3]

main = print e3
```

## El mónada IO 

Finalmente llegamos al punto en que podemos explicar con detalle como funciona la interacción con el mundo real la ejecución de un programa de Haskell mediante el ***mónada de IO*** (*** IO monad).

> IO viene de I/O que a su vez viene del inglés "Input/Output". Input/Output hace referencia a la comunicación entre el hardware y el software. "Input" viene de la introducción de información a un sistema; "Output" viene de la acción de un sistema de externalizar información. Por ejemplo, un teclado es un periférico que sirve para introducir información, mientras que un monitor es un periférico que sirve para externalizar información a un usuario; pero en general, el I/O sucede entre sistemas que no involucran a humanos de por medio, como en las redes de comunicación.

Veamos algunas funciones básicas para interactuar con la consola y lo que nos dicen sus tipos:

- `putStr :: String -> IO ()`
Dado un `String`, esta función genera un mónada de IO que no contiene información útil, por que el tipo del mónada generado es `IO ()` donde `()` es el tipo ***unit***, un tipo carente de información, pues su único valor posible es `()`.

- `putStrLn :: String -> IO ()`
Como `putStr`, pero adem´ås de imprir el `String`, también imprime un salto de línea.

- `getLine :: IO String`
`getLine` lee un `String` de la consola. Esta función no recibe parámetros, por la que más que ser una función, es una expresión; su tipo es `IO String`, pues a diferencia de `putStr` y `putStrLn`, la ejecución de `getLine` sí contiene información, dicha información es el `String` leído de la consola.

- `readLn :: Read a => IO a`
El tipo de clase `Read` se puede implementar por todos aquellos tipos que se puedan formar a partir de un `String`. Por ejemplo, el tipo `Int` puede y es miembro de la clase `Read`, pues un `String` como `"123123"` se puede *parsear* a un entero `123123`.
Entonces, `readLn` lee un `Read` de la consola y ese es el valor que carga el mónada `IO a`.

Para leer un Int se puede hacer algo así:
```active haskell
plusOne :: Int -> Int
plusOne = (+ 1)

main =
  do
    x <- readLn
    print $ plusOne x
```
Aquí, `plusOne x` hace que la inferencia de tipos de Haskell infiera que el tipo de `readLn` es `IO Int`, pues `plusOne x` implica que `x` es un Int.

```active haskell
plusOne :: Float -> Float
plusOne = (+ 1)

main =
  do
    x <- readLn
    print $ plusOne x
```
En este caso se infiere que el tipo de `readLn` es `IO Float` pues `plusOne x` implica que `x` es un `Float`.

Sin azucar sintáctico, se podría escribir de esta manera:

```active haskell
plusOne :: Float -> Float
plusOne = (+ 1)

main = readLn >>= \x -> print $ plusOne x
```

Es importante recalcar que las expresiones de tipo `IO a` no son sentencias, no se va a realizar ninguna operación de I/O fuera de la función `main`. Por ejemplo:

```active haskell
anIO :: IO ()
anIO = print "this never happens"

main = putStrLn "Hi!"
```

Incluso aunque `anIO` fuera parte de una expresión que sí se encuentra dentro de main, no quiere decir que el I/O sucedería, pues Haskell es lazy y no evalua sus parámetros a no ser necesario:

```active haskell
anIO :: IO ()
anIO = print "this will not happen"

anotherIO :: IO ()
anotherIO = print "this will happen"

f a b True = a
f a b False = b

main = f anIO anotherIO False
```

## Excepciones

Aunque nuestro código sea correcto, cuando interactua con el mundo real, las cosas pueden salir mal: se perdió la conexión a internet, el usuario conectó un periférico dañado, el usuario introdujo datos inválidos a los parámetros de nuestro programa, etc.

Las funciones que interactuan con el mundo real son las que pueden provocar dichas excepciones; a estas funciones se les conoce como ***[inseguras](http://stackoverflow.com/posts/32804182/revisions)*** (***unsafe***).

`readLn` es una función insegura, pues aunque su tipo es `Read a => IO a`, puede no regresar un valor y arrojar una excepción. Por ejemplo, si espera leer un `Int` (`IO Int`) y escribimos en la consola un `String` que no se puede parsear a un `Int`, no queda de otra más que reportar la excepción y terminar la ejecución. Intenta escribir un `String` que no es un `Int`, por ejemplo, `12ab` o un `Float`, por ejemplo `1.0`:

```active haskell
main =
  do
    x <- readLn :: IO Int
    print x
```

Por eso es necesario proteger el código de producción de todas las posibles excepciones y usar funciones inseguras lo menos posible. Una manera de reescribir el ejemplo pasado para hacerlo seguro es usando la función `readMaybe`, definida en el paquete `Text.Read`. El tipo de `readMaybe` es `Read a => String -> Maybe a` y lo que hace es que dado un `String`, lo "parsea" a un `Just a` si todo salió bien o regresa `Nothing` si algo salió mal. Pero `readMaybe` no lee nada de la consola, por lo que primero se tiene que usar `getLine` para obtener el `String`.

```active haskell
import Text.Read
main =
  do
    x <- getLine
    case (readMaybe x) :: Maybe Int of
      Nothing -> print "not an Int"
      Just i  -> print $ "the introduced Int is: " ++ (show i)
```
Ahora intenta introducir un `Int` (e.g. `1324`) y algo que no se pueda parsear como un `Int` (e.g. `13y4j`) y verás que ningún caso genera una excepción haciendo nuestro código *seguro*.

¿Y si quisieramos definir una función que lee un `String`, lo intenta parsear a un `Int` y si falla lo vuelve a intentar hasta lograrlo? En `Haskell` no hay ciclos, pero hay recursión y es lo que utilizaremos para definir esta función:

```active haskell
import Text.Read

readAnIntOrRetry :: IO Int
readAnIntOrRetry =
  do
    print "Introduce an Int"
    x <- getLine
    case (readMaybe x) :: Maybe Int of
      Just i -> return i  -- wrap the Int in the IO monad
      Nothing ->
        do
          print "That wasn't an Int"
          readAnIntOrRetry  -- here happens the recursion

main =
  do
    x <- readAnIntOrRetry
    print $ "the introduced Int is: " ++ (show x)
```

Hay varias cosas que vale la pena resaltar de este ejemplo:

- `readAnIntOrRetry` tiene el tipo `IO Int`; una vez que se "entra" a un mónada, no hay salida y dado que `readAnIntOrRetry` requiere leer de la consola, necesita "entrar" al mónada de `IO`.
- Hay un `do` adentro de otro `do`. No tiene nada de especial, sólo es la primer vez que se ve en estos tutoriales. Si no usáramos el azucar sintáctico, se vería algo como: `... >>= {-hi-}(... >>= ...){-/hi-} >>= ...`, donde la parte resaltada podría ser un `do` anidado.
- `readAnIntOrRetry` utiliza funciones que retornan valores monádicos de distinto tipo, e.i. `getLine :: IO String` y `print :: IO ()`, esto es completamente válido, pues `>>=` nos permite cambiar el tipo contenido por el mónada (recordando su tipo: `m a -> (a -> m b) -> m b` podemos ver que podemos iniciar con un `m a` (e.g. `IO ()`) y terminar con un `m b` (e.g. `IO String`) (`IO () -> (() -> IO String) -> IO String`). Lo que no se puede hacer es cambiar el tipo del contenedor, por ejemplo, no se puede cambiar de `IO` a `Maybe` así: `IO String -> (String -> Maybe String) -> Maybe String`, o en código:

```active haskell
f :: Maybe String -> Maybe String
f maybeAString =
  do
    x <- getLine      -- :: {-hi-}IO{-/hi-} String
    y <- maybeAString -- :: {-hi-}Maybe{-/hi-} String
    return (x ++ y)   -- this would be unclear, return into a the Maybe monad or into the IO monad?

main = print $ f (Just "hi")
```
El error es bastante claro: **Couldn't match type IO with Maybe**. Para poder combinar mónadas se utilizan los transformadores de mónadas (monads transformers), pero ese es tema de otro tutorial.

## Ejercicios
[Ejercicios](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-7/ejercicios)

[Soluciones](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-7/soluciones)

## Siguiente tutorial
[Work in progress](http://i.dailymail.co.uk/i/pix/2012/12/04/article-2242634-16584CBA000005DC-112_634x501.jpg).