Title: Dependency Tracing
Date: 2020-12-04
Modified: 2020-12-04
Category: programming languages
Tags: programming languages
Slug: dependency-tracking
Authors: Peter Stefek
Summary: Looking for arguments that make a difference 

**Background**  
A few weeks ago, I was talking to Jemma Issroff, a fellow Recurser, about a toy programming language she was building. We ended up thinking about automatically memoizing pure functions based on their parameters. As we thought about this problem more and more, we decided it would be interesting to be able to tell which parameters of function actually required to compute it's result at runtime. The idea at the time was that we could cache just based on those parameters.

**Problem Statement**  
Let's say we have a function $f$ that takes arguments $x_1,...,x_n$ and outputs some result $res$. We want to check at runtime which of those arguments are really necessary to compute $res$. Let's take a very simple example:  
```
let f = fn(x, y, z) {
  return y
}
```  
Now if we call $f(1,2,3)$, it's pretty clear that the result $2$ only depends on $2$, the value of the second argument. Of course we could have figured that out at compile time (and in fact most compilers will yell at you to tell you $x$ and $z$ are unused).
But now let's take a slightly more interesting case:  
```
let f = fn(x, y, z) {
  if (x > 0) {
    return y
  } else {
    return z
  }
}
```  
Unlike the first example, clearly all the variables are used here. However, if we evaluate $f(1,2,3)$ the output $2$ only depends on the values of $x$ and $y$.   

Let's stop here for a second and unpack that statement a little. Why am I saying the return value of $f(1,2,3)$ depends on both $x$ and $y$ even though the return value is exactly equal to the just value of $y$?  

The answer is because if the value of x were to change in a certain way (for example from $1$ to $-1$), our return value would change.   

Conversely, the return value of $f(-1,2,3)$ depends on $x$ and $z$.   

**Definition of Dependency**  
So far, we've been talking about what dependence means informally. For posterity's sake, I'm going to give a more concrete definition. We can say that expression $a$ depends on expression $b$ iff in order to compute the value of $a$ the value of $b$ must first be computed.  

**Rules of Dependency Propagation**  
Now that we have an intuitive understanding of how to determine which dependencies a function needs, let's try to harden this intuition into a concrete algorithm.  

First we're going to quickly introduce one more operator $+$.   
```
let f = fn(x,y) {
  return x + y
}
```  
The dependencies of any call to this function, for example $f(1,2)$, are simply x and y.
But now let's look at a more complicated example:  
```
Let f = fn(x, y, z) {
  let s = z + y
  if (s < 0) {
    return x
  } else {
    return s
  }
}
```  
This example isn't too much more complicated than any of the previous ones. A quick visual analysis will allow us to determine $f(1,2,3)$ will give a result of $5$ and will depend on $y$ and $z$. It will also allow us to determine, $f(1, 2, -3)$ will give a result of $1$ and depend on $x$, $y$ and $z$. However, there is a more systematic way we can evaluate the dependencies (you're probably even using it in your head).   

Let's create two rules, one for "+" and one for "if". Let's pretend x, y, z are generic expressions and deps(e) of an expression $e$ gives the set of all of its dependencies.  
Here are our two rules:  
 
- $let\,res = x + y;$ $deps(res) = deps(x) \cup deps(y)$ (where $\cup$ is the union of two sets)  
- $let\,res = if\,(x\,{ y }\,else\,{ z };$ if x is true then $deps(res) = deps(x) \cup deps(y)$ else $deps(res) = deps(x) \cup deps(z)$  

Combining these two rules we can trace the dependencies of any function made up of only additions and if statements! While that's not really mind blowing, I found it kind of interesting.  

Now if you are a battle hardened senior software engineer, you probably know that some of the most advanced programs powering cutting edge tech companies like Google and Facebook will occasionally use more operators than just "+" and "if".  

So how do we handle other operations? Simple! Create more rules!    

**Arithmetic**  
Unlike in grade school, here arithmetic is simple. Almost all arithmetic operators follow the exact same rules as "+" (except multiplication).  

**Short Circuit Operators**  
Okay now let's do multiplication. Imagine you have the following statement:  
```
let f = fn(m, a, n, y, a, r, g, s) {
  return m * a * n * y * a * r * g * s
}
```  
A call like $f(1, 2, 3, 4, 5, 6, 7, 8)$ depends on every argument passed the f. However, what about this call $f(0, 2, 3, 4, 5, 6, 7, 8)$? Notice that the only variable which really affects the output of the product is $m$ with value 0. As long as the value of $m$ remains 0, the values of the result will be 0 no matter what value any other variable takes. This idea is conceptually similar to [short circuit evaluation for boolean values](https://en.wikipedia.org/wiki/Short-circuit_evaluation).   

Now you might ask what if there are multiple values that are all zero $f(0, 2, 3, 0, 5, 6, 7, 0)$, which one should we choose to depend on?   
There are a couple options here:  

- Choose the first available zero. This matches up with classic short circuit evaluation.
- Choose the zero with the fewest dependencies. Choosing this option is less efficient but will ensure that our deps function returns the smallest set of dependencies possible.  

This optimization can also be applied to boolean functions such as && and ||.   

Short Circuit Rules (for the Classic Short Circuit Strategy):  

- $let\,res = x_0\,*... *\,x_n;$ If no expression $x_i$ is zero, $deps(res) = \bigcup_{i=0} deps(x_i)$.        Otherwise $let\,i =argmin_{i}\{x_i == 0\}, deps(res) = deps(x_i)$.

- $let\,res = x_0\,\&\&\,...\,\&\&\,x_n;$ If no expression $x_i$ is false, $deps(res) = \bigcup_{i=0}\,deps(x_i)$.   
Otherwise $let\,i = argmin_{i}\{x_i == false\}, deps(res) = deps(x_i)$ 

- $let\,res = x_0\,*... *\,x_n;$ If no expression $x_i$ is true, $deps(res) = \bigcup_{i=0} deps(x_i)$. Otherwise $let\,i = argmin_{i}\{x_i == true\}, deps(res) = deps(x_i)$

**Functions Calls**  
Here's a slightly trickier case:
```
let g = fn(x, y, z) {
  if (x > 0) {
    return y
  } else {
    return z
  }
}
let f = fn(x, y, z) {
 return g(z,x,y)
}
```
The above example represents a function call within our function call. In order to define this rule we need a quick definitions:

Let's define a function called $argdeps(f, a_0, a_1,...,a_n)$. $argdeps$ takes a function $f$ and its arguments $a_0,...,a_n$ and returns indices of the arguments which are needed to compute the corresponding value of $f$. Under the hood, this function is computed in the exact same way that we have been computing deps.

Let's define f to be a function with n arguments, and x_0,...,x_n to be n expressions.

Now we can make our rule. $let\,res = f(x_0,...,x_n);$  $deps(f) = U_{i = argdeps(f,\,x_0,...,\,x_n)} deps(x_i)$

**Lists**  
Now that we're feeling warmed up we can tackle one of the more complex rules sets.  

Let's take a quick look at a function involving lists.  
```
let f = fn(a) {
  return a[0] + a[2]
}
```

Now fellow travellers we have come to a fork in the road. To one side, lies the simple path on which we treat the entire list $a$ as a single unit. We would then say that the output of $f$ merely depends on a. To the other side lies the difficult path, to treat each element of $a$ as its own unit. 

Many would say, we can choose the best route to take based on tradeoffs and product requirements. Normally, I would agree that taking the simple path is a totally reasonable option. However on this particular journey, I believe we don't really have a choice, we have a responsibility.

Let's look back at our example from above:
```
let f = fn(a) {
  return a[0] + a[2]
}
```

The dependencies of this function are $a[0]$ and $a[2]$. This new notation is very simple but important, especially when talking about multi dimensional arrays. One subtle note is that this notation allows us to specify different levels of granularity. $a[0]$ for example, could be a single element or an entire array. I have also not actually defined what the set $deps(a[0])$ is yet. Before we actually get there, it will be useful to look at a few more examples.
```
let f = fn(a) {
  return a
}
```
This example is incredibly simple but important to look at. Let's look at $f([1,2,3])$. So it might seem like $f([1,2,3])$ just depends on the expressions at $a[0], a[1], a[2]$, but is that really correct?

The Recomputablity Test
Here's a way to test our intuition:

Say we call a function $f$ once with args $x_0,...,x_n$ giving us the resulting value $r_0$ and the resulting set of dependencies $D_0$. Then we call it again with arguments $y_0,...y_n$ and get the result $r_1$. If all of the values of the dependencies given by $D_0$ are the same between the two calls, the results $r_0$ and $r_1$ should be the same. (Note this test only tells us if a set of dependencies is not valid, not that it's valid)

Let's try using this test. $f([1,2,3])$ gives is the resulting array $[1,2,3]$ and prospective dependency set $D_0$ = $a[0], a[1], a[2]$ with corresponding values $1,2,3$.

Now what if we call $f([1,2,3,4])$? All of the values corresponding to $D_0$ remain unchanged so the value of $f([1,2,3,4])$ must be $[1,2,3]$ right? It's easy to see that this is wrong, so $a[0], a[1], a[2]$ cannot be our only dependencies.

Length Dependencies
One way to fix our problem is by taking a dependency on the length of a as well. We will use $lendeps(a)$ to denote the set of the expressions which the length of a depends on.

So one rule which sums up what we've been saying is:  
$let res = a;$ (where a is an list) $deps(res) = \big(\bigcup_{i=0} deps(x_i)\big) \cup {lendeps(a)}$ 

Since res is also an array we must also define its length dependencies with a second rule.

$lendeps(res) = lendeps(a)$

**List Concatenation**   
Now let's talk about concatenation:

```
let f = fn(a, b) {
  return a + b
}
```
If a and b are two lists, the resulting third list depends on both a, b, and their lengths. Let's define $res = a + b$. The rules for computing deps(res) and lendeps(res) are as follows:

- $deps(a+b) = deps(a) \cup deps(b) \cup lendeps(a) \cup lendeps(b)$  
- $lendeps(res) = lendeps(a) \cup lendeps(b)$

But we are not actually quite done yet. Check out the function below:

```
let f = fn(a, b) {
  let c = b + a
  return c[3]
}
```

This function returns the 4th element of the concatenated list $a + b$. So $f([1,2], [3,4,5])$ would return 1 and $f([1,2],[3,4,5,6])$ would return $6$.

Right now let's focus on the case of $f([1,2],[3,4,5,6])$. The result depends on the 4th element of $c$ which is the 4th element of $a$. Luckily the dependency set is simple here. It's just the 4th element of a. 

However if we look at the next case $f([1,2], [3,4,5])$, the dependencies are a little more complicated. The result of this function does not just depend on a[0]. Can you see why? 



What happens if we increase the size of b, for example by calling $f([1,2], [3,4,5,6])$? The answer is definitely no longer 1 so using the Recomputablity Test from before we can tell that the dependencies of $f([1,2], [3,4,5])$ cannot just be $a[0]$. You may already see where this is going, $f([1,2], [3,4,5])$ must depend on both $a[0]$ and $len(b)$. 

Here we are again faced with a choice. We can either say that all elements in a list depend on their value and the length of the list or we can break it down more granularly. Granularity costs time and memory so it's a legitimate tradeoff. I choose to go down the more granular road (in the interest of keeping the dependency set smaller) but I will write out the rules for both versions:


Recall above we defined res = a + b with two lists a and b, then made a rule to define deps(res) and lendeps(res). So here's a third rule which is recursively applied to all elements e of $res$.
(non granular version, larger dependency set) let x be the child of a or b which corresponds to e, deps(e) = deps(x) U lendeps(a)
(granular version) 
let x be the child of a which corresponds to e, deps(e) = deps(x)
let x be the child of b which corresponds to e, deps(e) = deps(x) U lendeps(a)

A quick note about the computation feasibility of this rule. This rule is applied recursively to every element in a and b so it is very computational intensive when you have large multidimensional arrays. I'd recommend using some kind of lazy evaluation for this rule in practice to avoid having to compute it for every element when unnecessary. 

Can I try this out?
You may have noticed that all the code in this article looked a little strange. That's because it is all written the custom programming Jemma has been building. The language supports all of the dependency tracking operations I talked about above and if you want to play around with it yourself!

Here are some examples:
