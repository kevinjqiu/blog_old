---
layout: post
title: Stripe Capture The Flag 2.0 - Problem 0
date: 2012-09-09 23:11
comments: true
categories: ctf
---

[Stripe](https://stripe.com) just finished running a second ["capture the flag"](https://stripe-ctf.com) challenge. They ran a similar challenge this February and was more focused on system level. This time, it's full-on web security.

In the next few posts, I'm going to discuss the problems in the challenge, how I solved them and what did I learn from from each challenge.

## Problem 0

Here are the code for level 0:

{% gist 3688655 level00.js %}
{% gist 3688659 level00.html %}

So you have a node.js server script, with an HTML front-end. The front-end allows you to submit a web form which allows you to retrieve *your* stored secret but the secret to level 1 is also stored in the same database.

Reading the code, the query on line 34 jumps out at you:

```javascript
    var query = 'SELECT * FROM secrets WHERE key LIKE ? || ".%"';
```

Even though I'm not too familiar with [nodejs](http://nodejs.org) or its db API, the part where it concatenates user input with ".%" looks suspicious. `||` is the SQL operator for concatenation, and '%' is the SQL wildcard that matches 0 or more characters of any kind. What if my user input is "%"?

Voilà! That's it! `%.%` gives you all passwords with namespace that has a dot in the middle.

## Conclusion

[SQL-injection](http://xkcd.com/327/) is a known security issue for a long time yet you'd be surprised how many sites are still subject to such exploits. The problem with level 0 code is exactly that: unsanitized user input is sent directly to the database for execution. So everytime a string concatenation is seen in a SQL statement, you have to ask yourself: is the ting being concatenated trustworthy? Use prepared statement or your database's escape function wherever possible.
