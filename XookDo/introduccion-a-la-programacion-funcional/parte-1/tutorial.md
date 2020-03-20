# Un primer vistazo a Haskell

En este primer tutorial, conoceremos los principales beneficios de la programción funcional pura y un poco de la sintaxis básica de Haskell.

## Sintaxis básica de Haskell
### "Hello, World!"
En este punto, no debería de ser problema entender que un "Hello, World!" puede ser tan simple como:

``` active haskell
main = putStrLn "Hello, World!"
```
### "where"
Algo muy usado en Haskell para evitar definir muchas funciones auxiliares por todos lados es el uso del `where`. A continuación, usaremos `where`:
``` active haskell
binOpApp binOp = op

main =
  do
    putStrLn (show (binOpApp add 1 2))
    putStrLn (show (binOpApp sub 1 2))
    where add = (+)
          sub = (-)
```
### "let .. in"
Una notación alternativa es el uso de ***let .. in***:
``` active haskell
binOpApp binOp = binOp

main =
  let add = (+)
      sub = (-)
  in do
      putStrLn (show (binOpApp add 1 2))
      putStrLn (show (binOpApp sub 1 2))
```
Sin duda, la sintaxis de Haskell es elegante. Haskell tiene las fortalezas de un lenguaje fuertemente tipado y la claridad de algunos lenguajes dinámicos.

## Ejercicios y soluciones

- [Ejercicios](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-1/ejercicios)
- [Soluciones](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-1/soluciones)

# Segunda parte
Cuando te sientas listo, continua con [la segunda parte](https://www.fpcomplete.com/user/XookDo/introduccion-a-la-programacion-funcional/parte-2/tutorial).