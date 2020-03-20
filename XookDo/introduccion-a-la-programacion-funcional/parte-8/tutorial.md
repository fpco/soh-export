# Pruebas automáticamente aleatorias con QuickCheck

La calidad es importante en cualquier producto o servicio y eso incluye al software. Muy probablemente, la prueba de calidad más importante en el desarrollo de software es si el código que escribimos realmente hace lo que queríamos. El método más estricto son las comprobaciones formales; con sistemas como [Coq](https://coq.inria.fr/) se puede autogenerar código Haskell a partir de la comprobación. Sin embargo, este método es muy avanzado, por lo que para código no crítico, se utilizan métodos más laxos como las pruebas aleatorias automáticas como se puede hacer con QuickCheck.

Para este tutorial, traduciré el documento en el que se introdujo originalmente QuickCheck (tomado de [aquí](http://www.eecs.northwestern.edu/~robby/courses/395-495-2009-fall/quick.pdf)) con la ventaja de que, a diferencia del documento, el código de ejemplo será ejecutable en el mismo tutorial presionando el botón de *play*; sin embargo, para varios casos he notado que la respuesta no se despliega correctamente y por lo tanto recomiendo copiar y pegar el código en un archivo, compilarlo y ejecturalo si se desea comprobar el resultado por uno mismo. El artículo original fue escrito por [Koen Claessen](http://www.cse.chalmers.se/~koen/) y [John Hughes](http://www.cse.chalmers.se/~rjmh/), ambos de la [Universidad Tecnológica de Chalmers](http://www.chalmers.se/en/Pages/default.aspx).

Existen implementaciones de QuickCheck para:

- [Haskell](https://wiki.haskell.org/Introduction_to_QuickCheck1)
- [F# - FsCheck](https://github.com/fscheck/FsCheck)
- [Scala - ScalaCheck](https://www.scalacheck.org)
- [C++](https://github.com/grogers0/CppQuickCheck)
- [JavaScript](https://bitbucket.org/darrint/qc.js/)
- [Ruby](https://github.com/mcandre/rubycheck)
- [Java](http://www.jcheck.org)
- [Prolog](http://www.swi-prolog.org/pack/list?p=quickcheck)
- [Swift](https://github.com/typelift/SwiftCheck)
- [etc](https://en.wikipedia.org/wiki/QuickCheck)

Para un caso de uso original, checa este análisis del problema [Estaciones de gasolina](https://www.fpcomplete.com/user/XookDo/algoritmos-funcionales/estaciones-de-gas) en donde se prueba la solución usando QuickCheck.

## QuickCheck: Una herramienta ligera para probar aleatoriamente programas escritos en Haskell

### Abstracto

QuickCheck es una herramienta que ayuda al programador de Haskell a formular y probar propiedades de las funciones que componen un programa. Las propiedades se expresan como funciones de Haskell, y pueden ser probadas automáticamente con entradas aleatorias y también es posible definir generadores de valores a la medida. Presentamos un número de casos de estudio en los que la herramienta se utilizó con éxito y también señalamos algunas dificultades que evitar. Las pruebas aleatorias son especialmente adecuadas para programas funcionales porque las propiedades se pueden expresar con bastante granularidad. Cuando una función es construida de componentes probados por separado, entonces las pruebas aleatorias serán suficientes para obtener una buena cobertura de la definición que esté a prueba.

### 1. Introducción

Las pruebas son por mucho el método más común usado para asegurar la calidad del software. También es un trabajo laborioso, llevándose hasta el 50% del costo del desarrollo de software. A pesar de la evidencia anecdótica de que los programas funcionales requieren de alguna manera menos pruebas (‘una vez que los tipos cuadran, usualmente funciona’), en la práctica es una gran parte del desarrollo en programación funcional.
El costo de realizar las pruebas motiva los esfuerzos en automatizarlas, completamente o parcialmente. Las herramientas de pruebas automáticas le facilitan al programador completar la realización de pruebas en menor tiempo o probar con mayor rigurosidad con el mismo tiempo disponible y hacen fácil repetir las pruebas después de cada modificación al programa. En este escrito, describiremos una herramienta, QuickCheck, que hemos desarrollado para probar programas escritos en Haskell.

Los programas funcionales son bastante adecuados para ser probados automáticamente. Es generalmente aceptado que las funciones puras son mucho más fáciles de probar que las que tienen efectos secundarios, porque uno no tiene que preocuparse por el estado antes y después de su ejecución. En un lenguaje imperativo, aun si todo el programa son funciones puras, los procedimientos de los cuales están construidas usualmente no lo son. Entonces, grandes unidades de código deben de ser probadas a la vez. En un lenguaje funcional, las funciones puras abundan (en Haskell, solo las computaciones en el mónada de IO son difíciles de probar, pues son las únicas impuras), y por lo tanto las pruebas se pueden hacer a un nivel bastante granular.

Una herramienta debe de poder determinar si una prueba pasó o falló; el humano probador debe de proveer un un criterio probable automáticamente. Hemos diseñado un simple lenguaje “domain-specific” de especificaciones probables que el probador usaría para definir propiedades esperadas de las funciones puestas a prueba. QuickCheck checa entonces que las propiedades se mantiene para un largo número de casos. Este lenguaje de especificaciones está embebido en Haskell usando su sistema de clases de tipos. Las propiedades son escritas normalmente en el mismo módulo que las funciones que prueban, donde estas sirven también como documentación probable del comportamiento del código.

Una herramienta para realizar pruebas también debe de ser capaz de generar casos de prueba automáticamente. Hemos elegido el método más simple, pruebas aleatorias, el cual compite es la práctica de manera sorprendentemente favorable con los métodos sistemáticos. Aunque, es insignificante hablar de pruebas aleatorias sin discutir la distribución de los datos a probar. Las pruebas aleatorias son mas efectivas cuando la distribución de las pruebas siguen las los casos reales, pero cuando se prueban unidades del sistema reutilizables en vez de todo el sistema, esto es simplemente imposible, dado la distribución de los datos usados realmente se desconocen para todos los posibles casos en los que la función es reutilizada. Una distribución uniforme es usualmente usada en su lugar, pero para la información obtenida de conjuntos infinitos, esto igual es insignificante. Hemos elegido poner la distribución sobre el control del humano que realiza las pruebas al definir un *lenguaje de generación de datos de prueba” (igualmente embebido en Haskell), y una manera de observar la distribución de los casos de prueba. Al programar un generador adecuado, quien realiza la prueba puede, no solo controlar la distribución de los casos de prueba, pero también asegurarse de que estos satisfacen arbitrariamente variantes complejas.

Un importante objetivo de diseño era que QuickCheck debería de ser *ligero*. Nuestra implementación consiste de un simple módulo de al rededor de 300 líneas. QuickCheck soporta todo el lenguaje de Haskell y sus extensiones. No depende de un sistema en particular de Haskell.
Es notoriamente difícil decir que tan efectivo es un método de pruebas en detectar errores. Aun así, hemos usado QuickCheck en una variedad de aplicaciones, yendo desde pequeños experimentos hasta sistemas reales, y hemos encontrado que funciona bien en la práctica. Hemos reportado algunas de estas experiencias en este documento, y hacemos notar que dificultades evitar.

### 2. Definiendo propiedades

#### 2.1 Un ejemplo sencillo

Como un primer ejemplo, tomamos la función standard *reverse* que invierte el orden de los elementos de una lista. Esa función satisface estas leyes útiles:
```haskell
         reverse [x] = [x]
    reverse (xs++ys) = reverse (ys++reverse) xs
reverse (reverse xs) = xs
```
Nótese que estas leyes se mantienen solo para valores finitos y totales (no para listas infinitas). En este documento, al menos que se diga específicamente lo contrario, siempre cuantificaremos sobre valores finitos completamente definidos. Esto es para hacer mas probable que las propiedades sean computables.

Para poder usar estas leyes usando QuickCheck, las representamos como funciones de Haskell. Entonces definimos:

```haskell
prop_RevUnit x =
  reverse [x] == [x]

prop_RevApp xs ys =
  reverse (xs++ys) == reverse ys ++ reverse xs

prop_RevRev xs =
  reverse (reverse xs) == xs
```

Ahora, si estas funciones regresasen `True` para cada posible argumento, entonces las propiedades se mantendrían. Las ponemos a prueba:

```active haskell
import Test.QuickCheck

prop_RevUnit x =
  reverse [x] == [x]

prop_RevApp xs ys =
  reverse (xs++ys) == reverse ys++reverse xs

prop_RevRev xs =
  reverse (reverse xs) == xs
  
main =
  do
    quickCheck (prop_RevUnit :: Int -> Bool)
    quickCheck (prop_RevApp :: [Int] -> [Int] -> Bool)
    quickCheck (prop_RevRev :: [Int] -> Bool)
```
> Se puede ver como esté metodo de expresar leyes nos podría servir cuando necesitamos crear una instancia una clase que tiene leyes, como `Semigroup`, `Monoid`, `Monad`, etc.

Y vemos que no se reportan fallas. Si hubiésemos definido mal alguna propiedad y esta no se mantuviera, podríamos ver como se reporta un error, como a continuación:

```active haskell
import Test.QuickCheck

prop_RevUnit x =
  reverse [x] == [x]

prop_RevApp xs ys =
  reverse (xs++ys) == {-hi-}reverse xs++reverse ys{-/hi-} -- wrong!

prop_RevRev xs =
  reverse (reverse xs) == xs
  
main =
  do
    quickCheck (prop_RevUnit :: Int -> Bool)
    quickCheck (prop_RevApp :: [Int] -> [Int] -> Bool)
    quickCheck (prop_RevRev :: [Int] -> Bool)
```
> Se puede ver que QuickCheck detecta el error y ofrece un contraejemplo que nos demuestra que efectivamente algo está mal.

A pesar de que las propiedades `prop_RevUnit`, `prop_RevApp` y `prop_RevRev` son polimórficas, es decir no importa el tipo contenido en la lista (`Int`, `Char`, etc.), Haskell necesita saber el tipo de la lista en compilación y no puede deducirlo sin una pista; en este caso le especificamos explícitamente cual es el tipo de la lista. Y de hecho esto es bueno, pues no todas las propiedades son polimórficas y hay veces que es distinta la semantica de una función polimórfica según el monomorfismo en particular a probar.

#### 2.2 Funciones
También somos capaces de formular propiedades que cuantifican sobre las funciones. Para probar que la composicicón de funciones es asociativa, necesitaremos definir el operador de igualdad extendida `(====)`.
```active haskell
{-# LANGUAGE FlexibleInstances #-}
import Test.QuickCheck

(f ==== g) x = f x == g x
prop_CompAssoc :: (Int -> Int) -> (Int -> Int) -> (Int -> Int) -> Int -> Bool
prop_CompAssoc f g h = (f.(g.h)) ==== ((f.g).h)

instance Show (Int -> Int) where
  show f = "<<function>>"

main = quickCheck prop_CompAssoc
```
La única dificultad que los tipos de las funciones causan, es que si un contraejemplo es encontrado, se imprimiría como `<<function>>` en la consola. Para verlo, intentemos probar que la composición de funciones es comutativa:
```active haskell
{-# LANGUAGE FlexibleInstances #-}
import Test.QuickCheck

(f ==== g) x = f x == g x
prop_CompComm :: (Int -> Int) -> (Int -> Int) -> Int -> Bool
prop_CompComm f g = (f.g) ==== (g.f)

instance Show (Int -> Int) where
  show f = "<<function>>"

main = quickCheck prop_CompComm
```
En este ejemplo, podemos ver que la composición de funciones no es comutativa, pues QuickCheck reporta un error, pero no podemos ver el contraejemplo.

#### 2.3 Leyes condicionales
Las leyes que son simples ecuaciones son representadas convenientemente por funciones booleanas, como lo hemos visto, pero en general muchas leyes solo se cumplen bajo ciertas condiciones. QuickCheck nos proporciona un combinador de implicación para representar dichas leyes condicionales. Por ejemplo, la ley `x <= y ==> max x y == y` puede ser representada por por esta definición:
```haskell
prop_MaxLe :: Int -> Int -> Property
prop_MaxLe x y = x <= y ==> max x y == y
```
De la misma manera, la función de inserción de una lista ordenada satisface la ley:
```active haskell
import Test.QuickCheck
import Data.List

isSorted :: (Ord a) => [a] -> Bool
isSorted []                     = True
isSorted (x:[])                 = True
isSorted (x:y:zs) | x <= y      = isSorted $ y:zs
isSorted _                      = False

prop_Insert :: Int -> [Int] -> Property
prop_Insert x xs =
  isSorted xs ==> isSorted (insert x xs)

main = quickCheck prop_Insert
```
Podemos ver que para esta propiedad, el tipo no es `Bool` sino `Property`. Esto es porque la semántica es distinta; si el lado izquierdo de la implicación no es `True`, se descarta ese caso y se intenta uno nuevo. ¡Pero! si muchos casos se descartan, puede que se realicen menos pruebas arrojando un mensaje similar a `Arguments exhausted after 64 tests.` Es responsabilidad del programador decidir si es suficiente o si debe de renfinar las pruebas (usando generadores especializados, por ejemplo).

#### 2.4 Monitoreando los datos de prueba
Quizás parezca que hemos probado la función de inserción lo suficiente para establecer su credibilidad, pero debemos de ser cuidadosos. Si modificamos la propiedad `prop_Insert` de esta manera:

```active haskell
import Test.QuickCheck
import Data.List

isSorted :: (Ord a) => [a] -> Bool
isSorted []                     = True
isSorted (x:[])                 = True
isSorted (x:y:zs) | x <= y      = isSorted $ y:zs
isSorted _                      = False

prop_Insert :: Int -> [Int] -> Property
prop_Insert x xs =
  isSorted xs ==>
    {-hi-}classify (null xs) "trivial"{-/hi-} $
      isSorted (insert x xs)

main = quickCheck prop_Insert
```
Vemos que las listas vacias (`null xs`) son clasificadas como pruebas triviales así:
![contando casos triviales.png](https://www.fpcomplete.com/media/5e104b69-db64-4cc1-b6c4-d4cdc25e99b1.png)

Si en su lugar, lo hubiésemos modificado de esta manera:
```active haskell
import Test.QuickCheck
import Data.List

isSorted :: (Ord a) => [a] -> Bool
isSorted []                     = True
isSorted (x:[])                 = True
isSorted (x:y:zs) | x <= y      = isSorted $ y:zs
isSorted _                      = False

prop_Insert :: Int -> [Int] -> Property
prop_Insert x xs =
  isSorted xs ==>
    {-hi-}collect (length xs){-/hi-} $
      isSorted (insert x xs)

main = quickCheck prop_Insert
```
Podemos ver que los casos de prueba se agrupan por la longitud de las listas con las que se realizaron las pruebas y que además nos muestra el porcentaje con el que dicha longitud ocurrió así:

![agrupando.png](https://www.fpcomplete.com/media/87f87793-419c-4d90-9072-3b3b9e1b6644.png)

En estos dos casos, podemos ver dos cosas. La primera es que QuickCheck se rinde ("gave up") y solo realiza la prueba para un reducido número de casos, esto se debe a que la mayoría de la listas generadas automáticamente, no cumplen con la preconducición de estar ordenadas (`isSorted xs ==> ...`), por lo la gran mayoría de los casos se descarta. También podemos ver que los casos que si cumplieron con la preconducición, no se encuentran distribuidos normalmente, pues es mucho más probable que una lista pequeña generada aleatoriamente esté ordenada a que una lista larga generada aleatoriamente. Para solucionar estos dos problemas, podemos usar un generador especializado que solo genera listas ordenadas, como a continuación:

```active haskell
import Test.QuickCheck
import Data.List

isSorted :: (Ord a) => [a] -> Bool
isSorted []                     = True
isSorted (x:[])                 = True
isSorted (x:y:zs) | x <= y      = isSorted $ y:zs
isSorted _                      = False

prop_Insert :: Int -> [Int] -> Property
prop_Insert x xs =
  {-hi-}forAll orderedList{-/hi-} $ \orderedList -> isSorted (insert x orderedList)

main = quickCheck prop_Insert
```
Y al ejecutar el programa podemos ver como se ejecuta mucho más rapido, pues ya no se pierde el tiempo descartando casos.
![orderdList.png](https://www.fpcomplete.com/media/99dab163-c2c9-4e75-ba8b-a6144e07434c.png)

#### 2.5 Estructuras infinitas
La función de Haskell `cycle` toma una lista no vacía y produce una lista que repite el contenido de dicha lista infinitamente. Podemos entonces definir la siguiente ley como una propiedad de QuickCheck:
```haskell
import Test.QuickCheck
prop_DoubleCycle :: [Int] -> Property
prop_DoubleCycle xs =
  not (null xs) ==>
    cycle xs == cycle (xs ++ xs)
```
Aunque la ley es cierta a simple vista, no la podemos someter a prueba, pues no se puede comparar dos estructuras infinitas en tiempo finito, e.i., tomaría una infinidad de tiempo. Se puede razonar que si tomáramos prefijos aleatorios de ambos ciclos y si estos fueran siempre iguales, entonces hay razón suficiente para creer que los ciclos son iguales:

```active haskell
import Test.QuickCheck

prop_DoubleCycle :: [Int] -> {-hi-}Int{-/hi-} -> Property
prop_DoubleCycle xs {-hi-}n{-/hi-} =
  not (null xs) ==>
    ({-hi-}take n .{-/hi-} cycle) xs == ({-hi-}take n .{-/hi-} cycle) (xs ++ xs)

main = quickCheck prop_DoubleCycle
```
Esta versión sí se puede ejecutar, pues después de crear los ciclos, toma un prefijo de tamaño finito.

Otro asunto relacionado con estructuras infinitas es la cuantificación sobre sobre estas. Veremos más adelante como tratar con propiedades que se cumplen para todas las listas infinitas.

### 3. Generadores
#### 3.1 Arbitrary
La manera en que se generan los casos de prueba aleatorios depende obviamente de su tipo. Por lo tanto, hemos introducido una clase de tipo `Arbitrary`; si queremos generar elementos arbitrarios de un tipo, debemos hacer entonces dicho tipo miembro de la clase `Arbitrary`.

```haskell
class Arbitrary a where
  arbitrary :: Gen a
```

`Gen a` es un tipo abstracto que representa un generador para el tipo `a`. El programador puede usar los generadores incluidos en QuickCheck o definir los suyos usando el combinador `forAll`.

Una primer definición podría ser `newtype Gen a = Gen (Rand -> a)` (más adelante mostraremos la definición completa). En esta definición, `Rand` es la "semilla" de un número aleatorio; un generador es sólo una función que puede manufacturar un valor de tipo `a` de una manera quasi aleatoria.

`Gen` también es un miembro de la clase `Monad` por lo que existen implementaciones para:
```haskell
return :: a -> Gen a
(>>=) :: Gen a -> (a -> Gen b) -> Gen b
```
`return` construye un `Gen` constante (realmente no es arbitrario) y `(>>=)` sirve para crear un `Gen b` usando un `Gen a` dado.

Algunos ejemplos de generadores:
```haskell
import Test.QuickCheck

-- choose (min, max) = an arbitrary Int between min and max.

instance Arbitrary Int where
  arbitrary = choose (-20, 20)

instance (Arbitrary a, Arbitrary b) => Arbitrary (a, b) where
  arbitrary = liftM2 (,) arbitrary arbitrary
```

#### 3.2 Generadores para tipos definidos por el usuario
Al final de la ultima sección, vimos los generadores para los tipos `Int` y `Arbitrary a, Arbitrary b) => (a,b)`; en esta sección, veremos como definir generadores para tipos definidos por el usuario.
Para facilitarnos este trabajo, QuickCheck nos proporciona diversos combinadores; el más sencillo es `oneof` que simplemente escoge un `Gen` de una lista de `Gen`s con una distribución uniforme, por ejemplo:
```haskell
import Test.QuickCheck

data Colour = Red | Blue | Green

instance Arbitrary Colour where
  arbitrary = oneof [return Red, return Blue, return Green]
```
Una manera de generar listas aleatorias:
```haskell
import Test.QuickCheck

instance Arbitrary a => Arbitrary [a] where
  arbitrary = oneof
    [return [], liftM2 (:) arbitrary arbitrary]
```
En este ejemplo, `liftM2` aplica el operador cons, `(:)`, a un `a` y una lista `[a]`. El problema con esta definición es que la longitud promedio de las listas producidas por esta definición, es de 1 sólo elemento. Para resolver esto, podemos utilizar la función `frequency`, la cual nos permite establecer con que frequencia se eligirá cada caso definido como elementos de una lista; a continuación, un ejemplo:
```haskell
import Test.QuickCheck

instance Arbitrary a => Arbitrary [a] where
  arbitrary = frequency
    [ (1, return [])
    , (4, liftM2 (:) arbitrary arbitrary) ]
```
En esta definición, es cuatro veces más probable que suceda el segundo caso que el primero.

Para otro tipo de datos más generales, resulta que necesitamos aun mayor control sobre la distribución de valores generados. Si quisieramos generar árboles arbitrarios siguiendo la mismá técnica que en el ejemplo pasado, nos quedaría algo así:
```haskell
import Test.QuickCheck

data Tree a = Leaf a | Branch (Tree a) (Tree a)

instance Arbitrary a => Arbitrary (Tree a) where
  arbitrary = frequency
    [ (1, liftM Leaf arbitrary)
    , (2, liftM2 Branch arbitrary arbitrary)]
```
El problema con este código es que sólo tiene 50% de probabilidad de terminar de generar el árbol, pues de suceder el segundo caso, son ahora dos árboles (el lado izquierdo y el derecho) los que se tienen que generar, haciendo que estemos más lejos en vez de más cerca de terminar la generación. Aun si llegara a terminar, el árbol generado sería muy grande pare ser apto para pruebas; se necesitan datos no muy largos para poder realizar muchas pruebas.

La solución que ofrece QuickCheck es la de darle al programador la habilidad de escoger que generar en base a un *tamaño*. Para esto, se necesita modificar la definición de `Gen` a `newtype Gen a = Gen ({-hi-}Int{-/hi-} -> Rand -> a)`, donde el primer parámetro, `Int`, introduce la noción de cierto *límite* en el tamaño del valor a generar. Utilizando el combinador `sized :: (Int -> Gen a) -> Gen a`, podemos entonces describir `Gen`s accediendo además a un entero que el programador deberá interpretar como un valor que controla el tamaño del valor a generar. Con esto, podemos entonces crear una generador más apto para generar árboles, por ejemplo:

```haskell
import Test.QuickCheck

data Tree a = Leaf a | Branch (Tree a) (Tree a)

instance Arbitrary a => Arbitrary (Tree a) where
  arbitrary = sized arbTree

arbTree 0 = liftM Leaf arbitrary
arbTree n = frequency
  [ (1, liftM Leaf arbitrary)
  , (4, liftM2 Branch (arbTree {-hi-}(n `div` 2){-/hi-})
                      (arbTree {-hi-}(n `div` 2){-/hi-})) ]
```
De esta manera, a lo mucho el arbol generado puede sólo tener aproximadamente log2(n) niveles, lo cual es un límite bastante razonable.
En cada prueba, varía el tamaño dado a `sized`, comenzando con un `n` pequeño y de ahí en adelante irlo incrementando para las pruebas posteriores. `sized` nos permitirá generar más y con mayor detalle casos de prueba, lo cual a su vez, haría mas eficiente al incrementar nuestras que satisfagan la precondición.

#### 3.3 Generando funciones
Si una de las propiedades que queremos probar incluye una función en su parámetro, necesitaremos poder genarar también funciones arbitrarias. Sorprendentemente, QuickCheck es capaz de esto. Para entender como, nótese que el tipo `Gen (a->b)` (generador de funciones) es igual a este `Int->Rand->a->b`, que reordenando los parámetros, es igual a `a -> Int -> Rand -> b`, que entonces se podría representar como `a -> Gen b`. Entonces, podemos definir una función `promote :: (a -> Gen b) -> Gen (a -> b)` y usarla para producir generadores de funciones a partir de un simple `(a -> Gen b)`, asumiendo que podemos construir un generador `Gen b` que de alguna manera dependa de un `a` dado. Para ayudarnos con esta dependencia que debe tener `Gen b` de `a`, definimos la siguiente clase:
```haskell
class Coarbitrary a where
  coarbitrary :: a -> Gen b -> Gen b
```
cuyo método `coarbitrary` modifica un `Gen b` usando un `a`. Se debe de pensar en `coarbitrary` como un transformador de generadores. Con la clase `Coarbitrary`, podemos entonces definir la instancia de `Arbitrary` para funciones:
```haskell
instance (Coarbitrary a, Arbitrary b) => Arbitrary (a->b) where
  arbitrary = promote (\a -> coarbitrary a arbitrary)
```
Entonces, el trabajo se encuentra en definir instancias de `Coarbitrary` para un `a` de manera que pueda generar un `Gen b` del `a -> Gen b` que nos interese. Para auxiliarnos en esto, utilizaremos la función `variant :: Int -> Gen a -> Gen a`, donde `variant n g` construye un generador que transforma la semilla generadora de un numero aleatorio de alguna manera que dependa de `n` antes de pasársela a `g`. Esta función está definida de manera muy cuidadosa, para que los generadores que construyamos usándola sean independientes. Dada una lista de enteros `[n1,n2,...,nk]`, podemos construir un transformador de generadores: `variant n1 . variant n2 . ... . variant nk`.
Con `variant` como auxiliar, podemos definir instancias de `coarbitrary` que elige entre variantes de un generador dependiendo del primer argumento de `coarbitrary`. Por ejemplo, la instancia de `Coarbitrary Bool` es:
```haskell
instance Coarbitrary Bool where
  coarbitrary b =
    if b then variant 0 else variant 1
```
Se puede ver como la decisión entre `variant 0` y `variant 1` depende del `b`, haciendo que efectivamente `coarbitrary b` tenga el tipo `Gen x -> Gen x`, pues tanto `variant 0` como `variant 1` tienen el tipo `Gen x -> Gen x` y por lo tanto `coarbitrary b g`, donde `g` es un generador `Gen x` produce un `Gen x` que depende de `b`.

Instancias de `Coarbitrary` para tipos recursivos pueden ser definidas siguiendo un patrón estándar, por ejemplo, para las listas es:
```haskell
instance Coarbitrary a => Coarbitrary [a] where
  coarbitrary []     = variant 0
  coarbitrary (x:xs) =
    variant 1 . coarbitrary x . coarbitrary xs
```
El primer caso es simple y el segundo es una serie de transformaciones, donde cada elemento de la lista es crítico en como se influenciará el generador a transformar. El patrón a seguir es: que cada miembro y su posición en la estructura recursiva forme parte del transformador final.

Finalmente, las funciones también se pueden interpretar como transformadores de generadores mediante la siguiente instancia:
```haskell
instance (Arbitrary a, Coarbitrary b) =>
           Coarbitrary (a -> b) where
  coarbitrary f gen =
    do
      a <- arbitrary
      coarbitrary (f a) gen
```
La idea es aplicarle a `f` un valor arbitrario y con el valor generado transformar el generador dado mediante `coarbitrary (f a) gen`.

### 4. Implementando QuickCheck
Como hemos visto, la función quickCheck puede manjear propiedades con un numero variante de argumentos que producen diferentes resultados. Para implementar esto, introducimos el tipo `Property` y creamos la clase `Testable`:
```haskell
class Testable a where
  property :: a -> Property
```
El tipo `Property` representa predicados que se pueden verificar mediante pruebas. Esto significa que necesita producir input aleatorio y calcular un resultado. Entonces, `Property` es una computación dentro del mónada `Gen`, terminando en un tipo abstracto `Result`, el cual mantiene los datos de los resultados de las pruebas, la clasificación de los datos de prueba y los argumentos utilizados en cada caso de prueba. La definición de `Property` es `newtype Property = Prop (Gen Result)`
Veamos algunas instancias de `Testable`:
```haskell
instance Testable Bool where
  property b = Prop (return (resultBool b))

instance (Arbitrary a, Show a, Testable b) => Tetsable (a->b) where
  property f = forAll arbitrary f

instance Testable Property where
  property p = p
```
Usando la función `property`, ahora podemos ver el tipo de `quickCheck`, el cual es `quickCheck :: Testable a => a -> IO()`.

### 5. Algunos casos de estudio
#### 5.1 Unificación
Al intentar probar el algoritmo de unificación con QuickCheck, sucedieron varias revelaciones, tanto en las implicaciones que tiene usar QuickCheck en el diseño como algunas dificultades que se debieron de evitar. Es demasiado grande para presentar a detalle, así que sólo discutiremos las lecciones que aprendimos.
##### 5.1.1 Impacto en las definiciones de tipos
En vez de haber definido el tipo `Term` de esta manera:
```haskell
data Term = Var Int | Constr String [Term]
```
Se tuvo que usar esta otra definición:
```haskell
data Term = Var Var | Constr Name [Term]
newtype Var = Variable Nat
newtype Name = Name String
```
La razón para definir tipos nuevos mediante `newtype` es que nos permite usar `String`s en distintos contextos con semántica distinta. Lo mismo para `Nat`. De esta manera, se puede definir generadores distintos para los nombres de variables (`Name`) que para `String`s comunes y corrientes. Sucede de manera similar con `Var` y `Nat`.
Si no se hubiese definido un tipo especial para los nombres, entonces hubiese sido muy poco probable que dos `String`s aleatorios se unificaran. Pero dado que sí definimos el tipo `Name`, se pudo definir el siguiente generador:
```haskell
instance Arbitrary Name where
  arbitrary = sized $ \n -> oneof
    [ return (Name ("v" ++ show i))
    | i <- [1..log n+1] ]
```
El cual vuelve mucho más probable que se repitan los nombres generados.
Se puede ver entonces como QuickCheck nos hace ser más específicos con los tipos que usamos, que como un extra, le permite al compilador detectar más errores.
##### 5.1.2 Probando propiedades funcionales
Un unificador necesita mantener la sustitución actual y también la posibilidad de errores en llamadas recursivas. Una manera conveniente de hacerlo fue usando un mónada. Definimos la unificación `monad M`, representada por una función con las siguientes operaciones:
```haskell
setV :: Var -> Term -> M ()
getV :: Var -> M Term
```
para leer y escribir variables, entre otras cosas. Fue posible definir el operador `eqM` de igualdad extendida sobre valores monádicos y probar las leyes de los mónadas y propiedades tales como esta:
```haskell
prop_SetGet v t = (do setV v t; getV v) `eqM` (do setV v t; return t)
```
##### 5.1.3 Errores encontrados
Un error que se detectó con QuickCheck fue en la especificación (ningun error se detectó en la implementación), donde se definió una función de sustitución que repetidamente sustituía hasta que ninguna variable sobraba y se definió la siguiente propiedad:
```haskell
prop_SubstIdempotent s t =
  subst s (subst s t) == subst s t
```
A pesar de que a simple vista parecía una propiedad obviamente correcta, QuickCheck demostró que la propiedad sólo se mantenía para sustituciones acíclicas, por lo que se tuvo que redefinir la propiedad de esta manera:
```haskell
prop_SubstIdempotent s t =
  {-hi-}acyclic s ==>{-/hi-} subst s (subst s t) == subst s t
```
Así fue como QuickCheck nos hizo pensar con más detalle en las propiedades de nuestro código y como este quedó mejor documentado.
Por otro lado, resultó ser mucho trabajo definir las propiedades, incluso fue más trabajo definir las propiedades que el algoritmo a probar. Esto se debe en parte porque funciones como `acyclic` no eran triviales de definir; una buena librería de teoría de conjuntos hubiera ayudado.
##### 5.1.4 Una falsa sensación de seguridad
El error más grande que descubrimos con este experimento fue la falsa sensación de seguridad que puede ser engendrada cuando el programa de uno pasa un gran numero de pruebas triviales. Ya nos hemos referido a este problema cuando discutimos las propiedades condicionales; en este ejemplo, nos *mordió duro*.

Muchas propiedades de unificación aplican al caso en que la unificación tiene éxito y pueden ser descritas convenientemente in la forma:
```haskell
prop_Unify t1 t2
  = unifier t1 t2 /= Nothing ==> ...
```
dado que nuestro unificador regresa `Nothing` cuando falla. Después de un poco de reflección, nos dimos cuenta que dos términos elegidos al azar eran muy probablemente unificables, dado que las variables ocurren bastante seguido y si cualquier término es una variable, entonces es muy probable que la unificación tenga éxito. Por el otro lado, si ningón término es una variable, entonces a probabilidad de que se unifiquen es muy pequeña. Por lo tanto, el caso en el que un término es una variable está muy sobrerepresentado entre los casos de prueba que satisfacen la precondición; encontramos que el 95% que satisfacían la precondición era porque alguno de los terminos era una variable.
La solución fue usar un generador especializado:
```haskell
prop_Unify =
  forAll probablyUnifiablePair $ \(t1,t2) ->
    s /= Nothing ==> ...
    where s = unifier t1 t2
```
Generamos "parejas probablemente unificables" al generar un término aleatorio y remplazando subterminos por variables de dos maneras distintas. Este método usualmente generaba términos unificables, pero podía fallar cuando las variables eran usadas inconsistentemente en dos teerminos. Con esta modificación, la proporción de casos triviales disminuyó a un rango razonable de entre 20-25%.
Esta experiencia resaltó la importancia de investigar la distribución de los casos de prueba cuando se usen propiedades condicionales.

#### 5.2 Propiedades de circuitos
##### 5.2.1 Lava en corto
[Lava](http://www.cse.chalmers.se/edu/year/2012/course/TDA956/Papers/Lava98.pdf) es una herramienta que sirve para describir, simular y formalmente verificar software. Lava es un lenguaje embedido (en lenguaje definido en otro lenguaje), lo cual significa que la descripción de los circuitos y sus propiedades están expresados en otro lenguaje, Haskell en este caso.
La idea es ver a un circuito de hardware cono una función de señales de entrada a señales de salida. El sistema Lava provee circuitos primitivos, como **and2**, **xor2** y **delay**. Más circuitos complicatods están definidos por combinaciones de estos. Circuitos definidos en Lava pueden ser simulados de la siguiente manera: uno provee las señales de entrada y se calculan Lava calcula las salidas:
![aaa.png](https://www.fpcomplete.com/media/e4e9361a-954e-47a9-885e-32f28a977857.png)

Además, el sistema Lava define combinadores de circuitos como la composición sequencial `(>->)`, composición paralela `(<|>)` y `column`, el cual toma un circuito y lo replica en una columna de circuitos, conectando los cables verticales.
##### 5.2.2 Propiedades en Lava
Las propiedades de los circuitos se pueden definir en una mnera similar. Por ejemplo, para definir la propiedad de que cierto circuito es comutativo, decimos:
```haskell
prop_Commutative circ (a, b) =
  circ (a, b) <==> circ (b, a)
```
donde `<==>` es la equivalencia lógica *elevada* a tipos arbitrarios conteniendo señales, en este caso un par.
Las propiedades se pueden verificar formalmente. Se hace al proveerle las entradas simbólicas al circuito o propiedad y calculando una expresión concreta en un tipo de Haskell representando el circuito.
Después podemos escribir la expresión en un archivo y llamar a un probador de teoremas externo. Todo esto se hace mediante la función `verify`. Así es como se verifica que un [sumador](https://es.wikipedia.org/wiki/Sumador) es comutativo:
```haskell
main = verify (prop_Commutative halfAdd)
```
El sistema Lava provee un número de funciones y combinadores que convenientemente expresan propiedades y formalmente las verifican.
##### 5.2.3 QuickCheck y Lava
Aunque fuimos capaces de verificar las propiedades acerca de circuitos en Lava, nos beneficiamos de extenderlo con una herramienta de pruebas como QuickCheck. Hay dos razones para esto:
Primero, llamar un probador de teoremas externos es un proceso pesado. Verificar un multiplicador de 32 bits, las fórmulas que generamos son bastante grandes y toman mucho tiempo en comprobarse.
Entonces, un ciclo típico de desarrollo es escribir las especificaciones del circuito primero, luego escribir una implementación, QuickCheck es para detectar bugs obvios y, por último, llamar al probador de teoremas externo para verificar. Añadir QuickCheck a Lava resulto ser bástante fácil, pues Lava ya tiene algunas nociones de propiedades.
##### 5.2.4 Pruebas de orden superior
La segunda razón para usar pruebas en Lava es simplemente que ¡podemos probar más propiedades de las que formalmente se pueden verificar! Los probadores de teoremas conectados a Lava apenas pueden manejar lógica de primer orden y el sistema de Lava solo es capas de generar formulas de ese tipo.
A veces, quisieramos probar propieadeds acerca de combinadores. Por ejemplo, provar que `column` se distribuye sobre `(>->)`:
```haskell
prop_ComposeSeqColumn circ1 circ2 inp =
  column (circ1 >-> circ2) inp <==> (column circ1 >-> column circ2) inp
```
es muy difícil de verificar en Lava para todo `circ1` y `circ2`. De hecho, dichas propiedades son difíciles de verificar automáticamente en general (podemos hacerlo para pequeños tamaños), pero dado que podemos generar funciones aleatorias, podemos al menos probar este tipo de propiedades para circuitos arbitrarios (dado que los circuitos son funciones).
Un inconveniente es que tenemos que fijar los tipos de estos circuitos, mientras que los combinadores de Lava, y por lo tanto sus propiedades, son polimórficas.
##### 5.2.5 Errores encontrados
Los errores encontrados gracias a QuickCheck fueron de dos tipos. El primer tipo de errores fueron errores que también se hubiesen podido encontrar con los probadores formales: errores lógicos. Pero el segundo caso si se trata de errores detectados únicamente por QuickCheck, dado que QuickCheck permitió probar con una variedad de entradas de tamaño aleatorio, mientras que con el método formal, siempre se probaba para tamaños preestablecidos.
#### 5.3 Probador de teoremas proposicional
Para fines educativos, implementamos dos diferentes y bien conocidos métodos para checar si un conjunto de cláusulas de lógica proposisional son contradictorios. Uno de estos métodos fue el método de [Davis-Putnam](https://en.wikipedia.org/wiki/Davis%E2%80%93Putnam_algorithm), el cual utiliza backtracking para general una lista de todos los modelos. El otro método fue el [método de Stålmarck](http://www.cfdvs.iitb.ac.in/download/Docs/verification/papers/sat/presentations/stalmarck-presentation.pdf).
```haskell
type Clause = [Lit] -- disjunction
type Model = [Lit] -- conjunction

davisPutnam :: [Clause] -> [Model]
stalmarck    :: Int -> [Clause] -> Maybe Model
```
La función `stålmarck` toma un argumento extra, un `Int` que es el llamado "nivel de saturación", un parámetro que limita la profundidad de las pruebas y usualmente se encuentra entre 0 y 3. Si el resultado de `stålmarck` es `Nothing`, significa que hubo una contradicción. Si el resultado es `Just m`, significa que cada modelo dol conjunto de cláusulas debe tener `m` como un submodelo.

Dado que `davisPutnam` es mucho mas directo de implementar que `stalmarck`, quizimos checar la implementación del segundo con la del primero. Aquí es como formulamos la propiedad descrita de manera informal en este parrafo:
```haskell
prop_Stalmarck_vs_DP :: Property
prop_Stalmarck_vs_DP =
  forAll clauses $ \cs ->
    forAll (choose (0, 3)) $ \sat ->
      case stalmarck sat cs of
        Nothing ->
          collect "contradiction" $
            davisPutnam cs == []

        Just m ->
          not (null m) ==>
            collect (length m) $
              all (subModel m) (davisPutnam cs)
```
Nótese que se collecta algo de información estadística: `collect "contradiction"` cuando el resultado fue `Nothing`, y el tamaño de `m` en caso de `Just m`. También se especificó que se debe de descartar el caso en que `stålmarck` regresa `Just []`: `case stålmarck sat cs of ... Just m -> not (null m) ==> ...`
¡Con esta propiedad, QuickCheck encontró 3 bugs! Estos bugs se debieron a supuestos injustificados implicitos que teníamos sobre la entrada. Las implementaciones de ambos algoritmos asumían que ninguna cláusula en la entrada pudo contener la misma literal dos veces y la función `stålmarck` asumía que ninguno de las cláusulas de entrada se encontraban vacias.
El generador de datos de prueba, `clauses` fue definido usando las mismas técnicas que en la sección "Impacto en la definicón de tipos". Probar la propiedad tomó 30 segundos y de la salida se pudo ver que la distribución de `Nothing` y `Just m` era de alrdedor de 50/50.
#### 5.4 Pretty Printing
Omitido.
#### 5.5 Edison
Omitido.
### 6. Discusión
#### 6.1 Pruebas aleatorias
A primera vista, la selección aleatoria de los casos de prueba podría parecer un enfoque muy ingenuo. Métodos sistemáticos son usualmente preferidos; en general, un *criterio de pruebas adecuadas* es definido y el proceso de pruebas procede al generar casos de prueba que cumplen con el criterio de suficiencia. Por ejemplo, un criterio simple es que cada estatuto alcanzable debería de ser ejecutado al menos en una prueba; un criterio más complejo es que cada camino de un estatudo de control (excepto por los ciclos) debería de ser seguido por al menos una prueba. Se ha propuesto una gran variedad de criterios, [he aquí un estudio](http://www.cs.toronto.edu/~chechik/courses07/csc410/p366-zhu.pdf).
Se eligió no basar QuickCheck en dichos criterios. En parte, esto es porque muchos criterios necesitarían ser reinterpretados antes de que pudieran ser aplicables a Haskell; es mucho menos claro, por ejemplo, que es un *camino de control de flujo* en un lenguaje con funciones de orden superior y evaluación perezosa. En parte, usar dichos criterios nos obligarían a usar métodos más pesados, incluso poder medir la covertura de todos los caminos, por ejemplo, requeriría modificaciones al compilador y por lo tanto, QuickCheck estaría atado a cierto compilador de Haskell. Generar datos de prueba para ejecutar un camino en específico requeriría analizar y resolver restricciones (para encontrar los valores adecuados que hagan ejecutarse cierto camino con ciertos valores). Aunque poder analizar y cumplir dichas restricciones sea posible para datos aritméticos, es muy difícil lograrlo para la variedad de tipos expresables en Haskell.

De todos modos, además de la dificultad de crear dicho sistema, no hay razón clara para creer que hacerlo produciría mejores resultados. En 1984, Duran y Ntafos compararon la probabilidad de detección de error para pruebas aleatorias con *partition testing* y descubrieron que las diferencias en efectividad eran pequeñas. Hamlet y Taylor repitieron su estudio de manera mas extensiva, corroborando los resultados originales. Aunque *partition testing* es ligeramente más efectivo para exponer errores, usar 20% más casos de prueba con en pruebas aleatorias eliminaba cualquier ventaja que tuviera *partition testing*.

Para pequeños programas en particular, es posible que las pruebas aleatorias van a ejecutar todos los posibles caminos. Usando QuickCheck, pudimos aplicar pruebas aleatorias a un grado bastante refinado: probamos propiedades de funciones individuales, pero las funciones que llaman son probadas independientemente. Entonces, aunque QuickCheck sea usado para probar un programa grande, siempre probamos una pequeña parte a la vez, por lo que se espera que las pruebas aleatorias funcionen particularmente bien.
Dado todo esto, elegir hacer pruebas aleatorias con QuickCheck nos pareció una opción viable.
#### 6.2 Criterios de exactitud
El problema de determinar si una prueba pasó o no es conocido como el problema del oráculo. Una solución es la de comparar la salida de un programa con la de otra versión del programa, quizás una mas antigua o mas simple y lenta pero obviamente correcta. Alternativamente, una especificación ejecutable podría jugar el mismo papel. Este tipo de oráculos pueden ser expresados simplemente como una propiedad de QuickCheck, aunque nuestras propiedades son mucho más generales.
Por otro lado, uno puede verificar que la salida de un programa es correcta de manera mucho mas rápido que uno puede computar la salida. Blum y Kannan utilizaron esto en su trabajo [Designing programs that check their work](http://dl.acm.org/citation.cfm?id=73015&dl=ACM&coll=DL&CFID=549961787&CFTOKEN=20087942); un verificador de programas está definido a ser otro programa que clasifica la salida del programa como correcta o erronea, con una alta probabilidad de realizar la clasificación correctamente y lo hace con estricta baja complejidad. Distinguen entre *probar* y *verificar* un programa; su propuesta es que los programas deben siempre de *verificar* su salida y de hecho Blum et al. mostraron como los programas que usualmente producen resultados correctos pueden incluso corregir resultados equivocados (en ciertos dominios). Por supuesto, verificadores de resultados igual se pueden expresar como propiedades de QuickCheck, aunque los usamos para probar un programa, mas que como una parte del programa final.
Además, el lenguaje de propiedades de QuickCheck es más general que un verificador de resultados. Mediante las propiedades condicionales o generadores de datos específicos, podemos expresar propiedades que se mantienen solo para un pequeño subconjunto de todas las posibles entradas. Asi, podemos evitar probar funciones en casos que llevan a errores en tiempo de ejecución, o casos en que no nos importa el resultado. Por ejemplo, no tiene sentido probar la función de inserción para una lista desordenada. Por otro lado, un verificador de resultados debe verificar que un programa produce la salida correcta en todos los casos, aun para los casos que no son interesantes. También, las propiedades de QuickCheck no están limitadas a la verificación del resultado de la llamada de una función, por ejemplo, la propiedad de asociatividad de un operador no se puede probar mediante la llamada de una función, pero aun así expresa una propiedad global útil que se puede probar mediante pruebas.
...
Se observó que los programas funiconales y las propiedades basadas en especificaciones eran un buen match y hacerlo en Haskell nos facilita hacerlo con una mínima curva de aprendizaje.
#### 6.3 Generación de datos de prueba
Herramientas comerciales de pruebas aleatorias generan datos de prueba para dominios limitados, en el que el objetivo es hacer que los casos de prueba se asejemen a casos reales de ejecución, descrito por el perfil operacional. En este caso, se pueden obtener inferencias estadísticas sobre el tiempo promedio de fallas del sistema.
Para generar datos más complejos, es necesario proveer una descripción de la estructura de los datos. Un enfoque popular para hacerlo es usando gramáticas. Pero, se entendio pronto que las gramáticas libres de contexto no pueden expresar todas las propiedades deseadas de los datos de prueba, por ejemplo, un programa generado aleatoriamente no contiene identificadores sin declarar. Por lo tanto, las gramáticas fueron mejoradas con acciones o extendidas a gramáticas atribuidas. Este enfoque ha sido el más usado para probar compiladores, aunque Maurer argumenta que tiene muchos otros usos.
Las gramáticas se han usado para pruebas sistemáticas, donde, por ejemplo, los datos de prueba generados es requerido para expandir cada producción al menos una vez. Un criterio de adecuación como este para probar compiladores. Maurer también utilizó gramáticas para pruebas aleatorias y notó el problema de terminación para gramáticas recursivas. Su solución, sin embargo, fue simplemente la de incrementar las probabilidades de generar hojas de tal forma que se garantizara la terminación. Nuestra experienca dice que esto resulta en, por mucho, una elevada proporción de casos triviales de prueba y por lo tanto en pruebas ineficientes dado que más tests deben de ejecutarse para probar el programa apropiadamente. Creemos que nuestro método para controlar el tamaño es muy superior.
Parece que la necesidad de aprender un lenguaje más complejo para gramáticas extendidas ha obstaculizado la adopción de estos métodos en la práctica. Al embeder un lenguaje generador de pruebas en Haskell, nosotros proveemos (por lo menos) las mismas capacidades, pero así el programador tiene sólo que aprender no mas que unos pocos operadores. Al mismo tiempo, nosotros porveemos todo el poder y flexibilidad que se necesitan para generar datos de prueba que satisfacen invariantes complejas en un lenjuage que el programador ya conoce. Al ligar generadores a tipos via el sistema de tipos de Haskell, liberamos del todo al programador de tener que especificar generadores en muchos casos y donde tienen que ser especificados, usualmente el programador solo tiene que escribir generadores para sus propios tipos.
#### 6.4 Sobre la alateoridad
Nos hemos encontrado algunos problemas interesantes al razonar sobre programas que utilizan generación de números aleatorios. En particular, el mónada `Gen` no es realmente un mónada. Recordando la primer ley de los mónadas:
`return x >>= f = f x`
Dado que nuestra implementación de `>>=` divide la semilla de su número aleatorio, entonces a `f` se le pasan *diferentes* semillas en ambos lados de la ecuación y por lo tanto puede producir diferentes resultados. Así que la ley simplemente no se cumple.
 Moralmente, consideramos que la ley se cumple pues ambos lados de la ecuación producen resultados con la misma distribución. Por "moralmente" nos referimos a que si nos permitimos reinterpretar la equivalencia de dos programas cuando se incluye la presencia de un generador de números aleatorios.
#### 6.5 Sobre la evaluación perezosa
Hemos argumentado en el pasado que la evaluación perezosa es una invaluable herramienta de programación que radicalmente cambia la manera en que los programas pueden ser estructurados, pero QuickCheck sólo es capaz de probar propiedades computables; hay conflicto?
De hecho, el conflicto es menor de lo que uno podría imaginar. Como ya vimos arriba, podemos usar sin problemas estructuras infinitas en las especificaciones, asumiendo que las propiedades que probamos son de hecho computables; por ejemplo, podemos probar que arbitrariamente largos prefixos en listas infinitas son iguales, en vez de comparar las listas enteras. El mónada `Gen` tiene un operador `>>=` perezoso (dado que dividimos la semilla generadora de números aleatorios, en vez de pasarlo por el primero y luego por el segundo), así que podemos libremente definir generadores que producen resultados infinitos. Lo que no se puede hacer es observar la no terminación, por lo que no podemos probar esta propiedad:
`reverse (xs ++ undefinde) == undefined`
El problema es que no hay manera de que un programa pueda detectar que otro programa produce un error al evaluar `undefined`. Existen algunas extensiones de Haskell que permiten esto y Andy Gill ha demostrado que, dadas dichas extensiones, podemos formular y checar propiedades como la de arriba.
#### 6.6 Algunas reflexiones
Estamos convencidos que una de las mayores ventajas de usar QuickCheck es que nos alienta a formular especificaciones formales y por lo tanto nos ayudan a entender nuestros programas. Aunque es decisión del programador hacerlo de todos modos, pocos lo hacen, quizás porque hay poca ganancia a corto plazo y quizás porque la especificación es menos valiosa si no hay verificación que corresponda al programa implementado. QuickCheck atiende estos dos problemas: nos da una recompenza a corto plazo via las pruebas automatizadas, y nos da razones para creer que las propiedades establecidas para cierto modulo se mantienen.
Hemos observado que los errores que encontramos están divididos casi uniformemente entre los errores en los datos de prueba generados, errores en la especificación y errores en el programa. La primer categoría, errores en los datos de prueba, sólo es útil en el sentido en que corregirlos nos ayuda a entender aun más nuestro programa y las propiedades que queremos probar. La tercer categaría es obviamente útil, en el sentido de que encontrar estos errores son los motivos por los que hacemos pruebas, pero la segunda categoría igual es importante: incluso si los errores en la especificación no revelan un error en el código, revelan un mal entendido sobre lo que hace. Corregir dicho malentendido mejora nuestra habilidad de hacer uso del código del código corregido después.
Cuando se están formulando las especificaciones, uno rapidamente descubre la necesidad de una librería de funciones que implementan abstracciones matemáticas comunes. Nosotros estamos desarrollando una implementación de de la teoría de conjuntos finitos usando QuickCheck; muchas de las abstracciones en ella son ineficientes para ser usadas en programas, pero en las especificaciones, donde el objetivo es establecer las propiedades los más claro y simple posible, encuentran su caso de uso.
La mayor limitación de QuickCheck es que no hay manera de medir la cobertura de las pruebas, queda como responsabilidad del programador investigar la distribución  de los datos de prueba y decidir si se ejecutaron suficientes pruebas. Aunque QuickCheck ofrece maneras de colectar la información de los datos de prueba, no podemos obligar al usuario a utilizarlas; el riesgo de no hacerlo es obtener una falsa sensación de seguridad sólo por ejecutar un grande número de pruebas.

### 7. Conclusiones
Hemos tomado dos ideas relativamente viejas: especificaciones como oráculos y pruebas aleatorias, y encontramos maneras de hacerlas asequibles para los programadores de Haskell. Primero, proveímos un lenguaje embedido para escribir propiedades, aportando expresividad sin ningún costo de aprendizaje. El lenguaje contiene características convenientes, tales como los cuantificadores, condicionales y monitores de pruebas de datos. Segundo, proveímos generadores de pruebas aleatorios estándar para los tipos estándar de Haskell, ayudando a reducir mucho el costo de implementarlos. Tercero, proveímos un lenguaje embedido para especificar generadores de pruebas aleatorios, los cuales pueden ser basados en los generadores estándar, dándole mayor control al programador sobre la distribución de los datos de prueba. También introdujimos una novedosa forma de controlar los tamaños de los datos aleatoriamente generados de tipos de datos recursivos.
Además, demostramos que la combinación de estas viejas técnicas funcionan extremadamente bien para Haskell. La naturaleza de la programación funcional permite descibir propiedades muy específicas y de manera local al hacer explícitas todas las dependencias de una expresión. Las pruebas aleatorias son especialmente útiles para encontrar errores en este tipo de expresiones pequeñas y detalladas.
Por último, la herramienta es ligera y fácil de usar y provee de recompensa a corto plazo para establecer explícitamente propiedades de las funciones de un programa, lo cual incrementa notoramiente el entendimiento de un programa, tanto para el programador como para fines de documentación.

### Agradecimientos
Los autores del árticulo original agradecen a Andy Gill, Chris Okasaki y a quienes anonimamente dieron sus comentarios sobre esté artículo.

### 8. Referencias
[Ver la sección 8 del artículo original](http://www.eecs.northwestern.edu/~robby/courses/395-495-2009-fall/quick.pdf).

### Apéndice sobre la implementación
[Ver la sección "Appendix: Implementation" del artículo original](http://www.eecs.northwestern.edu/~robby/courses/395-495-2009-fall/quick.pdf) o [el código fuente](https://github.com/nick8325/quickcheck).

## Ejercicios
[Ejercicios](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-8/ejercicios)
[Soluciones](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-8/soluciones)
