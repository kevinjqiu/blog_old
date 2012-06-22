---
layout: post
title: "Write sudoku solver in Clojure"
date: 2011-03-02 20:00
comments: true
categories: clojure, sudoku
---

...yeah, because the world just needs another [Sudoku](http://en.wikipedia.org/wiki/Sudoku) solver. Well, I'm not trying to solve world hunger with it, but just an attempt to practice [clojure](http://clojure.org), I took (read: stole) Peter Norvig's [sudoku solver](http://norvig.com/sudoku.html) algorithm (written in Python) and adapted it into Clojure. I put it up on Github under [sudoku-clj](https://github.com/kevinjqiu/sudoku-clj). The algorithm itself isn't *that* hard to understand. The porting to a lisp-y syntax made the code a little longer than its Python counterpart. I'm sure seasoned Lisp/Clojure users can point out dozens of places where more idiomatic/succinct syntax can be used (If you happen to be one, do tell, by the way).

Here's a few things I noticed:

* Mutable states in clojure are captured using `ref`s. The object itself (in this case, the grid, which is a hash map) doesn't mutate, but the reference is changed to point to different grid objects that represent a configuration at a given step.

* Clojure sequences are Lazy. A few times I tried to print out the current state (remaining digits) of the square, but if you simply do `(println seq)`, you will get a Java-ish `toString()` output of the sequence object. You need to force the lazy sequence to be evaluated by `(println (apply str seq))`. Needless to say, you lose the advantage of lazy sequences, so use it sparingly.

* Python's list comprehension syntax is fabulous. Clojure's counterpart for comprehension doesn't feel as elegent, nor is map a function onto a sequence to achieve that (the way I used it)

* [Cake](/2011/02/11/cake-the-yummy-clojure-build-system/) is yummy!

* The performance isn't great...I must have done something wrong, but the easy sudoku grid took about 2 seconds (with the JVM already booted), while the Python algorithm solves it in a fraction of a second.

* Because assign/eliminate are mutually recursive, my current implementation uses the naive way of doing recursion, i.e., let the stack grow. Clojure has a function `trampoline`, which adds a level of indirection that applies to mutually recursive functions. It uses `recur` at tail end position (basically translates the recursive calls into loops) which doesn't fill your process's stack. It might not be obvious (to me anyways) how one can do that with a few levels of function calls in between assign/eliminate, but I'm sure there's a way
