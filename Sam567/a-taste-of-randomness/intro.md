# Intro

#Foreword
The following tutorial series is based on my past and current research. As the name suggests, we will begin with a bite which will hopefully whet your appetite for more. 

A full appreciation of this mysterious subject will only come (in my personal experience) after a semester in stochastic computational physics, and if you stay with me through my Computational Physics series, we will eventually get there, but what we will be doing here is still a nice (I hope) overview.

#The Dream
Let us start with the vision that started it all. 

Imagine a mathematician who has been struggling with a problem for three months with no answer forthcoming. Furthermore, suppose the problem is something like: "Does there exist a *blabla* such that *blablabla* and *blablablabla* in dimension 444?"

Then, imagine the mathematician was to ask the question to his computer. The next day, the computer would say "Yes, there exists such a *blabla*; here is one such..."

Ok. By now, if you are not interested ... well, let us suppose you are indeed interested, we will now be a little more clear, and then I will show you how this is not science-fiction.


#The Problem Explained

If I had to give a name to the task set before me, I would call it: 
"Existential Proofs using Random Variables, and Computers."

This is the general idea, but we will eventually narrow it down to the specific proof I worked on, which is (this is here only for future reference) "The Hadamard conjecture."

In general, proving the existence of some mathematical object is hard. The traditional approach is to come up with a clever trick, and somehow construct such object.

In our case, 

we need to show that for dimensions `4k` where `k` is a natural number, there exists unitary matrices `u` such that each entry in the matrix is either `1/ sqrt (4*k)` or `-1/ sqrt (4*k)`. 

For instance, such matrix in dimension 2 looks like:

``` haskell
u =

0.71   0.71
0.71  -0.71
```
The unitary part means that the transpose of `u` multiplied by `u` gives the identity:

``` haskell

trans u <> u = ident n

where n is the dimension of u (number of rows).

```


#The Strategy Explained
In the past, many mathematicians have worked on specific cases of this problem (specific `k`s), and have found many combinatorial tricks to solve these specific cases. However, it doesn't become easier; no matter how many tricks are discovered, the next `k` is as hard if not harder.

We would like a general (and automated) way to attack the problem; this is what we propose: 

Supposing we can generate random matrices (more on that later) with any distribution we like, then the task becomes easy. 

We find a distribution for which Hadamard matrices are the most likely thing to occur, we generate tons and tons of matrices with this distribution, and surely enough, we will get Hadamard matrices! Once we found one, the proof is done. 

As you might appreciate, there are no tricks here (or a very big one, depending on how you look at it), so this method can be applied to a vast number of hard problems; turning them into something a computer can do.

#Conclusion: But does it work?
Most of my time went in developing the framework to deal with random variables in computers the same way we deal with them in mathematics (this will allow us to tackle many problems like the Hadamard Conjecture). 

However, this week, I have used this general framework to search for Hadamard matrices... and found some. Indeed, I found some in dimension 2, dimension 4, dimension 8, and dimension 12.

Right now, I'm refining the technique, making it more organic (not as in whole food; as in DNA), to see up to what dimension can I go on my laptop.

If you are interested, want more juicy details, want to prove your own deep mathematical conjecture using my framework, or have commentaries, suggestions, anything really ...

Feel free to leave me some Feedback.

By the way, this technique can also be applied to study the statistical properties of many systems. In fact, the technique is more adapted to this application.

Part of what makes this so interesting is that we don't really understand the limitations of these techniques, so to be able to implement these ideas and test them will tell us much.