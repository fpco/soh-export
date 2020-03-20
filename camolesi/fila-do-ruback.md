# Fila do ruback

Para que a solução seja válida, o elemento 1 só pode estar na posição 1 ou 2

- Se estiver na posição 1, o primeiro elemento está fixo e o número de soluções restantes é: nsol(n - 1)
- Se estiver na posição 2, o elemento 2 tem que estar na posição 1, então temos dois elementos fixos e o número de soluções restantes é nsol(n - 2)

Logo, nsol(n) = nsol(n - 1) + nsol(n - 2)

Vulgo fib(n + 1)

``` active haskell
import Data.List

valid l = all (<= 1) [abs (a - b) | (a, b) <- zip [1..] l]

bruteForce n = length . filter valid . permutations $ [1..n]

nsol 1 = 1
nsol 2 = 2
nsol n = nsol (n - 1) + nsol (n - 2) -- AKA Fibonacci

main = print [(bruteForce n, nsol n) | n <- [1..7]]
```