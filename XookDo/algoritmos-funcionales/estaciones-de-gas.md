# Estaciones de Gasolina

> El problema original fue tomado de [aquí](http://www.programcreek.com/2014/03/leetcode-gas-station-java/). La codificación de la entrada fue modificada para no usar arreglos.

## Definición del problema
Se tiene un circuito de n estaciones de gasolina. Sólo se puede recorrer en una dirección, cada gasolinera tiene cierta gasolina almacenada, el tanque del auto que recorre el circuito tiene un tanque de tamaño infinito (le cabe toda la gasolina que se le quiera introducir). El problema es encontrar a partir de que gasolinera se puede iniciar el recorrida de tal forma que se pueda completar el circuito (el tanque nunca se quede vacio entre gasolineras); Si existe una solución, `solution` debe de regresar el `Id` de la gasolinera de la cual se puede iniciar el recorrida; si no existe una solución, `solution` debe de regresar `-1`.

Los circuitos son tales que tienen al menos dos estaciones de gasolina

La entrada del programa es `Circuit` el cual es una lista de gasolineras, más no un ciclo, es decir, es sólo una lista finita. (Para hacer un ciclo de una lista, se puede usar la función [`cycle`](http://hackage.haskell.org/package/base-4.8.1.0/docs/Prelude.html#v:cycle))

## Representación en Haskell
El problema se puede expresar de esta manera en Haskell:
```active haskell
import Test.QuickCheck
import Data.List

data GasStation = GS { gas::Int, distanceToNext::Int } deriving Show
newtype Circuit = Circuit [GasStation] deriving Show

solution :: Circuit -> Int
solution = undefined            -- solution is to be done

-- it creates the trip to traverse for the list l starting at the nth gas station
trip l nth = take (length l).drop nth.cycle $ l

naiveSolution :: Circuit -> Int
naiveSolution (Circuit c) =
  maybe (-1) fst $ find snd tries
  where traverse = scanl (\tank (GS gas cost) -> tank + gas - cost) 0
        isSolution = all (>= 0) . traverse
        tries = map (\i -> (i, isSolution (trip c i))) [0..length c - 1]

-------------------- Tests --------------------
instance Arbitrary GasStation where
  arbitrary =
    do
      gas' <- arbitrary
      distanceToNext' <- arbitrary
      return $ GS (abs gas') (abs distanceToNext')

-- A circuit must have at least two gas stations
instance Arbitrary Circuit where
  arbitrary =
    do
      tail <- arbitrary
      gs1  <- arbitrary
      gs2  <- arbitrary
      return $ Circuit ([gs1,gs2] ++ tail)

-- given a [GasStation], it returns a list of net costs of traversing from a gas station
-- to its next gas station
netCost = map (\(GS g d) -> g - d)

-- if there is a solution, then, when traversing the whole circuit from the solution,
-- the tank never reaches a negative quantity of gas
positive_case c@(Circuit c') =
  let result = naiveSolution c
  in result /= -1 ==> isASuccess $ trip c' result
  where isASuccess = all (>= 0).scanl (+) 0.netCost

-- If all possible traversals have a tank with negative gas at some point, then there is no solution
negative_case c@(Circuit c') =
  let result = naiveSolution c
  in result == -1 ==> all id failureOfEachPossibility
  where isFailure = any (< 0).scanl (+) 0.netCost
        failureOfEachPossibility = map (isFailure . trip c') [0..length c' - 1]

-- Our final solution should output the same results as the naive implementation
solution_eqs_naiveSolution c =
  solution c == naiveSolution c

main =
  do quickCheck positive_case
     quickCheck negative_case
     quickCheck solution_eqs_naiveSolution
```

Hemos definido `naiveSolution` como parte de la especificación del problema. `naiveSolution` es una traducción obvia de la definición del problema del español a Haskell, pero su complejidad es O(n^2) en cuanto a tiempo de ejecución. También definimos dos pruebas, una para el caso en el que sí hay solución, `positive_case`, y otra para el caso en el que no hay solución, `negative_case`. Para poder correr las pruebas sobre la especificación, tuvimos que hacer a `GasStation` una instancia de `Arbitrary` (para mas información sobre pruebas en Haskell, visita [esta liga](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-8/tutorial). **Usar QuickCheck ayudó a encontrar bugs incluso para la implementación del caso trivial**.

Compilar y ejecutar el código en la línea de comandos se ve de la siguinete manera:
![c1](https://www.fpcomplete.com/media/da3c6120-4a33-4fec-bff3-20abdb3478b6.png)

Por ahora, la propiedad `solution_eqs_naiveSolution` falla, pues no hemos implementado la solución.

## Análisis del problema
Existen varias cosas que uno debe notar para llegar a la solución óptima, que es O(n) en tiempo de ejecución y O(1) en memoria de trabajo. La solución ingenua (`naiveSolution`) igual es O(1) en espacio de trabajo, pero O(n^2) en tiempo de ejecución; no obstante, nos servirá para verificar que nuestra solución final es correcta.

Para ayudarnos con el análisis, describiremos, más no implementaremos, las siguientes funciones:

- `next g_x` es la gasolinera que le sigue a la gasolinera `g_x` en el circuito.
- `kthGasStation g_x k` es la `kth` gasolinera que le sigue a `x` en el circuito.
- `acum g_x g_y` es la gasolina acumulada en el tanque si iniciáramos el recorrido en `g_x` y llegáramos y cargáramos en la gasolinera `g_y`.
- `stoppedAt g_x` es la última estación de gasolina a la que se puede llegar partiendo desde `g_x`, es decir, `stoppedAt g_x` es la primer gasolinera partiendo de `g_x` tal que se cumple esta desigualdad: `distanceToNext (kthGasStation g_x (stoppedAt g_x)) > acum g_x (stoppedAt g_x)`.
- `failedAt g_x` es `(next.stoppedAt) g_x`; es decir, la primer gasolinera desde `g_x` tal que `acum g_x (failed g_x)` es negativo.
- `dist g_x g_y` es la cantidad de gasolineras entre `g_x` y `g_y` incluyendo a `g_x` y a `g_y`.

### Lema 1
**Si `stoppedAt g_x == g_y`, entonces no tiene caso intentar iniciar el recorrido desde una gasolinera ubicada entre `g_x` y `g_y`, por lo que se debe de intentar desde `next g_y`.**
Supongamos que el circuito tiene estas gasolineras: `g_1 -> g_2 -> ... -> g_t-1 -> g_t`; y que `stoppedAt g_1 == g_t-1`, eso quiere decir que `acum g_1 g_2` es un número `>= 0` o no hubiésemos podido llegar a `g_t-1`, por lo que no tiene sentido intentar iniciar desde `g_2`, pues sólo nos estaríamos perdiendo de `acum g_1 g_2` y por lo tanto empeoraríamos la posibilidad de llegar más allá de `g_t-1`. El mismo razonamiento aplica para cada prefijo del tramo entre `g_1` y `g_t-1`, por lo que sólo tiene sentido tratar desde `g_t`; es decir, si `stoppedAt g_x` está definido para `g_x`, entonces la siguiente gasolinera de la cual hace sentido intentar iniciar el circuito es `(next.stoppedAt) g_x`.

### Lema 2
**Si hay suficiente gasolina para darle la vuelta al circuito, entonces existe una solución.**
Para probar esto, probaremos la contrapositiva, que dice: si no existe una solución, entonces no hay suficiente gasolina para darle la vuelta al circuito.

Dado que estamos asumiendo que no existe una solución (debido a que estamos probando la contrapositiva), `failed g_x` tiene que estar definida para toda `g_x`.

Podemos definir una función `acumByFailure` para todo circuito que no tiene solución (pues depende de `failedAt`). Esta función regresa la gasolina neta de recorrer el circuito y siempre da un numero negativo.

```haskell
acumByFailure gs = go (cycle gs) (length gs) 0
  where go _ 0 tank = tank
        go c l tank = go drop' (l-1) (tank + acum')
        where g_x = head c
              g_fail = failedAt g_x
              acum' = acum g_x g_fail
              drop' = drop (dist g_x g_fail) c
```
Lo que `acumByFailure` hace es darle al menos una vuelta al circuito y acumular en `tank` la gasolina neta. Dado que la acumulación es mediante `tank + acum'` y `acum'` siempre es negativa (dado que `acum' = acum g_x g_fail)`), entonces `acumByFailure` siempre regresa un número negativo.

Podemos entonces concluir que si el circuito no tiene solución, la función `failedAt g_x` está definida para cada `g_x` y por lo tanto la función `acumByFailure` está definida para dicho circuito y por lo tanto no existe suficiente gasolina para darle la vuelta al circuito, pues `acumByFailure` siempre regresa un número negativo. Y por su contrapositiva, si hay suficiente gasolina para darle la vuelta al circuito, entonces existe una solución.

### Lema 3
**Si existe una solución, entonces, dado un circuito, basta con encontrar donde empezar en el circuito tal que se pueda llegar al final del circuito sin tener que dar la vuelta completa.**

Dicho en otras palabras, el lema podría decir: Supongamos que tenemos un circuito: `g_1 -> ... -> g_n -> g_1`, entonces, si existe una solución, basta con encontrar una gasolinera entre `g_1` y `g_n` (incluyéndolas) que al iniciar desde esta el recorrido, se pueda llegar a `g_n`.

Para probarlo, asumimos que hay solución y suponemos que una gasolinera `g_k` es la primera de donde se puede iniciar el recorrido y llegar a `g_n`. ¿Cómo sabemos que podremos llegar a `g_k` de regreso? Si `acum g_k g_n` no fuese suficiente para llegar de regreso a `g_k`, entonces `g_k` no podría ser la solución y por el lema 1, menos podría serlo una estación entre `g_k` y `g_n`, por lo que tendría que ser una anterior a `g_k` (pues una posterior a `g_n` sería volver a empesar (`g_1`, `g_2`, etc)), pero dado que `g_k` fue la primera en poder llegar a `g_n`, ninguna anterior a `g_k` puede serlo, por lo que o es `g_k` la solución o no hay solución y dado que estamos asumiendo que sí hay solución, `g_k` tiene que ser la solución.

### Uniendo los lemas
El lema 3 ya depende del lema 1 en la frase "... y por el lema 1, menos podría serlo una estación entre `g_k` y `g_1`...". Pero el lema 3 se puede reformular usando el lema 2 de la siguiente manera:
** Si hay suficiente gasolina para darle la vuelta al circuito (si existe una solución, según el lema 2), entonces, dado un circuito, basta con encontrar donde empezar en el circuito tal que se pueda llegar al final del circuito sin tener que dar la vuelta desde donde se comenzó; Si `stoppedAt g_x == g_y`, entonces no tiene caso intentar iniciar el recorrido desde una gasolinera ubicada entre `g_x` y `g_y`, por lo que se debe de intentar desde `next g_y`.**
Por lo que nuestro algoritmo entonces solo tiene que encontrar a partir de cual gasolinera se puede llegar a la úlitma, calcular la gasolina neta (`acum g_1 g_n`) y si la gasolina neta es `>= 0` entonces dicha gasolinera es la solución, de lo contrario, no hay solución y para hacerlo más rápido, cada vez que una gasolinera falle, podemos reiniciar la búsqueda desde `next.stoppedAt $ g_x`.

## Solución final
Ahora podemos expresar nuestra solución en Haskell:
```active haskell
import Test.QuickCheck
import Data.List

data GasStation = GS { gas::Int, distanceToNext::Int } deriving Show
newtype Circuit = Circuit [GasStation] deriving Show

-- given a [GasStation], it returns a list of net costs of traversing from a gas station
-- to its next gas station
netCost = map (\(GS g d) -> g - d)

solution :: Circuit -> Int
{-hi-}solution{-/hi-} (Circuit circuit) =
  let c'            = netCost circuit
      (solution,_,_,acum) = foldl next (0,0,0,0) c'
  in {-hi-}if acum < 0 then -1 else solution{-/hi-}
  where next (tentativeSolution, currentGS, tank, totalNetGas) nextNetGas                     
          | tank + nextNetGas < 0 = ({-hi-}currentGS + 1{-/hi-},   currentGS + 1, 0,                 acum')                
          | otherwise             = (tentativeSolution, currentGS + 1, tank + nextNetGas, acum')      
          where acum' = totalNetGas + nextNetGas
-- At `if acum < 0 then -1 else solution`, it can be seen how if there is enough gas to go around
-- the circuit, then there is a solution.
-- At `currentGs + 1`, it can be seen how we skip all the stations that have been traversed each time
-- the tank doesn't have enough gas to reach the next station (`tank + nextNetGas < 0`).
  

-- it creates the trip to traverse for the list l starting at the nth gas station
trip l nth = take (length l).drop nth.cycle $ l

naiveSolution :: Circuit -> Int
naiveSolution (Circuit c) =
  maybe (-1) fst $ find snd tries
  where traverse = scanl (\tank (GS gas cost) -> tank + gas - cost) 0
        isSolution = all (>= 0) . traverse
        tries = map (\i -> (i, isSolution (trip c i))) [0..length c - 1]

-------------------- Tests --------------------
instance Arbitrary GasStation where
  arbitrary =
    do
      gas' <- arbitrary
      distanceToNext' <- arbitrary
      return $ GS (abs gas') (abs distanceToNext')

-- A circuit must have at least two gas stations
instance Arbitrary Circuit where
  arbitrary =
    do
      tail <- arbitrary
      gs1  <- arbitrary
      gs2  <- arbitrary
      return $ Circuit ([gs1,gs2] ++ tail)

-- if there is a solution, then, when traversing the whole circuit from the solution,
-- the tank never reaches a negative quantity of gas
positive_case c@(Circuit c') =
  let result = naiveSolution c
  in result /= -1 ==> isASuccess $ trip c' result
  where isASuccess = all (>= 0).scanl (+) 0.netCost

-- If all possible traversals have a tank with negative gas at some point, then there is no solution
negative_case c@(Circuit c') =
  let result = naiveSolution c
  in result == -1 ==> all id failureOfEachPossibility
  where isFailure = any (< 0).scanl (+) 0.netCost
        failureOfEachPossibility = map (isFailure . trip c') [0..length c' - 1]

-- If there is enough gas for completing the circuit, then there has to be a solution
{-hi-}enoughGas_impliesSolution{-/hi-} c@(Circuit c') =
  (foldl (+) 0.netCost) c' >= 0 ==> naiveSolution c /= -1

-- Our final solution should output the same results as the naive implementation
solution_eqs_naiveSolution c =
  solution c == naiveSolution c

main =
  do quickCheck positive_case
     quickCheck negative_case
     quickCheck enoughGas_impliesSolution
     quickCheck solution_eqs_naiveSolution
```
Habiendo encontrado el segundo lema útil para verificar que nuestros algoritmos son correctos, lo uncluimos en las pruebas como `enoughGas_impliesSolution`.
Ahora que hemos implementado `solution`, todas las pruebas pasan, incluyendo en la que decimos que nuestra solución final es extensionalmente equivalente a la solución ingenua:
![c2.png](https://www.fpcomplete.com/media/26039e06-6f23-4dd5-9f84-1a74878aaa19.png)

Dado que nuestra solución consiste en un `foldl next (-1,0,0,0) c`, donde `next` realiza sólo un número constante de operaciones, y que después del `foldl` sólo se realiza un numero constante de operaciones, entonces la solución es O(n) en tiempo de ejecución. Dado que no usamos ningún vector o arreglo o estructura recursiva para el espacio de trabajo, la solución es O(1) en espacio de trabajo.