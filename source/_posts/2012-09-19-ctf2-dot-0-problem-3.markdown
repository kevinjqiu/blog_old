---
layout: post
title: Stripe Capture The Flag 2.0 - Problem 3
date: 2012-09-19 00:01
comments: true
categories: ctf
---

## Level 3

Finally we get to level 3. Here's the setup:

```
After the fiasco back in Level 0, management has decided to fortify the Secret Safe into an unbreakable solution (kind of like Unbreakable Linux). The resulting product is Secret Vault, which is so secure that it requires human intervention to add new secrets.

A beta version has launched with some interesting secrets (including the password to access Level 4)
```

Here's the code for the server (Python finally!)
{% gist 3747632 %}

...and here's the front-end:
{% gist 3747637 %}

From the description, we know that we need to break into bob's account to retrieve the password to level 4, although breaking into eve and mallory's accounts are attempting :) Afterall, who wouldn't want to know the proof of [P=NP](http://en.wikipedia.org/wiki/P_versus_NP_problem) or how to make a [perpetual motion machine](http://en.wikipedia.org/wiki/Perpetual_motion_machine)?

Anyhow, the front-end is a typical login page: you get username and password input fields, and they're sent off (via POST) to a server script.

The server is a simple Flask app that gets the user input, checks them against a table of username and salted password hashes.

## TODO:
Tried:
* terminate sql => no go
* how about union the result? => go!


## Conclusion
This is a canonical SQL injection. Again, NEVER trust user input. Always sanitize.
