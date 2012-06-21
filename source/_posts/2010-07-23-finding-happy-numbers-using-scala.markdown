---
layout: post
title: "Finding Happy Numbers using Scala"
date: 2010-07-23 19:48
comments: true
categories: scala
---

The problem was posted on [Programming Praxis](http://programmingpraxis.com/2010/07/23/happy-numbers/). The algorithm itself is pretty straightforward, anyone can do it with a few if/else/fors, but to coerce myself to think functionally, I decide to practice writing it in [Scala](http://www.scala-lang.org).

A number is a happy number if the sum of square of its digits eventually arrive at 1. For example, 7=&gt;7<sup>2</sup>=49=&gt;4<sup>2</sup>+9<sup>2</sup>=97=&gt;9<sup>2</sup>+7<sup>2</sup>=&gt;130=1<sup>2</sup>+3<sup>2</sup>+0<sup>2</sup>=10=&gt;1<sup>2</sup>+0<sup>2</sup>=1, so 7 is a happy number. 17 is not a happy number because by applying the above process, it goes into a loop.

Step1: calculating the sum of squares of a number

To get a list of numbers from a given number, we can first convert the number into a string, and then map every character of the string to its corresponding integer value. A more mathematical way is to divide the number continually by 10 until the original number becomes 0, adding the remainder to a list each time...The first method is easier to visualize, so here it goes:
`n.toString.toCharArray.map{digit=>Integer.valueOf(""+digit)}`

It's quite wordy but I've yet to find a better way. I'm pretty sure there is so if you know, do inform me.

Now I need to square each item in the list. So modify the above statement:
`n.toString.toCharArray.map{digit=&gt;Math.pow(Integer.valueOf(""+digit).doubleValue,2.0)}`

One thing bothers me is that I have to explicitly call `someInt.doubleValue`...I thought Scala does the implicit conversion for me? Then I realized `Integer.valueOf(...)` gives me `java.lang.Integer`, not `scala.lang.Int`. So I have to write a implicit conversion function myself:

```scala
implicit def integer2double(i:Integer):Double = i.doubleValue
```

Now I can get rid of the hideous `someInt.doubleValue`.

Since we're doing implicit conversion already, why not just implicitly convert a numeric character to a double so that it can be accepted by `Math.pow`?

```scala
implicit def char2double(ch:Char):Double = Integer.valueOf("" + ch).doubleValue
```

Now the code can be shortened to

```scala
n.toString.toCharArray.map { digit =&gt; Math.pow(digit, 2) }
```

Isn't that sweet? Implicit conversion is cool but it's easy to get carried away and do everything implicit, which makes the code hard to maintain, so there gotta be a balance somewhere. In the scope of this small exercise, I guess it's OK to use it.

Now that we have the squares of individual digits in a list, we can calculate the sum by reducing or folding it:

```scala
((n.toString.toCharArray.map { digit =&gt; Math.pow(digit, 2) }).foldLeft(0.0) { _ + _ }).toInt
```

`{\_ + \_}` seems a lot like line noise. The underscore converts the statement into a closure. It's a shortcut for `(a,b)=>a+b`. It's succinct yet should be used sparingly.

Step2:return

lol, yeah, no extra fluff is needed. We can capture the constraints in one return statement, but before that, I need to decide on what states I need to carry on from each recursive step. (You know I'm going to use recursion, don't you? :P)

There are a couple of states involved:
1. the limit. It's the number of steps we allow before the final "1" is reached. It took number seven 5 steps to reach 1. This variable is the cut-off: if the given number can't reach 1 before the limit, it's considered unhappy (return `false`)
2. the current number of steps
3. the set of numbers already appeared. In the case of seven, the set is `{49 97 130 10}`. This set is used to determine if the calculations fall into a loop. If the current calculated number appears in this set, the original number is unhappy.

So here's our method:

```scala
def isHappyNumber(n:Int, limit:Int, numOfTries:Int, alreadySeen:Set[Int]):Boolean = {
val sos = ((n.toString.toCharArray.map { digit =&gt; Math.pow(digit, 2) }).foldLeft(0.0) { _ + _ }).toInt
return sos == 1 ? true : (!alreadySeen.contains(sos) &amp;&amp; numOfTries+1 &lt;= limit &amp;&amp; isHappyNumber(sos, limit, numOfTries+1, alreadySeen+sos))
}
```

The second line basically says: when `sos` (sum of squares) is 1, return `true`, otherwise, is the number already in the set of numbers seen during the calculation?, if not, does the number of calculation exceed the limit? if not, repeat the calculation, with sos being the "original" number, increase the counter and put sos in the `alreadySeen` set.

Note that this is a [Tail recursion](http://en.wikipedia.org/wiki/Tail_recursion) function, we can add `@tailrec` annotation to the method to let the compiler optimize it - turn the recursion into a loop so that it won't grow in stack.

Now that we've had the body of the function, we can write an overload method that provides initial values:

```scala
def isHappyNumber(n:Int):Boolean = isHappyNumber(n, 10, 0, Set[Int]())
```

To find all happy numbers between 1 and 100:
```scala
println (1 to 100 filter { isHappyNumber(_) })
```
