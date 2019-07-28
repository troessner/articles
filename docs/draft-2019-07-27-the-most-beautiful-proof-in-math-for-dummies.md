## The Most Beautiful Proof In Math For Dummies

![Prime Number Spiral](https://upload.wikimedia.org/wikipedia/commons/8/8e/Hexgrid_prime_number_sprial.svg)

Let's start with a bold statement: Physics, biology, chemistry, all of it is built on math. Math in turn is built on numbers and numbers are built on prime numbers - integers that can't be divided by any other number (technically they can be divided by themselves and 1 but that's not really relevant to us now). Any integer is either a prime number like 2 (the only even prime number), 3, 5 and so on or a composite number, that is an integer than can be expressed as the product of prime numbers. 6 is a composite number because it can be expressed as the product of 2 and 3 (so two prime numbers), 12 is the product of 2 times 2 and 3 (note that we can use prime numbers more than once in the product) and 30 is the product of 2, 3 and 5 (so 3 prime numbers). You get the idea.

So following this train of thought (even if you disagree) it's fairly obvious that at the heart of everything there are prime numbers and that is one of many reasons why they have fascinated scientists (not just mathematicians) and still do so. Particularly because there are still many, many fundamental and open questions that even after thousands years of research we can not answer. A good example of that would be the [Goldbach Conjecture](https://en.wikipedia.org/wiki/Goldbach's_conjecture) which is so easy to state that a child can understand it:

> Every even integer greater than 2 can be expressed as the sum of two primes.

While the problem is easy to understand, a proof has eluded mathematician for centuries now with [Leonhard Euler](https://en.wikipedia.org/wiki/Leonhard_Euler) (probably the most productive and influential mathematician ever) even saying that this probably will never be proven.

Now that we established why prime numbers are both so important and mysterious at the same time, what do we know about them? Well, for instance we know that there are infinitely many of them. It doesn't stop after 2, 3, 5, 7, 11...it goes on forever. How do we know this? Because this has been proven by [Euclid](https://en.wikipedia.org/wiki/Euclid) (often referred to as the "father of geometry") around 2300 years ago already in the arguably most beautiful proof in Math.

Since there wasn't much Math existing to build on top for such proofs, almost all proofs from that time are quite simple and could in theory be understood by any layman. "Could" because they actually can't since for most mathematicians terseness was and is much more important than being easily digestable.

Let's take a look at the proof as you can find it online ([first google hit](https://primes.utm.edu/notes/proofs/infinite/euclids.html)):

> Theorem.
There are more primes than found in any finite list of primes.

>Proof.
Call the primes in our finite list p1, p2, ..., pr.  Let P be any common multiple of these primes plus one (for example, P = p1p2...pr+1).  Now P is either prime or it is not.  If it is prime, then P is a prime that was not in our list.  If P is not prime, then it is divisible by some prime, call it p.  Notice p can not be any of p1, p2, ..., pr, otherwise p would divide 1, which is impossible.  So this prime p is some prime that was not in our original list.  Either way, the original list was incomplete.

I dare you to find anybody in the street regardless of education that can understand this proof.

So let's do this proof in a way that any smart 10 year old can understand, shall we?

But first things first - we have to establish some basic ideas before we can do this.

### Every Number That Divides Two Numbers Also Divides Their difference

Let's look at this intuitively with some example.

Say we take 3, 6 and 12. 3 divides 6 (it's 2) and 12 (it's 4). 12 minus 6 is ... 6 and we just saw that 3 divides 6. A bit more complex example: 5, 15 and 35. 5 divides 15 and 35 but this time the difference between 35 and 15 is a different number, 20. However 5 also divides 20 (it's 4) so we seem to be on the right track here.

Can we proof that this works for all numbers? Yes, we can.

Say we have two numbers called a and b (e.g. 15 and 35 in our example above). Both are divided by c (following our example that would be the 5). a (15) divided by c (5) is called m (that would be 3) and b (35) divided by c (5) is called n (that would be 7).

Just writing this down gives us two equations:

[1] a = c * m
[2] b = c * n

We can now do something that might seem odd to you but that is perfectly fine: We just subtract the whole second equation from the first one. So [1] - [2]:

a - b = (c * m)  - (c * n)

By simply applying the [Distributive Property](https://en.wikipedia.org/wiki/Distributive_property) we get:

a - b = c (m - n)

m - n is just another number. Following our example above 7 and 3, so 4.
Now we just need to read it out loud and we are done: "a minus b is the same as c times some number" and thus, every number that divides two numbers also divides their difference.

### No Prime Number Divides 1

That one is quick and easy: 1 itself is not a prime number and the smallest prime number is 2. 1 divided by 2 doesn't yield an integer (but rather a fraction: 0.5), so 2 doesn't divide 1. The same is by extension valid for all numbers bigger than 2 and thus all prime numbers.

### The Actual Proof
