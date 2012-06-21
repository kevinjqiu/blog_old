---
layout: post
title: "Launch Windows Explorer and goto the current working directory in shell"
date: 2010-06-09 19:45
comments: true
categories: tips
---

I like working in the command prompt - it's fast and you can do batch processing, etc, but sometimes, I need a GUI (Windows explorer) to explore files in the current directory. I know I can type "explorer.exe" on the command line to launch Windows explorer, but can it automatically go to the directory I'm working in? The answer is yes, although hardly ever [advertised](http://support.microsoft.com/kb/314853")

Now, we need to find a way to find out the current working directory. On the command prompt, it's stored in the environment variable %CD%, so combine the two:

```
explorer /root,%CD%
```

and it works beautifully.

Put this in a .bat file and put it on your path, you can launch Windows explorer anywhere and have it go directly to the folder you're in on the command line!
