# Funciones y tipos de datos algebraicos

En el tutorial pasado, usamos funciones sin su explicación completa. En este tutorial abarcaremos los detalles de las funciones en Haskell y las estructuras de datos sobre las que estas operan, los tipos de datos algebraicos. Para unir estos dos temas, también abarcaremos el tema de **búsqueda de patrones**.

## Las funciones en Haskell
Como ya habiamos dicho, todas las funciones en Haskell reciben un solo valor y producen un solo valor. El tipo de una función `f` que recibe una instancia del tipo `T` y produce una instancia del tipo `R` se firma en Haskell con la siguiente signatura.
```haskell
f :: T -> R
```
Pero gracias a la **currificación automática** de Haskell, podemos expresar funciones que reciben múltiples parámetros de manera natural. La signatura de una función `f` que recibe **n** parametros y produce un valor de tipo `R` se describe de la siguiente manera.
```haskell
f :: T1 -> ... -> Tn -> R
```

Como ejemplo de la currificación automática, analizemos la función `binOpApp`. Recordemos que `binOpApp` es una función que

- recibe un operador binario de enteros
- recibe dos enteros
- produce el resultado de aplicar el operador binario a los enteros.
```haskell
binOpApp :: (Int -> Int -> Int) -> Int -> Int -> Int
binOpApp binOp = binOp
```

Una notación alterna de su firma es:
```active haskell
binOpApp :: (Int -> Int -> Int) -> (Int -> Int -> Int)
binOpApp binOp = binOp

main = putStrLn (show (binOpApp (+) 5 10))
```
Ahora queda claro que `binOpApp` se puede firmar como una función que recibe un solo parámetro y regresa un solo valor. Sin embargo, es posible que no haya quedado claro su funcionamiento y para eso haremos la aplicación de sus parametros paso a paso, utilizando una técnica llamada **partial application**.
Primero aplicamos el operador binario `+`, que para usarlo como función en vez de como operador, necesitamos rodearlo de paréntesis (como lo hemos estado haciendo).

```haskell
binOpApp :: (Int -> Int -> Int) -> Int -> Int -> Int
binOpApp binOp = binOp

binOpApp1 :: Int -> Int -> Int
-- que es lo mismo que
binOpApp1 :: Int -> (Int -> Int)

binOpApp1 = binOpApp (+)
```
Después podemos aplicar un entero, el `5`, por ejemplo.
```haskell
binOpApp2 :: Int -> Int
binOpApp2 = binOpApp1 5
```
Y luego otro, el `10`, por ejemplo, para obtener finalmente el resultado.
```haskell
binOpApp3 :: Int
binOpApp3 = binOpApp2 10
```
Todo junto y ejecutable:
```active haskell
binOpApp :: (Int -> Int -> Int) -> (Int -> Int -> Int)
binOpApp binop = binop

binOpApp1 = binOpApp (+)
binOpApp2 = binOpApp1 5
binOpApp3 = binOpApp2 10

main = putStrLn (show binOpApp3)
```

La [asociatividad](http://www.angelfire.com/planet/maipage/compi/asociatividad.html) del operador [`->`](https://wiki.haskell.org/Keywords#-.3E) es a la derecha, por lo que que cada vez que vemos algo como:
`:: T1 -> T2 -> T3 -> ... -> Tn`, en realidad es `:: T1 -> (T2 -> (T3 -> (... -> Tn)))`.

## Tipos de datos algebraicos
Las funciones son, en general, mapeos de un dato a otro. Una estructura de datos junto con un nombre forman un tipo; es decir, dos datos pueden tener la misma estructura y ser tipos distintos si tienen distintos nombres.

> No pueden haber dos tipos con el mismo nombre dentro del mismo namespace, pues no se sabría a cual tipo se está refiriendo (habría ambigüedad). Si se requiere usar un tipo de un módulo cuyo nombre ya está siendo utilizado, seguir [este wiki](https://wiki.haskell.org/Import).

Ya que hemos entendido lo básico de las funciones, es momento de aprender como construir los tipos sobre los cuales las funciones operarán. Los **tipos de datos algebraicos** son usados para definir tipos en Haskell.

Los *tipos* en Haskell se producen mediante los ***constructores de tipos*** (***type constructors***); una manera de pensar sobre los *constructores de tipos* es que son funciones especiales que sólo reciben tipos y producen tipos. Cuando un *constructor de tipos* no recibe ningún parámetro, se dice que es miembro de la especie `*`; cuando un *constructor de tipos* recibe un tipo miembro de la especie `*`, se dice que es miembro de la especie `* -> *` y etc. Los constructores de tipos se pueden combinar mediante ***multiplicación*** y ***adición***.

Como primer ejemplo, veamos las *enumeraciones*, que son los tipos más sencillos en Haskell.

### Enumeraciones (especie `*`)
Una *enumeración* es un *constructor de tipos* miembro de la *especie* `*`, es decir, que no recibe ningún parámetro. Los colores del arcoiris son un buen caso de uso para una *enumeración*.
```haskell
data RainbowColors = Red | Orange | Yellow | Green | Blue | Indigo | Violet
```
Aquí, `RainbowColors` es el *constructor de tipos* (*type constructor*) y `Red`, `Orange`, etc. son los *constructores de datos* (*data constructors*).
La *especie* de RainbowColors es `*`.

Los posibles casos de `RainbowColors` están separados mediante el operador `|` (*adición*).

A continuación, un simple caso de uso.
```active haskell
data RainbowColor = Red | Orange | Yellow | Green | Blue | Indigo | Violet

colorToStr Red    = "Rojo"
colorToStr Orange = "Naranja"
colorToStr Yellow = "Amarillo"
colorToStr Green  = "Verde"
colorToStr Blue   = "Azul"
colorToStr Indigo = "Indigo"
colorToStr Violet = "Violeta"

main =
  do putStrLn (colorToString Red)
     putStrLn (colorToString Indigo)
```
### Constructores de orden superior ([polimorfismo](https://wiki.haskell.org/Polymorphism))
Un "*constructor de tipos de orden superior*" es un *constructor de tipos* con uno o más parámetros. Los *constructores de tipos de orden superior* pueden ser miembros de la especie `* -> *`, `(* -> *) -> *`, `* -> * -> *`, etc.; cualquier especie excepto `*`.

Un *constructor de tipos de orden superior* muy usado en Haskell es `Maybe` que sirve para denotar la posible presencia o ausencia de un valor. A continuación, crearemos nuestra propia definición con fines ilustrativos.
```haskell
data Maybe a = Just a | Nothing
```
El *constructor de tipos* `Maybe` es miembro de la especie `* -> *` que se puede leer como: "toma un tipo miembro de la especie `*` y produce un tipo miembro de la especie `*`". La `a` en la definicón de `Maybe` es una *variable de tipo* (*type variable*); a Maybe se le puede aplicar entonces un tipo (miembro de la especie `*`) que sustituya a la `a`. `Just a` se utiliza para expresar la presencia de un valor de tipo `a` y `Nothing` para expresar la ausencia.

> Al `Maybe` ser parte de "Prelude" (librería estándar de Haskell), no es necesario ni definirlo ni importarlo para usarlo; como se había dicho antes, es un constructor muy común. A continuación, un simple caso de uso. 
```active haskell
data RainbowColor = Red | Orange | Yellow | Green | Blue | Indigo | Violet
{-
No es necesario definir ni importar el constructor Maybe
data Maybe a = Just a | Nothing
-}

maybeColorToStr Nothing       = "Color ausente"
maybeColorToStr (Just Red)    = "Rojo"
maybeColorToStr (Just Orange) = "Naranja"
maybeColorToStr (Just Yellow) = "Amarillo"
maybeColorToStr (Just Green)  = "Verde"
maybeColorToStr (Just Blue)   = "Azul"
maybeColorToStr (Just Indigo) = "Indigo"
maybeColorToStr (Just Violet) = "Violeta"

main =
  do putStrLn (maybeColorToStr (Just Green))
     putStrLn (maybeColorToStr Nothing)
```
En este ejemplo puedes ver dos cosas. Primero, que el código ya no está quedando muy elegante; más adelante en este tutorial veremos más sintaxis básica de Haskell que nos permitirá reescribirlo de manera más elegante. Segundo, que usamos `RainbowColor` como argumento de `Maybe` implicitamente para obtener un tipo sencillo (miembro de la especie `*`), `Maybe RainbowColor`.
```haskell
-- La especie de los constructores
Maybe a            :: * -> *
Maybe RainbowColor :: *
-- El tipo de los datos
Just Red           :: Maybe RainbowColor
Nothing            :: Maybe RainbowColor
  -- `Nothing` no hace ninguna referencia a RainbowColor; es la inferencia
  -- de tipos del compilador lo que lo asocia a dicha enumeración.
```
### Multiplicación, adición y tipos recursivos
Ya que vimos los *constructores de tipos* "simples" y "de orden superior", veremos como los podemos combinar para hacer *constructores de tipos complejos*.
#### Multiplicación
Cuando un dato es formado por **n** datos, se modela mediante la **multiplicación** (o **unión**). Como ejemplo, modelaremos el concepto de *nombre completo* que consiste en dos datos: *nombre* y *apellido*.
```haskell
data FullName = FullName String String
billGates = FullName "William" "Gates"
```
> En este emjemplo, el *constructor de tipos* y el *constructor de datos* tienen el mismo nombre, **FullName**.

#### Adición
Cuando un dato es uno de **n** posibilidades, se utiliza la *suma*, que fue lo que ya utilizamos para modelar `RainbowColor` y `Maybe`.
```haskell
data RainbowColor = Red | Orange | Yellow | Green | Blue | Indigo | Violet

data Maybe a = Just a | Nothing
```
Las multiplicaciones y adiciones se pueden combinar:
```haskell
data Name = FullName String String | Nickname String
billGates = FullName "William" "Gates"
me        = Nickname "Lay"

-- both, billGates and me belong to the type Name
```
#### Tipos recursivos
Y los tipos pueden ser recursivos. Un ejemplo clásico es la modelación de un [árbol binario etiquetado](https://en.wikipedia.org/wiki/Binary_t+-+ree).

Como ejemplo, modelaremos [este árbol binario etiquetado](https://en.wikipedia.org/wiki/Binary_tree#/media/File:Binary_tree.svg)
```haskell
data LBTree a = InternalNode a (LBTree a) (LBTree a) | Leaf (Maybe a)

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
```
El *constructor de datos* **InternalNode** representa un nodo interno, aquel que no es un nodo hoja y es la unión de *tres* datos: el valor del nodo y sus dos hijos (izquierdo y derecho).
El *constructor de datos* **Leaf** representa un nodo hoja; dado que un nodo puede no tener un hijo izquierdo o un hijo derecho, el valor del nodo hoja es un `Maybe a`.
En el ejemplo, **a** es asociado al tipo `Int`. Que `LBTree a` sea parte de la definición de `InternalNode` y que `InternalNode` sea parte de la definición de `LBTree` hacen a `LBTree` un *constructor de tipos* recursivo.

> Desconozco la razón por la cual se utilizan los nombres **adición** y **multiplicación**. Probablemente tenga su origen en el cálculo y el álgebra [1](http://chris-taylor.github.io/blog/2013/02/10/the-algebra-of-algebraic-data-types/) o en teoría de categorías[2](https://en.wikipedia.org/wiki/Addition#Addition_in_set_theory_and_category_theory) o ambas[3](http://arxiv.org/pdf/math/0504565v1.pdf). Si alguien tiene una respuesta definitiva, mándenme un correo, por favor.

## Sintaxis básica de Haskell: Búsqueda de patrones
Ahora que tenemos un mejor entendimiento de los tipos y datos en Haskell, podemos hablar de lo que es la **búsqueda de patrones** (***pattern matching***).
La búsqueda de patrones consiste en usar "patrones" para encontrar la expresión adecuada a evaluar. Existen muchas formas de búsqueda de patrones en Haskell

### Patrón explícito
Este es el caso más sencillo de *búsqueda de patrones*. Consiste en usar valores en vez de patrones en la definición de las funciones. Este tipo de pattern matching ya lo hemos estado usando; el primer caso donde lo usamos fue en la definición de `f` del tutorial pasado.
```haskell
f first second True = first
f first second False = second
```
Cuando invocamos a `f` con los argumentos: `putStrLn "Hello"`, `putStrLn "Goodbye"` y `True`, es `True` (el tercer argumento) el que indica que es la primer definición la que hace "match" y por lo tanto se retorna `first`. Cuando invocamos a `f` con los argumentos: `putStrLn "Hello"`, `putStrLn "Goodbye"` y `False`, nuevamente es el tercer argumento el que hace match, pero esta vez con la segunda definición de `f` y por lo tanto se retorna `second`.

### "case .. of"
Otra manera en la que podemos definir `f` en una sola expresión es usando *case .. of*.
```active haskell
f first second condition =
  case condition of True -> first
                    False -> second

main = do
  f (putStrLn "Hello") (putStrLn "Goodbye") True
  f (putStrLn "Hello") (putStrLn "Goodbye") False
```

### Guardias
Otra manera en la que podemos definir `f` en una sola expresión es usando *guardias* (*guards*).
```active haskell
f first second condition | condition == True = first
                         | condition == False = second

main = do
  f (putStrLn "Hello") (putStrLn "Goodbye") True
  f (putStrLn "Hello") (putStrLn "Goodbye") False
```

### Comodines
Cuando un patrón no va a ser utilizado, conviene mostrar nuestra intención y no utilizar un nombre para dicho patrón. En su lugar, se puede usar un *comodín* (*wildcard*), `_`.
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

### Retomando maybeColorToStr
Ahora que ya sabemos hacer búsqueda de patrones, podemos reescribir `maybeColorToStr` de la siguiente manera:
```active haskell
data RainbowColor = Red | Orange | Yellow | Green | Blue | Indigo | Violet

maybeColorToStr Nothing  = "Color ausente"
maybeColorToStr (Just c) = case c of
                             Red -> "Rojo"
                             Orange -> "Naranja"
                             Yellow -> "Amarillo"
                             Green  -> "Verde"
                             Blue   -> "Azul"
                             Indigo -> "Indigo"
                             Violet -> "Violeta"

main =
  do putStrLn (maybeColorToStr (Just Green))
     putStrLn (maybeColorToStr Nothing)
```

## Ejercicios y soluciones

[Ejercicios](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-2/ejercicios)

[Soluciones](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-2/soluciones)

# Tercera parte
Cuando te sientas listo, continua con [la tercera parte](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-3/tutorial).