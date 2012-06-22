---
layout: post
title: "Scala Simple Build Tool -- Not so simple after all"
date: 2011-03-17 20:04
comments: true
categories: scala, sbt
---

**Update:** I got sbt working by building directly from the master branch from their github repo. The current version is 0.7.5. The tagged 0.9.4 version is actually an older version. Anyway, tried it and kinda loved it.

This is just another late night rambling...I was trying to get a proper scala build system setup. I was using Maven scala plugin for a while, but longing for something simpler and more scalanic (is there such a word?). I was pretty [happy](2011/02/11/cake-the-yummy-clojure-build-system/) with [Cake](https://github.com/ninjudd/cake), the Clojure build system and expected SBT to allow me to break away from using Maven to build Scala projects...boy, was I wrong...

First off, when you google 'simple build tool', you get a link to the SBT [Google code home page](http://code.google.com/p/simple-build-tool/). Well, nothing wrong there, except the "latest" version on Google code was 0.7.4 and it was half a year ago...Maybe it's not that outdated, so I downloaded it, followed [this](http://code.google.com/p/simple-build-tool/wiki/Setup) instruction and setup my `~/bin/sbt` script. Running it, it asked me to setup projects, and it only supported up until Scala 2.7.7...Hrm, 2.8 was out for a while now, so obviously, SBT 0.7.4 isn't the latest. Reading their home page more carefully, they're moving the repository to [Github](https://github.com/harrah/xsbt). Awesome! I'd pick Github over Google Code any time too.

Heading over to their Github repo, and found the latest stable version is 0.9.2. Good! So it should support Scala 2.8 now! Downloaded the zip, unzipped it, and of course it wasn't executable. You need to build it. There's a README.md, so quickly I less'ed it. For step 1, it asked me to go to the setup wiki page on Google Code (!), which is the steps I did setting up 0.7.4...I guess they're using 0.7.4 as a bootstrapping build...Anyways, I did that. Step 2 was to run `sbt update "project Launcher" proguard "project Simple Build Tool" "publish-local"`. Of course it didn't work. It's complained 0.7.4 version of sbt-launch can't download Scala 2.7.7 from any of the repository...bummer! But hey, I can download Scala 2.7.7 lib from Maven! So I quickly updated pom.xml of one of my projects to use Scala 2.7.7 and did an upgrade. Now 2.7.7 is happily in my local Maven repo. Ran that command again, hooray! It started to build, and judging by the number of packages it's building, "simple" isn't the first adjective that comes into my mind. Anyway, it's building at least, so even if it's a little complicated, so be it...Except...of course it broke half way... and why?

```
[info]   Post-analysis: 107 classes.
[info] == Precompiled 2.7.7 / compile ==
[info] 
[info]    Precompiled 2.8.0 / compile ...
[info] 
[info] == Precompiled 2.8.0 / compile ==
[info]   Source analysis: 9 new/modified, 0 indirectly invalidated, 0 removed.
[info] Compiling main sources...
[warn] there were deprecation warnings; re-run with -deprecation for details
[warn] one warning found
[info] Compilation successful.
[info]   Post-analysis: 108 classes.
[info] == Precompiled 2.8.0 / compile ==
java.lang.OutOfMemoryError: PermGen space
        at java.lang.ClassLoader.defineClass1(Native Method)
        at java.lang.ClassLoader.defineClassCond(ClassLoader.java:632)
        at java.lang.ClassLoader.defineClass(ClassLoader.java:616)
```

You've gotta be kidding me! I set `-Xmx512M` and it's not enough? And why is it building every version of Scala **from source**?? Is there something called a...JAR? 

Anyway, increased `-Xmx` from 512 to 1024M, ran again, wait, and same thing happened again! Out of PermGen space...urrgh...

I decided to give up, at least for the day... SBT is anything but simple, at least from my experience. I know it's open source and people put efforts into it without compensation, so I shouldn't be critical about it. I'll give it a try again, and hopefully it's worth the time investment.
