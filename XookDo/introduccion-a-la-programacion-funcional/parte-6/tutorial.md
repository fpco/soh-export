# Funtores, funtores aplicativos y monoides

En este tutorial veremos clases de tipos un poco más avanzadas. Comenzaremos con la clase `Functor`, continuaremos con la clase `Applicative`, después `Semigroup` y al final `monoid`. Estudiar las clases `Functor` y `Applicative` nos prepararán para el siguiente tutorial sobre la clase `Monad`. En Haskell, es común que la definición de una clase de tipo esté acompañada de "leyes" que se tienen que cumplir a implementar dichas clases. Las leyes de la clase `Functor` y `Applicative` no son las más sencillas, por lo que también estudiaremos en este tutorial las clases `Semigroup` y `Monoid`, cuyas leyes son bastante simples. También existe una relación entre los monoides y los mónadas comúnmente expresada con la frase: "Un mónada es sólo un monoide en la categoría de endofuntores", eso pertenece a la rama de teoría de categorías, lo cual es un tema muy aparte.


## Funtores (Functor)
La clase `Functor` puede ser implementada por todas aquellas estructuras de datos a las cuales se les pueda aplicar una función a todos sus elementos.

La especificación de la clase `Functor` es la siguiente:
```haskell
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```
`fmap` es la función que *mapea* una función a todos los elementos del tipo que implemente la clase. En el tipo de `fmap` (`(a -> b) -> f a -> f b `), se puede ver que el primer parámetro es una función `a -> b`, esa es la que se aplica a todos los elementos dentro de `f` y el segundo parámetro es el functor, `f a`.

### Leyes de los funtores
A veces no basta con dar una implementación para las funciones de una clase; a veces se tienen que seguir ciertas *leyes*. La clase `Functor` tiene dos leyes:

- Ley de identidad
  `fmap id = id`, es decir, si intentamos mapear `id` sobre los elementos de algún contenedor, nos debe de dar lo mismo a que si no hubiésemos hecho nada (sin efectos secundarios).
- `fmap (f.g) x = fmap f (fmap g x)`
En teoría, cada implementación de una clase debe de venir acompañada de demostraciones matemáticas que demuestren que se cumplen las leyes. En el último tutorial veremos como usar QuickCheck para demostrar con cierta confiabilidad que se cumplen las leyes; es decir, no son demostraciones sino simples pruebas con alta probabilidad de confiabilidad.


### Listas como funtores
Un ejemplo que ahora debería de ser fácil de entender son las listas; de hecho, la única razón por la que la función se llama `fmap` y no simplemente `map` es porque `map` ya está reservado para listas, pero hacen exactamente lo mismo.

Si la función `map` no existiera, podríamos implementar `Functor` para listas de esta manera:

```haskell
instance Functor [] where
  fmap _ [] = []
  fmap f (x:xs) = f x : fmap xs
```
Pero `map` sí existe y esta es precísamente su definición:
```haskell
map _ [] = []
map f (x:xs) = f x : fmap xs
```
Por lo que la implementación de `fmap` para listas es simplemente:
```haskell
instance Functor [] where
  fmap = map
```
Un ejemplo de su uso:
```active haskell
main =
  do
    print $ fmap (+ 1) []
    print $ fmap (+ 1) [1,2,3]
```
En el *prelude* de Haskell ya viene incluida la implementación de `Functor` para `[]`, por lo que uno no tiene que escribirla para usarla.

### Maybe como funtor
El tipo `Maybe` es otra instancia de `Functor`. En el *prelude* de Haskell también ya viene incluida la implementación de `Functor` para `Maybe`, por lo que uno no tienes que escribirla para usarla.

Su implementación es obvia:
```haskell
instance Functor Mabye where
  fmap _ Nothing = Nothing
  fmap f (Just x) = Just (f x)
```
Un ejemplo de su uso:
```active haskell
main =
  do
    print $ fmap (+ 1) Nothing
    print $ fmap (+ 1) (Just 0)
```
### Nuestra implementación de Functor para un árbol binario etiquetado
Si modelamos un árbol binario etiquetado así:
```haskell
data LBTree a = InternalNode a (LBTree a) (LBTree a) | Leaf (Maybe a)
```
Podriamos hacerlo miembro de la clase `Functor` de la siguiente manera:
```haskell
instance Functor LBTree where
fmap f (InternalNode x left right) = InternalNode (f x) (fmap f left) (fmap f right)
fmap f (Leaf Nothing) = Leaf Nothing
fmap f (Leaf (Just x)) = Leaf (Just f x)
```

Y para demostrar su funcionalidad, reutilizaremos [este ejemplo](https://en.wikipedia.org/wiki/Binary_tree#/media/File:Binary_tree.svg)
```active haskell
data LBTree a = InternalNode a (LBTree a) (LBTree a) | Leaf (Maybe a) deriving Show

instance Functor LBTree where
  fmap f (InternalNode x left right) = InternalNode (f x) (fmap f left) (fmap f right)
  fmap _ (Leaf Nothing) = Leaf Nothing
  fmap f (Leaf (Just x)) = Leaf (Just (f x))

example = InternalNode 2
            (InternalNode 7
              (Leaf (Just 2))
              (InternalNode 6
                (Leaf (Just 5))
                (Leaf (Just 11))))
            (InternalNode 5
              (Leaf Nothing)
              (InternalNode 9
                (Leaf (Just 4))
                (Leaf Nothing)))

main = print $ fmap (* 10) example
```

## Funtores aplicativos (Applicative Functor)
Los funtores aplicativos son parecidos a los funtores, pero con la diferencia de que el "contenedor" puede albergar funciones también y que el operador `<*>`, lo que sería `fmap`, pero para funtores, mapea una función que se encuentra en un contenedor; quizás quede más claro si comparamos los tipos de `fmap` y `<*>`:
`<*> :: {-h-}f{-/h-} (a -> b) -> f a -> f b`
`fmap ::  (a -> b) -> f a -> f b`

Dado que `Maybe` es una instancia de un funtor, además de poder hacer esto:
```haskell
fmap (+ 1) (Just 2) = Just 3
```
También podemos hacer esto:
```haskell
(<*>) (Just (+ 1)) (Just 2) = Just 3
-- Or in its more natural usage, as an infix operator:
Just (+ 1) <*> Just 2 = Just 3
```

Para poder introducir una función dentro de un contenedor, usamos la función `pure` que junto con `<*>` forman la clase `Applicative`.

Pudimos haber escrito el ejemplo pasado como:
```haskell
pure (+ 1) <*> Just 2 = Just 3
```

La definición completa de `Applicative` es:
```haskell
class Functor f => Applicative f where
  pure :: (a -> b) -> f (a -> b)

  <*> :: f (a -> b) -> f a -> f b

  <$> :: (a -> b) -> f a -> f b
  h <$> f = pure h <*> f
```
El operador `<$>` contiene una definición default que se deriva de `pure` y `<*>`, por lo que no es necesario implementarla, basta con implementar `pure` y `<*>`. La utilidad de `<$>` es la de ahorrarnos tener que llamar primero a `pure` antes de poder aplicarle un valor usando `<*>`a la función que deseamos meter dentro del contenedor. Por ejemplo:

```active haskell
import Control.Applicative  -- The Applicative class is defined in the Control.Applicative module
main =
  do
    print $ {-hi-}pure{-/hi-} (+ 1) {-hi-}<*>{-/hi-} Just 2
    print $ (+ 1) {-hi-}<$>{-/hi-} Just 2
```
### Leyes de los funtores aplicativos
Además de las leyes inherentes de ser un funtor, un funtor aplicativo debe de cumplir con estas leyes:

- Ley de identidad

  `pure id <*> x = x`, similar a la primer ley de los functores
- `fmap g x = pure g <*> x`

Esta otra ley relaciona a los functores con los functores aplicativos al poner a `fmap`, `pure` y a `<*>` en la misma ecuación:

- Homomorfismo

  El término homomorfismo proviene de teoría de categorías; un homomorfismo es una función que mapea de un objeto a otro con la misma estructura matemática. Esta es la ley:
  `pure f <*> pure x = pure (f x)`

- Intercambio

  En la expresión `u <*> pure y` no debe importar si primero se ejecuta `(<*>) u` o si primero se ejecuta `pure y`, la ley dice así: `u <*> pure y = pure (\f -> f y) <*> u`
- Composición

  Esta es la ley: `u <*> (v <*> w) = pure (.) <*> u <*> v <*> w`, pero no queda muy clara su intención. Yo la reescribiría de esta manera:
  `C f1 <*> (C f2 <*> C x) = pure ((f1.f2) x)` donde `C` es el contenedor. Así queda más claro que la intención de la ley es expresar que se podría primero realizar la composición entre `f1` y `f2`, después la aplicación de `x` y al final meter todo en un contenedor con `pure`, pues a pesar de que en Haskell no existen sentencias, solo expresiones, al momento de la verdad (al momento de la ejecución) sí existe cierto orden de ejecución, por lo que a veces es necesario expresiar que el orden de ejecución de las partes de una expresión **no** debería de importar.

### Maybe como funtor aplicativo
Ya hemos usado a `Maybe` como funtor y su implementación no debe resultar extraña:

```haskell
import Control.Applicative

instance Applicative Maybe where
  pure f = Just f
  Nothing <*> _ = Nothing
  _ <*> Nothing = Nothing
  Just f <*> Just x = Just (f x)
```
Y aquí un ejemplo de cada caso:
```active haskell
import Control.Applicative
import Data.Char

e1 :: Maybe String
e1 = Nothing <*> Just "hi"      

e2 :: Maybe String
e2 = Just (map toUpper) <*> Nothing

e3 :: Maybe String
e3 = Just (map toUpper) <*> Just "hi"

e4 :: Maybe String
e4 = pure (map toUpper) <*> Just "hi"

e5 :: Maybe String
e5 = map toUpper <$> Just "hi"

main =
  do
    print e1
    print e2
    print e3
    print e4
    print e5
```

### Listas como funtor aplicativo
`Maybe` nos permitió poner una función dentro de un contenedor, pero ¿cuál sería el comportamiento de aplicar una lista de funciones a una lista de valores? Primero veamos un ejemplo y después lo explicaremos presentando la definición de los funtores aplicativos para listas.
```active haskell
import Control.Applicative
import Data.Char
main =
  do
    print $ [(+ 1), (* 2)] <*> [3,4,5]
    print $ [Prelude.map toUpper, (\x -> x ++ ", world!")] <*> ["hello", "hi"]

```
Podemos ver que cada función en la lista de la izquierda se aplica a cada elemento de la lista de la derecha. Si ahora vemos la implementación de de `Applicative` para listas, podemos comprobarlo:

```haskell
import Control.Applicative

instance Applicative [] where
  pure x = [x]
  fs <*> xs = [f x | f <- fs, x <- xs] -- for each f in fs, for ecah x in xs
```

### Más allá de funciones de un solo parámetro
El tipo del operador `<*>` es `f (a -> b) -> f a -> f b`, pero eso no significa que sólo se puedan usar funciones de un sólo parámetro, pues la `b` podría sustituirse por otra función, digamos, `c -> d`. Por ejemplo:

```active haskell
import Control.Applicative

-- here we can see that f (a -> {-hi-}b{-/hi-}) gets substituted by Just (Int -> (Int -> Int)),
--   where f = Maybe, a = Int and {-hi-}b = (Int -> Int){-/hi-}
f1 :: Maybe (Int -> (Int -> Int))
f1 = pure (+)

f2 :: Maybe (Int -> Int)
f2 = f1 <*> Just 1

f3 :: Maybe Int
f3 = f2 <*> Just 2

main =
  do
    print $ f3
-- or all together
    print $ pure (+) <*> Just 1 <*> Just 2
-- and even simplier
    print $      (+) <$> Just 1 <*> Just 2
```

## Semigrupos
Los semigrupos son un concepto del algebra que se representa en Haskell con la clase `Semigroup`, definida en `Data.Semigroup`. Un semigrupo es un conjunto de valores cerrados por una operación asociativa, es decir, que dados dos elementos del semigrupo, la operación asociativa regresa otro elemento del mismo semigrupo. En Haskell, la operación asociativa de un semigrupo es `<>`; esta es la definición completa de un semigrupo:
```haskell
class Semigroup a where
  (<>) :: a -> a -> a

  sconcat :: NonEmpty a -> a
  sconcat (a :| as) = go a as where
    go b (c:cs) = b <> go c cs
    go b []     = b
```
`NonEmpty a` representa una lista no vacia: `NonEmpty v = x :| xss` donde `xss` es una listá común (posiblemente vacía). Entonces, `sconcat` es una función que dada una lista no vacía de `a`s, los combina en un solo valor mediante `<>`.

### Leyes de los semigrupos
La única regla que deben de cumplir los semigrupos es que **`<>` debe de ser asociativa**, por lo que los enteros junto con la resta, no pueden formar un semigrupo, pues no es lo mismo `(1-2)-3 = -4` que `1-(2-3) = 2`.

A continuación, una instancia de `Semigroup` que sí cumple con la ley de asociatividad para `<>`:
```active haskell
import Data.Semigroup
import Data.List.NonEmpty

instance Semigroup Int where
  (<>) = (+)

aNonEmptyList = (1 :| [2,3]) :: NonEmpty Int

main =
  do
    print $ (1::Int) <> 2
    print $ sconcat aNonEmptyList
    -- It is only necessary to specify the type of one number, the compiler will infer the rest
```

## Monoides
Los monoides son un concepto de algebra que se representa en Haskell con la clase `Monoid` definida en `Data.Monoid`:
```haskell
class Monoid a where
  mempty  :: a
  mappend :: a -> a -> a
 
  mconcat :: [a] -> a
  mconcat = foldr mappend mempty
```
`mappend` es un operador binario que dado dos `a`s regresa otra `a`, es decir, un monoide `a` está cerrado bajo la operación `mappend`.

### Leyes de los monoides
Además de ofrecer implementaciones para `mempty` y `mappend`, se deben de cumplir dos leyes:

- Identidad de `mempty` para el lado izquierdo y derecho:
  - `mappend mempty x = x`
  - `mappend x mempty = x`
- Asociatividad de `mappend`. `mappend (mappend x y) z = mappend x (mappend y z)`

Por ejemplo, `Int` y la función `+` forman un monoide, donde `Int` es `a`, `+` es `mappend` y el `0` es `mempty`. Se podría comprobar facilmente que `(+) 0 = id` por lo que `0` cumple con los requisitos para ser el `mempty` de `Int` y `(+)`. A continuación la [demostración por inducción](https://es.wikipedia.org/wiki/Inducci%C3%B3n_matem%C3%A1tica) (probablemente la instancia mas senilla de la inducción matemática):

```haskell
-- base case:
(+) 0 0 = 0           -- e1
-- for any n:
(+) 0 n = n           -- e2, since 0 + n = n
-- now we need to show that "(+) 0 (n + 1) = n + 1" holds:
(+) 0 (n + 1) = n + 1 -- e3, to be shown
  -- we know that (n + 1) = (0 + n) + 1 and that (0 + n) = (+) 0 n,
  -- so (+) 0 (n + 1) equals to:
  ((+) 0 n) + 1
  -- replacing ((+) 0 n) by the right hand side of e2:
  n + 1 -- which is what we wanted to proof of e3
```

La implementación de `Monoid` para `Int`, `0` y `(+)` es:
```haskell
import Data.Monoid

instance Monoid Int where
  mempty = 0
  mappend = (+)
```
¿Qué crees que haga `mconcat`?
```active haskell
import Data.Monoid

instance Monoid Int where
  mempty = 0
  mappend = (+)

listOfInts = [1,2,3] :: [Int] -- we need to specify that those Nums are Ints.
main = print $ mconcat listOfInts
```
Así es, intuitivamente se podría decir que los junta a todos en un solo valor; formalmente se dice que `mconcat` hace, pues, lo que dice su definición: `foldr mappend mempty`.

### Multiples instancias de la misma clase para el mismo tipo
`Int` también se podría hacer una instancia de `Monoid` usando `(*)` como `mappend` y `1` como `mempty`:
```active haskell
import Data.Monoid

instance Monoid Int where
  mempty = 1
  mappend = (*)

listOfInts = [1,2,3] :: [Int] -- we need to specify that those Nums are Ints.
main = print $ mconcat listOfInts
```

Pero, ¿qué pasa si queremos ambas instancias de `Monoid` para `Int`? una con `mappend = (+)` y otra con `mappend = (*)`. No se puede, intentémoslo:

```active haskell
import Data.Monoid

instance Monoid Int where
  mempty = 0
  mappend = (+)
  
instance Monoid Int where
  mempty = 0
  mappend = (-)

listOfInts = [1,2,3] :: [Int]
main = print $ mconcat listOfInts
               -- ummm, do you mean "mconcat = foldr {-hi-}(+){-/hi-} 0 listOfInts"
               -- or "mconcat = foldr {-hi-}(-){-/hi-} 0 listOfInts"
```
El compilador dice algo de "instancias duplicadas de la clase Monoid para Int". Afortunadamente existe una solución, utilizar `newtype`.

`newtype` nos permite crear un tipo nuevo (no un simple sinónimo de tipo como con `type`); podemos entonces crear dos tipos nuevos a partir de `Int` y hacer cada uno una instancia de `Monoid`, uno con `mappend` a base de `(+)` y otro con `mappend` a base de `(*)`:

```active haskell
import Data.Monoid

newtype IntMP = IntMP Int  -- for Monoid Int with mappend based on (+)
newtype IntMT = IntMT Int  -- for Monoid Int with mappend based on (*)

instance Show IntMP where
  show (IntMP x) = show x

instance Show IntMT where
  show (IntMT x) = show x

instance Monoid IntMP where
  mempty = IntMP 0
  IntMP x `mappend` IntMP y = IntMP (x + y)
  
instance Monoid IntMT where
  mempty = IntMT 0
  IntMT x `mappend` IntMT y = IntMT (x - y)

intsMPs = [IntMP 1, IntMP 2, IntMP 3]
intsMTs = [IntMT 1, IntMT 2, IntMT 3]

main =
  do
    print $ mconcat intsMPs
    print $ mconcat intsMTs
```

De hecho ya existe algo similar a `IntMP` y que no sólo es para `Int`s sino para toda la clase `Num`; se llaman `Sum` y funciona así

```active haskell
import Data.Monoid

intsAsSum = [0,1,2] :: [Sum Int]
floatsAsSum = [0,1,2] :: [Sum Float]
doublesAsSum = [0,1,2] :: [Sum Double]

main =
  do
    print $ mconcat intsAsSum
    print $ mconcat floatsAsSum
    print $ mconcat doublesAsSum
```

También existe `Product`:
```active haskell
import Data.Monoid

intsAsProduct = [0,1,2] :: [Product Int]
floatsAsProduct = [0,1,2] :: [Product Float]
doublesAsProduct = [0,1,2] :: [Product Double]

main =
  do
    print $ mconcat intsAsProduct
    print $ mconcat floatsAsSum
    print $ mconcat doublesAsSum
```

### Listas como instancias de Monoid
Una vez sabiendo que las listas pueden ser instancias de `Monoid`, no debe de ser muy difícil imaginar que valor será `mempty` y cual `mconcat`. ¿Qué función recibe dos listas y produce una lista? `(++)`, entonces `(++)` puede ser `mconcat`. Si `(++)` es `mconcat`, ¿cuál valor podría ser `mempty` tal que `(++) mempty xs = xs` para todo `xs`? La lista vacía (`[]`), obviamente.

Esta es la instancia de `Monoid` para las listas:
```haskell
instance Monoid [] where
  mempty = []
  mconcat = (++)
```

Ahora podemos crear expresiones que son compatibles con cualquier monoide:
```active haskell
import Data.Monoid

f m1 m2 m3 m4 = (mappend m1 m2, mappend m3 m4)

main =
  do
    print $ f (1::Sum Int) (2::Sum Int) (3::Sum Int) (4::Sum Int)
    print $ f [1] [2] [3,4] [5]
```

## Ejercicios
[Ejercicios](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-6/ejercicios)

[Soluciones](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-6/soluciones)

## Siguiente tutorial
Mónadas. [Work in progress](http://imgc.allpostersimages.com/images/P-473-488-90/61/6169/X3SG100Z/posters/man-waving-from-empire-state-building-construction-site.jpg)