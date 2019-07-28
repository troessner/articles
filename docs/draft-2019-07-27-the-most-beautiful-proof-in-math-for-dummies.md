## The Most Beautiful Proof In Math For Dummies

![Prime Number Spiral](https://upload.wikimedia.org/wikipedia/commons/8/8e/Hexgrid_prime_number_sprial.svg)


Let's start with a bold statement: Physics, biology, chemistry, all of it is built on math. Math in turn is built on numbers and numbers are built on prime numbers. A prime number is a natural number greater than 1 that cannot be formed by multiplying two smaller natural numbers. A natural number greater than 1 that is not prime is called a composite number.

Examples for prime numbers are 2 (the only even prime number), 3, 5, 7, 11 and so on. An example of a composite number would be 6: 6 is a composite number because it can be expressed as the product of 2 and 3 (so two prime numbers). Let's look at some other examples of composite numbers and how they are constructred from prime numbers:

```
8 = 2 * 4 = 2 * 2 * 2
12 = 2 * 2 * 3
30 = 2 * 3 * 5
```

You get the idea.

Following this train of thought (even if you disagree) it seems that at the heart of everything there are prime numbers and that is one of many reasons why they have fascinated scientists (not just mathematicians) since the beginning of science and still continue do so. Particularly because there are still many, many fundamental and open questions that even after thousands years of research we can not answer. A good example of that would be the [Goldbach Conjecture](https://en.wikipedia.org/wiki/Goldbach's_conjecture) which is so easy to state that a child can understand it:

> Every even integer greater than 2 can be expressed as the sum of two primes.

Intuitively this makes sense:

```
4 = 2 + 2
6 = 3 + 3
8 = 3 + 5
```

And so on.

While the problem is easy to understand, a proof has eluded mathematician for centuries now with [Leonhard Euler](https://en.wikipedia.org/wiki/Leonhard_Euler) (probably the most productive and influential mathematician ever) even saying that this probably will never be proven.

Apart from math, prime numbers appear in all kinds of contexts of biology, chemistry, physics, cosmology, you name it so they seems to have a significance that goes far beyond pure math.

Now that we established why prime numbers are both so important and mysterious at the same time, what do we know about them? Well, for instance we know that there are infinitely many of them. It doesn't stop after 2, 3, 5, 7, 11...it goes on forever. How do we know this? Because this has been proven by [Euclid](https://en.wikipedia.org/wiki/Euclid) (often referred to as the "father of geometry") around 2300 years ago already in the arguably most beautiful proof in Math.

Since there wasn't much Math existing to build on top of for such proofs, almost all proofs from that time are quite simple and could in theory be understood by any layman. "Could" because they actually can't since for most mathematicians terseness in proofs was and is much more important than being easily digestable.

Let's take a look at the proof as you can find it online ([first google hit](https://primes.utm.edu/notes/proofs/infinite/euclids.html)):

> Theorem.
There are more primes than found in any finite list of primes.

>Proof.
Call the primes in our finite list p1, p2, ..., pr.  Let P be any common multiple of these primes plus one (for example, P = p1p2...pr+1).  Now P is either prime or it is not.  If it is prime, then P is a prime that was not in our list.  If P is not prime, then it is divisible by some prime, call it p.  Notice p can not be any of p1, p2, ..., pr, otherwise p would divide 1, which is impossible.  So this prime p is some prime that was not in our original list.  Either way, the original list was incomplete.

I dare you to find anybody in the street regardless of education that can understand this proof.

So let's do this proof in a way that any smart 10 year old can understand. But first things first - we have to establish some basic ideas before we can tackle the proof itself.

### Every Number That Divides Two Numbers Also Divides Their difference

Let's try to understand this intuitively by looking at an example:

Say we pick 18 as the first number and 6 as the second one. What divides both of them? 3 would be a good example here.

```
18 / 3 = 6
6 / 3 = 2
```

The results of that division (so 2 and 6) doesn't really interest us now, just that there *is a result*, so it is divisible.

What about the difference?

```
18 - 6 = 12
12 / 3 = 4
```

As you can see, 3 also divides the difference of 18 and 6.

Can we proof that this works for all numbers? Yes, we can.

Say we have two numbers called a and b (e.g. 18 and 6 in our example above). Both are divided by c (3 in our example). Call the result of that first division (18 / 3) m (corresponding to 6) and n the result of the second division (so 6 / 3 = 2).

Writing this down gives us two equations ("// indicates a comment"):

```
a = c * m // -> a fancy way of saying 18 = 3 * 6
b = c * n // -> 6 = 3 * 2
```

We can now do something that might seem odd to you but that is perfectly fine: We just subtract the whole second equation from the first one:

```
a - b = (c * m) - (c * n)
```

By simply applying the [Distributive Property](https://en.wikipedia.org/wiki/Distributive_property) we get:

```
a - b = c (m - n)
```

(m - n) is just another number. Following our example above with m as 6 and n as 2 this difference is 4.
Now we just need to read it out loud and we are done:

> a minus b is the same as c times some number

And thus, every number that divides two numbers also divides their difference.

### No Prime Number Divides 1

That one is quick and easy: 1 itself is not a prime number and the smallest prime number is 2. 1 divided by 2 doesn't yield an integer (but rather a fraction: 0.5), so 2 doesn't divide 1. The same is by extension valid for all numbers bigger than 2 and thus all prime numbers.

### The Fundamental Theorem Of Arithmetic

Quoting from [wikipedia](https://en.wikipedia.org/wiki/Fundamental_theorem_of_arithmetic)

> The fundamental theorem of arithmetic states that every integer greater than 1 either is a prime number itself or can be represented as the product of prime numbers and that, moreover, this representation is unique, up to (except for) the order of the factors.

Let's look at a simple example to hammer this idea home.

We'll start with 12. How can we write this in prime factors?

```
12 = 2 * 6 = 2 * (2 * 3)
```

Does the order matter? No. So

```
12 = 2 * 2 * 3 = 2 * 3 * 2 = 3 * 2 * 2
```

Now the important question: Can we write this in *any* other way as product? The answer is no, we can't. And that's exactly what the theorem of arithemtic is about.

### The Actual Proof

Ok, you made it this far and now we have everything we need to finally do this proof.

What we are going to do is a so called "reductio ad absurdum" proof. That means you assume the opposite of what you want to proof and then show that this will lead to a contradiction, thus proofing what you actually set out to proof.

We want to proof that there are infinitely many primes. The opposite of that is that there are finitely many primes. This means there is a finite list of all primes { 2, 3, 5, 7, ...., X }. We don't know what the last prime number is so I called it X. It doesn't really matter though what X is as you'll see as the proof progresses. Don't be scared of the brackets syntax {}, that is just the mathematical notation for a set: A bucket with different things in it. Numbers in this case.

We'll need to write down this list a bit more mathemical though by abstracting away the concrete numbers so it looks like this: { p<sub>1</sub>, p<sub>2</sub>, p<sub>3</sub>,... p<sub>n</sub> }. p<sub>1</sub> is the first prime number (so 2), p<sub>2</sub> the second (so 3) and so on. p<sub>n</sub> is the last one, X in my example above.

Now let's just multiply all of those numbers together which results in another number that we will call P:

P = p<sub>1</sub> * p<sub>2</sub> * p<sub>3</sub> * p<sub>n</sub>

and let's set another number called Q like this:

```
Q = P + 1
```

What kind of number is Q? Can it be a prime? No, it can't, because it is already greater than the list of all prime numbers itself. Imagine if {2, 3, 5} were the only prime numbers, their product P would be 30. Q would be 31 so it's obvious it can't be on that list.

Can it be a composite number? Remember the fundamental theorem of arithmetic above:

> every integer greater than 1 either is a prime number itself or can be represented as the product of prime numbers

We know P is a composite number (by the way we constructed it as product of all primes) and now we assume that Q is another composite number. This means that Q is a product of primes as well (just like P) and since we constructed P by multiplying all primes we have this means that there is a prime number, let's call it d, from the set of all prime numbers, that divides both P and Q.

Now we need to recall what we said in introduction above:

> Every number that divides two numbers also divides their difference.

The difference between Q and P is:

```
Q - P = 1
```

However we previously said that no prime number divides 1. Hence our whole assumption that Q is a composite number is wrong. It also can't be a prime. And now we have arrived at a contradiction. So our assumption that prime numbers are finite is wrong.

Hence the opposite is true: *Prime numbers are infinite*.
