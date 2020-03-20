# Clases de Tipos

Ya hemos visto con buen detalle las funciones y los tipos en Haskell. En este tutorial veremos como expresar que para cierto tipo, existen ciertas funciones usando ***clases de tipos*** (***type classes***).

## Motivación
Retomemos el último ejemplo del tutorial pasado:

```active haskell
data Human = FullName String String | Nickname String
simpleSalute (FullName firstName _) = "Hi, it's me, " ++ firstName ++ "!"
simpleSalute (Nickname nickname)    = "Hi, it's me, " ++ nickname ++ "!"

billGates = FullName "William" "Gates"
me        = Nickname "Lay"

main = do
  putStrLn (simpleSalute billGates)
  putStrLn (simpleSalute me)
```
Supongamos que también queremos modelar a los [*pokemones*](http://assets6.pokemon.com/assets/cms2-es-es/img/video-games/video-games/pokemon_battle_trozei/pokemon_battle_trozei_visit_site_169.jpg); los *pokemones*, según recuerdo, [sólo pueden decir el nombre de su especie](https://www.youtube.com/watch?v=GyJPzwM__v4) así que asumiremos que un pokemón saluda diciendo el nombre de su especie:

```active haskell
data Human   = FullName String String | Nickname String
data Pokemon = PokemonKind String

simpleSalute (FullName firstName _) = "Hi, it's me, " ++ firstName ++ "!"
simpleSalute (Nickname nickname)    = "Hi, it's me, " ++ nickname ++ "!"
simpleSalute (Pokemon pokemonKind) = pokemonKind

billGates = FullName "William" "Gates"
me        = Nickname "Lay"
pikachu   = Pokemon "Pikachu"

main = do
   putStrLn (simpleSalute billGates)
   putStrLn (simpleSalute me)
   putStrLn (simpleSalute pikachu)
```
Hasta ahora todo va mas o menos bien, pero no sabemos aún como implementar `simpleSalute` para *pokemones*. Si ejecutamos el programa nos dará un error diciendo que `Human` y `Pokemon` no hacen match; esto se debe a que `Human` y `Pokemon` son tipos completamente distintos y `simpleSalute` no puede tener simultáneamente los tipos `Human -> String` y `Pokemon -> String`.

Una alternativa es hacer `Human` y `Pokemon` parte de un tipo superior, `ThingThatSalutes`:
```active haskell
data Human = FullName String String | Nickname String
data Pokemon = PokemonKind String
data ThingThatSalutes = Human Human | Pokemon Pokemon

simpleSalute :: ThingThatSalutes -> String
simpleSalute (Human (FullName firstName _)) = "Hi, it's me, " ++ firstName ++ "!"
simpleSalute (Human (Nickname nickname))    = "Hi, it's me, " ++ nickname ++ "!"
simpleSalute (Pokemon (PokemonKind pokemonKind)) = pokemonKind ++ "!"

billGates = FullName "William" "Gates"
me        = Nickname "Lay"
pikachu   = PokemonKind "Pikachu"

main = do
   putStrLn (simpleSalute (Human billGates))
   putStrLn (simpleSalute (Human me))
   putStrLn (simpleSalute (Pokemon pikachu))
```
Y funciona, pero en mi opinión hay mucho "wrapping" (`(simpleSalute (Human billGates))`, `(simpleSalute (Pokemon pikachu))`) y el nombre del tipo `ThingThatSalutes` parece indicar que ya estamos creando abstracciones no muy buenas. Por fortuna, hay una alternativa, las ***clases de tipos*** (***type classes***).

## Clases de tipos (Type classes)
Una *clase de tipo* es un mecanismo del sistema de tipos el cual nos permite definir clases de tipos. Una clase de tipo consiste en un nombre para la clase y las funciones que deben estar definidas para el tipo que quiera ser miembro de dicha clase. A continuación, definiremos una *clase de tipo*, `Salutable`, para resolver de manera más "elegante" el último ejemplo.

```active haskell
data Human = FullName String String | Nickname String
data Pokemon = PokemonKind String

-- Salutable's type class definition
{-hi-}class Salutable a where{-/hi-}
  simpleSalute :: a -> String
  
-- Human's implementation of the Salutable type class
{-hi-}instance Salutable Human where{-/hi-}
  simpleSalute (FullName firstName _) = "Hi, it's me, " ++ firstName ++ "!"
  simpleSalute (Nickname nickname)    = "Hi, it's me, " ++ nickname ++ "!"

-- Pokemon's implementation of the Salutable type class
{-hi-}instance Salutable Pokemon where{-/hi-}
  simpleSalute (PokemonKind p) = p ++ "!"

{-hi-}introduction x y ={-/hi-}
  do
    putStrLn (simpleSalute x)
    putStrLn (simpleSalute y)

billGates = FullName "William" "Gates"
me        = Nickname "Lay"
pikachu   = PokemonKind "Pikachu"

main = do
   introduction billGates me
   introduction billGates pikachu
```

Ahora que podemos hacer que un humano y un pokemon saluden, definamos una función `introduction` que recibe dos instancias de `Salutable` y regrese el saludo de cada uno. Para esto, necesitaremos usar las restricciones de tipo.

## Restricciones de clase (class constraints)
### Restricciones sobre los parámetros de una función
Si checamos el tipo de `introduction` en [GHCi](https://en.wikibooks.org/wiki/Haskell/Using_GHCi_effectively):
![introduction's type](https://www.fpcomplete.com/media/7dd59027-cdc9-4b0d-9765-6c3b34722043.png)
, podemos ver que el tipo de `introduction` inferido por el compilador es `introduction :: (Salutable a, Salutable a1) => a -> a1 -> IO ()`; lo cual nos lleva a este tema, ***restricciones de clase*** (***class constraints***).

`introduction :: (Salutable a, Salutable a1) => a -> a1 -> IO ()` significa que cualquier tipo que sustituya a la *variable de tipo* `a` debe de ser miembro de la clase `Salutable`; lo mismo para la variable de tipo `a1`. No es lo mismo a escribir `introduction :: (Salutable a) => a -> a -> IO ()`, pues eso forzaría que ambos parámetros de `introduction` sean del mismo tipo (sea cual sea), lo cual no es la intención.

A `introduction` no le podemos pasar argumentos miembros de cualquier tipo; por ejemplo, no le podemos pasar dos `String`s:
```active haskell
class Salutable a where
  simpleSalute :: a -> String

introduction x y =
  do
    putStrLn (simpleSalute x)
    putStrLn (simpleSalute y)

main = do
   {-hi-}introduction "Not a Salutable" "I'm a String"{-/hi-}
```
pues `String` no es miembro de la *clase de tipo* `Salutable`. A pesar de que nosotros no tuvimos que especificar el tipo de `introduction`, el compilador pudo inferir que sus argumentos pueden ser miembros de cualquier tipo siempre y cuando dichos tipos sean miembros de la *clase de tipo* `Salutable`. La inferencia es posible dado el uso de `simpleSalute` sobre los argumentos `x` y `y` en la función `introduction`; para que `simpleSalute x` y `simpleSalute y` hagan sentido, `x` y `y` deben de ser miembros de la clase `Salutable`.

Además de las restricciones de tipos sobre los parámetros de las funciones, existen otras formas de restricciones de tipos. A continuación veremos el resto de estas.

### Restricciones de clase en definiciones de clases, implementaciones de clases y definiciones de constructores de tipos
Ahora que el concepto de restricciones de clase ha quedado claro, veremos la sintaxis que nos permitirá establecer restricciones en otras construcciones además de las definiciones de funciones.

#### Sintaxis básica de Haskell
##### Definiciones de clases de tipos
```haskell
class (SomeExistingClass a) => ClassBeingDefined a1 where
someFunction :: a1 -> a -> a2
```
##### Implementaciones de clases
```haskell
instance ClassBeingDefined TypeImplementingTheClass where
  someFunction (SomeExistingClass a1) => ClassBeingDefined -> SomeExistingClass -> a2
  someFunction memberOfTypeImplementingTheClass memberOfSomeExistingClass = ...
```
##### Definiciones de constructores de tipos
Algunos llaman a esto "herencia".
```haskell
data (SomeExistingClass a) => SomeTypeConstructor a = SomeDataConstructor a
```
## Ejemplo
A continuación, un ejemplo de todos los casos de restricciones de tipos (tomado de [aquí](https://en.wikibooks.org/wiki/Haskell/Classes_and_types#A_concerted_example)). El ejemplo fue modificado para sólo utilizar los conceptos vistos hasta ahora.

```active haskell
-- Type synonyms
type PointName = String
type PointX = Int
type PointY = Int
type DeltaInX = Int
type DeltaInY = Int

-- Type constructors
data Position      = Position PointX PointY
data PositionDelta = PositionDelta DeltaInX DeltaInY
data NamedPoint    = NamedPoint PointName PointX PointY

-- Location, in two dimensions.
class Located a where
  getPosition :: a -> Position

class (Located a) => Movable a where
  setPosition :: a -> Position -> a

instance Located NamedPoint where
  getPosition (NamedPoint pointName x y) = Position x y

instance Movable NamedPoint where
  setPosition (NamedPoint pointName _ _) (Position x y)
    =
      NamedPoint pointName x y

-- Moves a value of a Movable type by the specified displacement.
-- This works for any movable, including NamedPoint.
move :: (Movable a) => a -> PositionDelta -> a
move p (PositionDelta dx dy) =
  setPosition p newPosition
  where
    Position x y = getPosition p
    newPosition = Position (x + dx) (y + dy)

showNamedPoint (NamedPoint pointName x y) =
  pointName ++ " is at (" ++ (show x) ++ ", " ++ (show y) ++ ")"

main =
  do
    putStrLn (showNamedPoint p)
    putStrLn (showNamedPoint p')

  where
    p = (NamedPoint "The point" 1 1)
    delta = PositionDelta 1 2
    p' = move p delta
```
Nuevamente, este ejemplo es sólo para fines pedagógicos. Se debe de tener mucho cuidado de no abusar de las *clases de tipos*. A continuación, algunos casos de éxito para las *clases de tipos*.

## Clases de tipos básicas del Prelude
### Eq

[`Eq`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Eq) define los operadores [`==`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#v:-61--61-) y [`/=`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#v:-47--61-) que se utilizan para comparar datos.

```active haskell
main =
  do
    putStrLn (show (1 == 2))
    putStrLn (show (1 /= 2))
```
Además de `Int`, otras instancias de `Eq` son:

- [`Float`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Float)
- [`Char`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Char)
- [`Number`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Number)
- [`Bool`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Bool)

### Ord
Los tipos capaces de pertenecer a la clase [`Ord`](https://hackage.haskell.org/package/base-4.8.1.0/docs/Data-Ord.html) aquellos sobre los cuales puede existin un [orden total](https://es.wikipedia.org/wiki/Orden_total). La clase [`Ord`](https://hackage.haskell.org/package/base-4.8.1.0/docs/Data-Ord.html) hereda de la clase [`Eq`]
```haskell
class Eq a => Ord a where
...
```
y define las siguientes funciones y operaciones:
```haskell
compare :: a -> a -> Ordering
(<) :: a -> a -> Bool
(<=) :: a -> a -> Bool
(>) :: a -> a -> Bool
(>=) :: a -> a -> Bool
max :: a -> a -> a
min :: a -> a -> a
```
donde [Ordering](http://hackage.haskell.org/package/base-4.8.1.0/docs/Prelude.html#t:Ordering) es una enumeración:
```haskell
data Ordering =
    LT -- "less than"
  | GT -- "greater than"
  | EQ -- "equals"
```
Algunos tipos miembros de la clase [`Ord`](https://hackage.haskell.org/package/base-4.8.1.0/docs/Data-Ord.html) son:
- [`Int`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Int)
- [`Integer`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Integer)
- [`Float`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Float)
- [`Char`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Char)
- [`Number`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Number)
- [`Bool`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Bool)

### Show
[`Show`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Show) lo hemos usado en todos los tutoriales; se utiliza para obtener una representación de una estructura de datos en un `String`. La única función que define es `show`.
Algunas instancias de `Show` son:

- [`Int`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Int)
- [`Float`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Float)
- [`Char`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Char)
- [`Number`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Number)
- [`Bool`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Bool)

```haskell
boolToStr :: Bool -> String
boolToStr bool = show bool
```

### Num
[`Num`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Num) define las siguientes funciones y operaciones:
```haskell
(+) :: a -> a -> a
(-) :: a -> a -> a
(*) :: a -> a -> a
negate :: a -> a
abs :: a -> a
signum :: a -> a -- regresa el signo de un Num (+1 o -1)
fromInteger :: Integer -> a
```
Algunas de sus implementaciones son:

- [`Int`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Int)
- [`Integer`](http://haddock.stackage.org/lts-3.4/base-4.8.1.0/Prelude.html#t:Integer)

## Ejercicios

[Ejercicios](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-5/ejercicios)

[Soluciones](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-5/soluciones)

## Siguiente tutorial
[Siguiente tutorial](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-6/tutorial) 
