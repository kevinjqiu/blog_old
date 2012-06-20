---
layout: post
title: "GWT code split is awesome"
date: 2010-03-26 13:38
comments: true
categories: java, gwt
---

For the past few days, I've been working on code splitting our existing GWT application. The application is fairly big with about 20k loc (excluding javadoc, comments and tests). The download size (with obfuscated mode) is about 1.5M, and with pretty mode, is a whooping 10+M. Obviously this is not acceptable.

GWT2 provides developer guided code splitting feature. It sounds complicated and under the hood, it may very well be (involves a lot of graph theories to figure out dependencies and so on) but from the user's point of view, it's very easy. You just have to wrap your potentially big operations in a `GWT.runAsync()` call.

However, since our application is using GWT best practices (dependency injection, MVP pattern), it's not as straightforward as GWT doc describes. A bit of digging on the internet leads me to [this page](http://code.google.com/p/google-gin/issues/detail?id=61)

## Gin patch

Someone contributed a patch to gin, which made split points transparent to the user of gin. The presenters that aren't needed initially can be wrapped in an AsyncProvider&lt;T&gt; instance - which by the blessing of deferred binding, translates into a GWT.runAsync call in the generated code. The patch hasn't been accepted into gin's trunk yet, but it's fairly easy to apply the patch and rebuild. A huge thanks to [fazal.asim](http://code.google.com/u/fazal.asim/) who hacked and contributed this patch.

## Results

The result of code splitting is encouraging - with very little structural change, we're able to reduce the initial download size to 29% of the total size:

<a href="http://reminiscential.files.wordpress.com/2010/03/untitled.png"><img class="aligncenter size-medium wp-image-194" title="code split result" src="http://reminiscential.files.wordpress.com/2010/03/untitled.png?w=300" alt="" width="300" height="93" /></a>

## Changes
I mentioned we needed very little structural change, but we did have to change something around. This is because with code splitting, the presenters that are split out from the initial download are not instantiated until they're used/downloaded. This means you cannot put logic in their constructors, and responding to place change has to be initiated by the container presenters.

## Improvements
Code splitting is awesome. However, if I'm allowed to voice a complaint, the report compiling time is just excruciating! For our application, it usually takes about 10 minutes to compile SOYC report - maybe a few minutes too long given the specs of my machine isn't too bad (Quad Core, 3G memory). Also, the compiled SOYC report takes up 600M of hard disk space! Ouch! Maybe instead of emitting HTML pages, they can make SOYC report a JS application, with data being encoded in JSON format?

Anyway, this doesn't take anything away from the awesome job GWT team has done for developers.

## Follow-up
Thanks to AsyncProxy, which provides a blocking (synchronous) interface while utilizing GWT.runAsync. This way, I'm able to build a view proxy that implements the same interface while keeping the real view components out of the initial call graph. The result of this, is a further reduction of the initial download size.

<a href="http://reminiscential.files.wordpress.com/2010/03/untitled1.png"><img class="aligncenter size-medium wp-image-197" title="Untitled1" src="http://reminiscential.files.wordpress.com/2010/03/untitled1.png?w=300" alt="" width="300" height="88" /></a>The initial download size is 13.77% of the total code size! Sweet!
