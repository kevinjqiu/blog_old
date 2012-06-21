---
layout: post
title: "Use function currying to reduce repetition and make code clean"
date: 2010-09-29 19:56
comments: true
categories: scala
---

Lately, I've been writing a parser using the Scala parser-combinator framework to parse some saves from a game. As a responsible programmer (:P), I write unit tests for each rule. However, I found myself having to write the following code over and over again:

```scala
@Test
def testRule1() {
  parserRule.apply(new CharSequence("someInput")) match {
    case Success(result, _) => {
      assertEquals("expected", result)
      /* other asserts if the result is a collection of something else */
      }
    case NoSuccess(msg, _) => fail(msg)
  }
}
```

It's worth noting that you cannot pass in a string value to a `Parser`. A parser expects a `CharSequence` `CharSequenceReader` object. Luckily, with Scala, you can define implicit conversion to turn a string into a `CharSequence` `CharSequenceReader`:

```scala
implicit def str2charSeqReader(v:String) = new CharSequenceReader(v)
```

Another goody Scala offers is that `obj.apply(arg)` method can be invoked by `obj(arg)`, so the above code becomes:

```scala
/* ... */
parserRule("someInput") match {
/* ... */
}
```

It's much cleaner.

However, the matching part is not clean still. It's not that bad for a single test case, but it's hardly a good practice to copy/paste it all over the test code base. We need to refactor this. The first thing comes to mind is to extract the matching part into a separate method, and let the caller provide the assertion part. Because Scala is a functional language, we can declare the method takes a function as a parameter, instead of using the delegate pattern, creating another class to encapsulate the method to be called, as you would do in Java:

```scala
def tryMatch[T](pr:ParseResult[T], f:(T)=>Unit) {
  pr match {
    case Success(r, _) => f(r)
    case NoSuccess(msg, _) => fail(msg)
  }
}
```

Now the consumer (the test code) looks like this:

```scala
@Test
def test1() {
  tryMatch(rule1("input"), (result)=>{assertEquals("expected",result)})
}
```

It's better, except that when you have multiple assert* methods need to be called, it quickly becomes ugly:

```scala
@Test
def test2() {
  tryMatch(listRule("1,2,3,4"), (result)=>{assertEquals(1,result(0));assertEquals(2,result(1));assertEquals(3,result(2));assertEquals(4,result(3))})
}
```

To do better, we can use [function currying](http://en.wikipedia.org/wiki/Currying). I read about function currying before, but not until now did it dawn on me that I can use this technique to make my code look cleaner. Basically, currying means that a function with n parameters is the same as the function being applied one parameter at a time in succession. e.g., function `add(x1:Int,x2:Int,...,xn:Int)` adds all its parameters. `add(1,2)=[add(1)](2)`, where `add(1)` becomes a partially applied function that takes an integer and returns an integer. Then the partial function is applied to the second parameter which gives the eventual result.

This is all very theoretical, but the practical implication is that we can transform `tryMatch()` method shown above from a method with 2 parameters into a curried function. Why? Because with Scala, when you have a method that only takes a single parameter (as is the case with the partially applied function), you can replace parentheses with curly brackets! Let's see how it works:

```scala
def tryMatch[T](pr:ParseResult[T])(f:(T)=>Unit) {
  pr match {
    case Success(r, _) => f(r)
    case NoSuccess(msg, _) => fail(msg)
  }
}
```

Basically, we break the two parameter definitions. This creates a PartialFunction in Scala. No other code needs to be changed at all. Now we can call tryMatch like this:

```scala
@Test
def testList() {
  tryMatch(listRule("1 2 3")) {
    result => {
      assertEquals(1, result(0))
      assertEquals(2, result(1))
      assertEquals(3, result(2))
    }
  }
}
```

This is like creating your own control flow (like try-catch-finally). Ain't that neat?
