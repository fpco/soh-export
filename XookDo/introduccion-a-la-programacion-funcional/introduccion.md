# Introducción

Este es el primer tutorial de la serie *Introducción a la Programación Funcional*. A lo largo del curso, se concocerán los fundamentos de dicho paradigma y, al usar **Haskell** como nuestro lenguaje, obtendremos las bases para usar Haskell en producción o el conocimiento necesario para usar librerías funcionales en lenguajes tradicionales (Java, C#, Javascript, etc.) y ser prolífico en lenguajes más funcionales como F#, Scala, etcétera.
Los temas que se abarcarán en el curso son:

- [Un primer vistazo a Haskell](https://www.fpcomplete.com/user/XookDo/introducci-n-a-la-programaci-n-funcional/parte-1/tutorial)
- [Funciones y tipos de datos algebraicos](https://www.fpcomplete.com/user/XookDo/introducci-n-a-la-programaci-n-funcional/parte-2/tutorial)
- [Algo sobre listas y todo sobre funciones](https://www.fpcomplete.com/user/XookDo/introducci-n-a-la-programaci-n-funcional/parte-3/tutorial)
- [Azúcar sintáctico y otros artilugios](https://www.fpcomplete.com/user/XookDo/introducci-n-a-la-programaci-n-funcional/parte-4/tutorial)
- [Clases de tipos](https://www.fpcomplete.com/user/XookDo/introducci-n-a-la-programaci-n-funcional/parte-5/tutorial)
- Mónadas
- Testeo

En cursos subsecuentes de Haskell, iremos más allá de las bases y abordaremos todo lo necesario para usar Haskell en producción; *desde* crear APIs, páginas web dinámicas, multithreading *hasta* hacer simulaciones sobre un modelo matemático.

En cursos subsequentes de *programación funcional*, veremos como podemos usar otros lenguajes funcionales como F# para crear aplicaciones móviles nativas.

Si no tienes mucha experiencia con la programación funcional, tienes mucho que aprender y por lo tanto tomaré como responsabilidad ser lo más sucinto posible. Mi objetivo es que puedas trabajar expresándote de manera formal y elegante.

Si algo no queda del todo claro al leerlo, seguramente es por diseño y será aclarado más adelante en el mismo tutorial o en tutoriales siguientes. Si crees que algo no está bien, puedes mandarme un correo a laygr@outlook.com.

Para realizar los ejercicios, puedes crear una cuenta en [fpcomplete](https://www.fpcomplete.com) y así poder ir utilizando los templates preparados para los tutoriales.

### Reconomiento
La mayoría de la información fue tomada de documentos publicados en [Internet]() y de algunos libros: [The Haskell School Of Expression](http://www.amazon.com/Haskell-School-Expression-Functional-Programming/dp/0521644089/ref=mt_paperback) por [Paul Hudak](http://haskell.cs.yale.edu/people/paul-hudak/) y [Automata, Computability and Complexity: Theory and Applications](http://www.amazon.com/Automata-Computability-Complexity-Theory-Applications/dp/0132288060) por [Elaine Rich](http://www.cs.utexas.edu/~ear/). Mi labor ha sido simplemente la de recabar y sintetizar la información; quizás lo único que hacemos todos en realidad. Otras cosas las recuerdo de mis tiempos de estudiante en el [ITESM](http://www.itesm.mx/wps/wcm/connect/Campus/MTY/Monterrey); particularmente la clase de Lenguajes de Programación, con [Santiago Conant](http://homepages.mty.itesm.mx/sconant/) y Matemáticas Discretas, con [Alejandra González](http://cs.mty.itesm.mx/portal/sites.google.com/site/mtycs1/Listado_de_Profesores/alejandra-gonzales.html).

## ¿Por qué estudiar programación funcional pura?



### ¿Qué es ser funcional?
Un programa es algo que recibe una entrada y computa una salida. Por ejemplo, un programa que calcula la edad promedio de una familia recibiría como entrada los datos de la familia y produciría como salida la edad promedio.
Este sencillo programa se puede modelar como una función `avg` en Haskell. Si modelamos la edad como un entero (`Int`), la función tendría como entrada una *lista de edades* (`[Int]`) y como salida una *edad promedio* (`Int`). La signatura de esta función sería `avg :: [Int] -> Int`.

Un **lenguaje funcional puro** es aquel cuyas aplicaciones son una composición de funciones puras. La interacción entre el software y el mundo real se hace entonces, en el caso de Haskell, mediante el uso del ***I/O monad*** (`IO ()`) que veremos a detalle más adelante en el tutorial dedicado a los mónadas.

En Haskell, la ejecución de un programa es la ejecución de su función `main :: IO ()` y todas las subllamadas que deriven de esta. La función main es de tipo `IO ()` y por lo tanto puede interactuar con el mundo real (base de datos, interfáz humano-computadora, etc.).

### ¿Qué es ser *puro*?
Un lenguaje *puro* (*pure*) es aquel cuyas expresiones

- siempre se evaluan al mismo resultado dados los mismos argumentos. Es decir, su resultado no depende de un entorno global, e.g. variables estáticas.
- no causan efectos secundarios al ser evaluadas.
- son inmutables, o sea, que el significado de una expresion no puede cambiar en tiempo de ejecución.

Esto nos obliga a separar claramente el estado del comportamiento, lo cual es bueno. Por otro lado, mantener el estado de manera eficiente es el principal reto de programar en Haskell, pero hay librerias que nos harán el trabajo sencillo.

### ¿Qué es ser fuertemente tipado?
Un lenguaje *fuertemente tipado* (*strongly typed*) es aquel cuyos tipos no cambian en tiempo de ejecución. Esto implica que con sólo ver los tipos de una función, ya podemos saber mucho de su intención.

En Haskell, declarar los tipos de una función es casi siempre opcional, pues el compilador (al menos [GHC](https://www.haskell.org/ghc/)) inferirá todos los tipos posibles de inferir. Esto nos ahorrará mucha escritura sin perder los beneficios de un lenguaje fuertemente tipado. A partir de ahora, **omitiré la declaracón de la *signatura*** (declaración de tipos o "firma") de las funciones y valores cuando sea apropiado.

Ser fuertemente tipado nos evita *errores de tipo* en tiempo de ejecución pues el compilador los detecta en tiempo de compilación. A continuación, declararemos una funcion que recibe un `Int`, pero le pasaremos un `String` para **provocar un *type error*** en tiempo de compilación.

``` active haskell
increaseOne :: Int -> Int
increaseOne x = x + 1

main = do
    putStr "increaseOne \"¡Hola!\" = "
    putStrLn (show (increaseOne "¡Hola!"))
```
>El error mencionará un problema con un `[Char]`; El tipo `String` en Haskell es un sinónimo para una lista de caracteres o en otras palabras, en Haskell, `String` y `[Char]` son sinónimos.

**Siempre** es mejor detectar un error en nuestro código en *tiempo de compilación* que en *tiempo de ejecución*.

### ¿Qué es ser *flojo*?
Un lenguaje *flojo* (*lazy*) es aquel cuyas expresiones no son evaluadas hasta ser necesario. Esto implica que podemos pasar funciones en los parámetros y estas podrían nunca ser evaluadas.

En este ejemplo, los parámetros *first* y *second* sólo se evaluan cuando es necesario. Cuando llamamos la función `f` con el tercer parámetro {`True`|`False`}, el {primer|segundo} parámetro es devuelto y el
{segundo|primero} nunca es evaluado. Esto nos permite componer funciones de manera eficiente.
``` active haskell
f first second True = first
f first second False = second

main =
  do
    putStr "f (putStrLn \"hello\") (putStrLn \"goodbye\") True = "
    f (putStrLn "hello") (putStrLn "goodbye") True
    putStr "f (putStrLn \"hello\") (putStrLn \"goodbye\") False = "
    f (putStrLn "hello") (putStrLn "goodbye") False
```
Ser flojo también facilita trabajar con *Streams*, es decir, series infinitas de datos, pues, al ser flojo, se pueden expresar datos infinitos sin tener que calcularlos. Por ejemplo, podemos expresar una lista infinita de enteros y sólo usar los primeros 3 elementos:
``` active haskell
infinite_list = [1, 2 ..] -- [1, 2, 3, etc...]

main = putStrLn (show (take 3 infinite_list))
```
Si Haskell no fuera flojo, antes de usar la lista se tendría que calcular toda esta y, al ser esta infinita, tomaría una infinidad de tiempo.

### ¿Cómo es la abstracción en un lenguaje funcional?
Básicamente, la abstracción consiste en parametrizar una función. A la función `increaseOne` se le puede parametrizar el valor `1` y llamarse entonces `increaseN`, como lo haremos a continuación.

``` active haskell
{-
Antes:
increaseOne :: Int -> Int
increaseOne x = x + 1
-}

increaseN :: Int -> Int -> Int
increaseN x n = x + n

main =
  do
    putStr "increaseN 1 2 = "
    putStrLn (show (increaseN 1 2))
```

Podemos notar que ahora la función `increaseN` recibe *dos* argumentos en vez de *uno* y por lo tanto el tipo de la función cambió de ser `Int -> Int` (recibe un `Int` y produce un `Int`) a `Int -> Int -> Int` (recibe dos `Int`s y produce un `Int`).

> Técnicamente, todas las funciones en Haskell reciben un solo argumento y producen un solo valor, ya sea otra función o un valor sencillo. En el siguiente tutorial veremos el tema de *partial application* con más detalle; para leer más, visita [Haskel.org - Partial application](https://wiki.haskell.org/Partial_application).

Para continuar con el proceso de abstracción, podríamos abstraer la suma de la función `increaseN` y llamarla `binOpApp` (aplicación de un operador binario o en inglés "binary operator [application]()") de la siguiente manera.

``` active haskell
binOpApp :: Int -> Int -> (Int -> Int -> Int) -> Int
binOpApp x y binOp = binOp x y

main =
  do
    putStr "binopApp 1 2 (+) = "
    putStrLn (show (binOpApp 1 2 (+)))
    putStr "binOpApp 1 2 (-) = "
    putStrLn (show (binOpApp 1 2 (-)))
```
Lo más destacable es que el tipo de la función `binOpApp` nos indica que su tercer parámetro es una funcion que recibe dos `Int`s y produce un `Int`.

### ¿Cómo es la refactorización en un lenguaje funcional?
Existen varias refactorizaciones válidas (*semantic preserving*) en Haskell. Algunos ejemplos son:

- Movimiento del orden de parámetros. Podemos modificar `binOpApp` para que `binOp` sea su primer parámetro.
``` active haskell
binOpApp :: (Int -> Int -> Int) -> Int -> Int -> Int
binOpApp binOp x y = binOp x y

main =
  do
    putStr "binOpApp (+) 1 2 = "
    putStrLn (show (binOpApp (+) 1 2))
    putStr "binOpApp (-) 1 2 = "
    putStrLn (show (binOpApp (-) 1 2))
```
- Después podemos hacer *eta reduction* al remover la `x` y la `y` en ambos lados:
``` active haskell
binOpApp :: (Int -> Int -> Int) -> Int -> Int -> Int
binOpApp binOp = binOp

main =
  do
    putStr "binOpApp (+) 1 2 = "
    putStrLn (show (binOpApp (+) 1 2))
    putStr "binOpApp (-) 1 2 = "
    putStrLn (show (binOpApp (-) 1 2))
```
> En esta definición de `binOpApp`, la signatura de su tipo es crucial para su funcionamiento; si no fuera por la signatura, podría ser ambiguo el número de parámetros que recibe y por lo tanto su significado dependería de como `binOpApp` sea usada y de lo que el compilador infiera de dicho uso. Al menos en este ejemplo, el código funciona sin la signatura de `binOpApp`.

Ahora debe de quedar claro que `binOpApp` es completamente redundante y que se ha utilizado simplemente como herramienta pedagógica.

Otras refactorizaciones (más no todas) son:

- la introducción de lambdas (funciones anónimas) (*lambda abstraction*)
- renombramiento de variables (*alpha conversion*)
- la aplicación de un valor a una función (*beta reduction*)

Lo especial de Haskell es que su sistema de tipos nos facilita mucho las refactorizaciones por detectar muchos errores en tiempo de compilación; algunos dicen que dado un programa que funciona, una refactorización que compila probablemente sea correcta. Otros simplemente dicen que si el código compila, probablemente funciona. [Why Haskell just works](https://wiki.haskell.org/Why_Haskell_just_works).