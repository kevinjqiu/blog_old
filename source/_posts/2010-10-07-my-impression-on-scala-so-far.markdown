---
layout: post
title: "My impression on Scala so far"
date: 2010-10-07 20:01
comments: true
categories: scala
---

I've been exploring Scala on and off for some time now. Here's my highly subjective and very limited impression of Scala.

What I like about Scala:

## 0) It's statically-typed language.
That's right! I don't care what you ninjas say. As much as I love dynamic languages, I just prefer statically typed language for big projects. The benefit of having type information is enormous for a project with a large code base. I know you have to write unit tests for it anyways, but when 80% of your unit test code is checking types, it's just counter-productive.

## 1) The ability to combine functional programming style with OOP.
This is what Scala is known for - a hybrid language that boasts the better in both OOP world and FP world. I'm not a functional programming purist, but I think Scala did a good job blending the functional programming elements into OOP. Writing Scala programs, I find myself much more empowered to be able to choose from both styles where I see fit. The ability to pass functions around and apply high order functions can significantly reduce boilerplate code and visual clutter.

## 2) The ability to create your own control flow (well, sort of).
I had a [blog post](http://reminiscential.wordpress.com/2010/09/29/scala-use-function-currying/) about how to use function curry to create a customized control flow. This is very empowering and yet, it's not an one-off syntactic sugar that's sprinkled randomly into the language like some other language would do. Customization of control flow is a result of combining generic language features (such as currying, by-name parameter, implicit conversion, etc).

## 3) Object instead of static

Scala does away with the Java static keyword. Instead, it provides "object" keyword to define a class that only has one instance. It has the same power as singleton design pattern, without the extra boilerplate code. (if you think implementing singleton in Java is simple, think again. Especially in the multi-threaded context). Also, you can use the same construct to define "companion" objects, which can be used to implement the factory pattern, and put in miscellaneous methods, implicit conversion methods and so on. Extremely powerful yet elegant.

## 4) Operator overloading (well, sort of).

OK, it's not exactly operator overloading - Scala allows many non-alpha-numeric characters in the method name, so operators are essentially methods. duh! if you think about it...why should + be treated any different from `add()`? Being able to use special characters in method names makes code easy to understand. However, I'm a bit disappointed that question mark `?` cannot be used in method names...One of the things I liked about LISP is you can define a function `(odd? (x) (...))` and readers immediately know this function returns a boolean. I don't have to ponder whether I should name it `isOdd()` or `hasChildren()`. Anyway...

## 5) Traits

In most cases, multiple inheritance is manageable. Seems Java threw the baby out with the bath water - rejecting multiple inheritance completely. It's ironic that the AOP guys cracks open Java classes in the byte code level to do the "mixins"...With Scala, this seems to be natural.

## 6) Built-in parser-combinator library

It's easy to do some non-mission critical parsing using the Scala standard library. The library itself is implemented as an internal DSL such that writing parser rules feels like writing EBNF directly.

Here's what's not so hot for me so far:

## 1) The generics still is obtrusive sometimes.

That said, however, it's vastly superior than Java's generics system with far better type inferencing. But sometimes, you still have to write down a lot of types especially on a parameterized method. What's worse is because Scala is more strict than Java wrt types, you cannot ignore generic types the way you can in Java code. Moreover, because Scala runs on JVM, and because of type-erasure, you still can't write code such as val t = new T()...although we can't blame this on Scala.

## 2) The collection library.

For each collection data structure, Scala has the mutable implementation, immutable implementation, and the native Java Collection Library. If you're writing code that calls Java library, very often you have to wrap JCL lists/maps/sets into the Scala ones, because you want to use the nice functional features Scala provides. I'm not smart enough to know a better solution to this, I'm just merely pointing out a sore spot. Clojure has a very clever and elegant way of integrating JCL data structures into Clojure code, but since Clojure and Scala are vastly different, I don't know how relevant this is.

## 3) Start up time
One of Scala's goal is to be able to both scale up and scale down. Scala programs can be run as scripts by the interpreter, but it can't compete with Python or Ruby in terms of development turn-around time in terms of scripting. Again, this is because of the dreadful JVM startup process.

## 4) Lack of good tooling

The best Scala development environment I found so far is the Scala plugin for IntelliJ. It's still not great for debugging, but for coding and integration with Maven and such, it's great. The Eclipse plugin lags significantly, despite having built a brand new website and seemingly having more resource pour into it...I know I shouldn't be overly critical of a community effort, but it's just frustrating that it can't even do auto import of classes...

Overall, I think Scala really has the potential to become a big name in the programming language landscape. It can do everything Java can and does better. It has so much features that Java doesn't dare to add to keep backward compatibility or keep the language dead simple or politics or whatever reason...

Scala is not a hype, but in order to be adopted more rapidly, I think it needs to

1. Stablize the language and core library.

2. Tooling, tooling, tooling! Seriously.
