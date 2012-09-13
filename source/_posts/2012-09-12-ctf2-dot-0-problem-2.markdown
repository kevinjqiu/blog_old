---
layout: post
title: Stripe Capture The Flag 2.0 - Problem 2
date: 2012-09-12 23:50
comments: true
categories: ctf
---

## Level 2

{% gist 3711719 index.php %}

In level 2, we're faced with a PHP app that allows you to upload a "profile picture". The password to level 3 is contained in a "password.txt" file of the document root, as revealed in line 49. Of course, you won't be able to click on the link and get the file. The directory is protected, and we have to somehow exploit the code.

Reading through the code, it's a clear that whatever file uploaded to the server will be under ``uploads/``, and the file is publicly accessible through ``<base>/uploads/<your_file_name>``, as seen on line 37. The file input is expecting a image file, but it doesn't restrict the type of file it accepts. What if we upload a PHP script, read the file content `../password.txt`?

With that in mind, I quickly cooked up a PHP script:

```php
<?php
echo file_get_contents('../password.txt');
?>
```

uploaded it and hit it with curl. Guess what? The password is right there in the clear!

## Conclusion

There's a few problems with this app:

* The user shouldn't be able to upload files of any type they want. Restrict to only image files if you're expecting profile pictures.
* The above point is necessary but not sufficient. The bigger problem is that the server is not properly configured. Files under ``uploads/`` folder should be considered "user input" and thus should not be able to be executed on the server. Much more exploits can be done here and as it turns out, some later levels require the control of this machine.

