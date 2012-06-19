---
layout: post
title: "Explore Clojure: Building a Bifid cipher"
date: 2009-10-20 00:00
comments: true
categories: clojure
---

Lately I’ve been teaching myself [Clojure](http://clojure.org), a Lisp dialect on the JVM platform. I still love Erlang and still learning it, but Clojure has a special draw for me being a JVM language and its Lisp roots. I studied Scheme (another Lisp dialect) in my college years and deemed it purely academic. However, Clojure has the potential of changing this and bring the expressiveness of Lisp and the power of functional programming to the Java world.

The best way to learn a language is to use it to solve some non-trivial practical problems. I stumbled [this](http://programmingpraxis.com/2009/10/13/bifid-cipher/) problem on [Programming Praxis](http://programmingpraxis.com/). It’s about building a [Bifid](http://en.wikipedia.org/wiki/Bifid_cipher) cipher, which I thought could be a good practical problem for me to solve in Clojure.

## Polybius Square

Bifid cipher is based on [Polybius Square](http://en.wikipedia.org/wiki/Polybius_square), which looks like this:

```
  1 2 3 4 5
1 B G W K Z
2 Q P N D S
3 I O A X E
4 F C L U M
5 T H Y V R
```

Every encodable letter is given a vector value, which is the coordinate in this square. For example, `X=[3 4]`, `H=[5 2]`. The first task would be building the Polybius square in Clojure.

My function `polybius-square` is going to take 2 parameters: `charset` is a list of encodable characters, and `size` is the size of the Polybius square.

```clojure
(defn polybius-square
  "Polybius square by the given charset and size"
  [charset square-size]
  (map #(vector %1 [%2 %3]) charset
    (for [x (range 1 (+ 1 square-size)), y (range 1 (+ 1 square-size))] x)
    (take (* square-size square-size) (cycle (range 1 (+ 1 square-size))))))
```

Let’s see how this works. Suppose you’re calling this function with
`(polybius-square "ABCDE" 3)`, `charset` is bound to “ABCDE” and `size` is bound to 3. What I want is a seq of vectors in the form of: `(\Letter [x y])` where `x` and `y` are the coordinates. It’s natural to use `map`, with the function-to-map taking three parameters and vectorize them. That’s what the anonymous function does:

```clojure
#(vector %1 [%2 %3])
```

`%1` is the letter, `%2` and `%3` are the coordinates. I want the coordinate sequence to be like this: `[1 1], [1 2], [1 3], [2 1], [2 2], [2 3]`. In imperative language, you’d use a nested for loop, but we’ve established that’s ugly. We want the parameters fed to map to be of the following:
`"ABCDEFGHJI" (1 1 1 2 2 2 3 3 3) (1 2 3 1 2 3 1 2 3 1 2 3)`

To generate the first list, we use list comprehension:

```clojure
(for [ x (range 1 (+ 1 square-size)), y (range 1 (+ 1 square-size))] x)
```

It reads: for every `x` in range 1 to `square-size`+1 (because “range” is right-exclusive) and for every `y` in the same range, output the value of `x` and take the whole output as the list. Because the binding of `y` comes later, and Lisp evaluates from right to left, the result of this list comprehension is `(1 1 1 2 2 2 3 3 3)`

Now let’s look at the other list: `(1 2 3 1 2 3 1 2 3)` – a cyclic list. Fortunately, Clojure has a built-in `cycle` function, which returns a lazy sequence of cycles of a collection. We want to call it with cycle `'(1 2 3)`, so it’s going to be `(cycle (range 1 (+ 1 square-size)))`. Because cycle will be evaluated to an infinite sequence, we have to tell it when to stop.

When you see the square as a flat list, it’s clear that the cycle should stop when it reaches `square-size^2`, hence `(take (\* square-size square-size) (cycle (range 1 (+ 1 square-size))))`.


## The codec maps

After the Polybius square is built, we have the bases for our encryption. However, it’s not user-friendly: it’s a flat list of tuples. We want two maps for fast searching from letters to their encoding forms `(letter-2-code)` and from transposed vectors to the letters `(transposed-2-letter)`.

```clojure
(defn- letter-2-code
  "Produce a mapping between letters and their codes"
  [square]
  (letfn [(helper [the-square the-map]
          (if (empty? the-square)
            the-map
            (let [the-item (first the-square)
                  the-key (first the-item)
                  the-number (second the-item)]
              (recur (rest the-square) (assoc the-map the-key the-number)))))]
    (helper square (empty hash-map))))
```

The strategy here is to walk through the list, add each item to a `hash-map`. Here we define an inner function, which is basically a recursive helper. The function itself is pretty straightforward. Take note that in Clojure, when you need to do tail recursion, always use `recur` instead of calling the function by its name. This is because JVM cannot do automatic tail cut optimization (TCO), but Clojure does the optimization through the `recur` function. After the binding, we kick off the recursion by calling it with the initial list.

In the similar fashion, we have:

```clojure
(defn- transposed-2-letter
  "Produce a mapping between the transposed codes to letter"
  [square]
  (letfn [(helper [the-square the-map]
            (if (empty? the-square)
              the-map
              (let [the-item (first the-square)
                    the-key (second the-item)
                    the-letter (first the-item)]
                (recur (rest the-square) (assoc the-map the-key the-letter)))))]
    (helper square (empty hash-map))))
```

## Encode
Now comes the `encode` function. It’s important to note that the encoding will be different if a different polybius square is provided. Therefore, both `encode` and `decode` functions take the Polybius square as parameter.

During the encoding phase, we first convert every character in the message into a vector, then reorganize the vectors. To better illustrate, we use an example:
Suppose we have the following mapping:
`{I [3 3], H [3 2], G [3 1], F [2 3], E [2 2], D [2 1], C [1 3], B [1 2], A [1 1]}`
and we want to encode the word “CEDE”
`CEDE => ([1 3] [2 2] [2 1] [2 2])`, then we want to transpose the “matrix” composed of the above vectors, so it would be like:
`([1 2] [2 2] [3 2] [1 2])`, and finally looking up every item in the transposed-2-letter mapping

```
{[3 3] I, [3 2] H, [3 1] G, [2 3] F, [2 2] E, [2 1] D, [1 3] C, [1 2] B, [1 1] A}
=>BEHB
```

To get a mapping of message to vectors, we again map a function to the list:
`(map #(get l2c % %) message)` (l2c has been bound)

Then we apply the interleave function to the result, this gives:
`(1 2 2 2 3 2 1 2)`. This is good but we want a list of vectors instead of a flat list. Clojure provides partition function, which does exactly it:
`(partition 2 '(1 2 2 2 3 2 1 2))`
gives:
`([1 2] [2 2] [3 2] [1 2])`

Then, we want to lookup every item in the result in the `transposed-to-letter` map and get a list of mapped characters. To achieve this, again, we use map, providing a different mapping:
`(map #(get tr2l % %) ...)` (tr2l is bound)

This returns a list of characters. We want a string, so we concatenate the characters by applying `str` function:
`(apply str (map #(get tr2l % %) ...))`

Here’s the complete code:

```clojure
(defn encode
  "Return the message encoded by the specified polybius-square"
  [message polybius-square]
  (let [l2c (letter-2-code polybius-square)
        tr2l (transposed-2-letter polybius-square)]
    (apply str
      (map #(get tr2l % %) (partition 2 (apply interleave (map #(get l2c % %) message)))))))
```

Here you see the expressiveness of Clojure: all operations are chained together and formed in expression.

The `decode` function behaves similarly. The only trick is we need to split the translated code list in two. We use `partition` method again, with `size` being half of the entire list.

```clojure
(defn decode
  "Return the decoded message using the specified polybius-square"
  [encoded polybius-square]
  (let [l2c (letter-2-code polybius-square)
        c2l (transposed-2-letter polybius-square)
        codes (apply concat (map #(get l2c % %) encoded))
        len (count codes)
        parts (partition (/ len 2) codes)]
    (apply str (map #(get c2l %1 %1) (map #(vector %1 %2) (first parts) (second parts))))))
```

## Putting it together

Now with all the pieces are in place, we can put it together by writing a simple test:
```clojure
(defn -main
  "A simple test case entry point"
  []
  (let [square (polybius-square "ABCDEFGHIKLMNOPQRSTUVWXYZ" 5)]
    (do (println (encode "BIFIDCIPHER" square))
      (println (decode (encode "BIFIDCIPHER" square) square)))))
```

Output:

BGAHFRQTOXW
BIFIDCIPHER

Have the complete source code [here](http://github.com/kevinjqiu/bifid-clj), I’m sure there are a lot to be improved of the above implementation of the Bifid cipher. Especially currently I have no elegant way to solve the situation where the letter to be encoded is not in the Polybius square. Also, I’m sure my code isn’t entirely the idiomatic Clojure. If you have any suggestions, don’t hesitate to write a comment.

Although I have been learning Clojure for 3 days, I’m already enjoying its expressiveness and the clean look. As a Java programmer, Clojure also gives me the benefit of full access to the Java land. The potential is endless!

