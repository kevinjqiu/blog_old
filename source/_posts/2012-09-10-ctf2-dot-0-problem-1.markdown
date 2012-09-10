---
layout: post
title: Stripe Capture The Flag 2.0 - Problem 1
date: 2012-09-10 14:14
comments: true
categories: ctf
---

## Level 1

Now we get to level 1. We are presented with a simple web form with the PHP code powering it. 

{% gist 3692642 index.php %}

The PHP script checks if the input combination matches the combination in 'secret-combination.txt' file, and present the user with the password to the next level if the combinations match.  Obviously, we're not going to guess the combination. 

There are a few 'handy' methods in PHP that are extremely dangerous. [`extract`](http://php.net/manual/en/function.extract.php) is one of them. It will extract the content of the passed-in associative array, and import them into the global scope. e.g., `extract(array('foo'=>'bar'));` will make a global variable `$foo`. What's more dangerous is that if you already have a variable named `$foo`, it will be overwritten with the new value in the associative array.

Because the secret combination's location is stored in `$filename` variable, we need to somehow manipulate the input to point `$filename` to something else.  Looking at line 27:

```html
<form action="#" method="GET">
</form>
```

So the form is submitted using GET! So manipulating the variable is as easy as sending the endpoint with query param `filename=<xyz>`.

Now, what will the `xyz` be? The `$filename` variable is passed into `file_get_contents()` function. The parameter to the function is simply a string, and PHP defined a few 'handy' [streams](http://php.net/manual/en/wrappers.php.php). `php://input` caught my eyes. The doc says `php://input is a read-only stream that allows you to read raw data from the request body.`. Hey, the form is submitted using GET, so there won't be a request body. The input parameter is also sent using GET variable `attempt`, so I just need to send an empty `attempt` and point the filename to `php://input`: `?attempt=&filename=php://input`

...And indeed it works!

# Conclusion

* Never, ever use `extract()` in serious applications. Historically, PHP is used to build simple websites so it included many functions that puts "convenience" over security. Global variables are a bad idea, and having the ability to pollute the global space from any input is way worse.
* `file_get_contents()` has the ability to take any string as parameter, including some named streams. They are handy but they pose potential threats.
* Again, don't trust user input!
