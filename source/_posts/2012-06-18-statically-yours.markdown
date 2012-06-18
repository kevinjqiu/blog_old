---
layout: post
title: "Statically Yours"
date: 2012-06-18 16:54
comments: true
categories: technology
---

**Migrating from wordpress to octopress**

I'm not an avid blogger but like everyone else, I have a [blog](http://reminiscential.wordpress.com) which I casually write about life and programming. Being hosted by [wordpress](http://wordpress.org), it was an out-of-the-box solution and comes with a lot of bells and whistles. However, for a programming blog, it has some significant shortcomings:

*  Conflation of content and style.
   A Wordpress post is written in a weird combination of HTML markups and custom wordpress macros, which means you have to rely on their WYSIWYG editor to generate the correct markups, which means you can't use your favourite editor to write a blog post.

*  Limited versioning.
   Everytime you save a blog post, it creates a revision, but to view the diff is not as easy as `git diff`.

*  Embedding a code snippet sucks
   Wordpress uses a syntax highlighting macro, but the language it supports is very limited. There's third-party plugin that allows you to embed code snippet from [gist](http://gist.github.com), but you have to subscribe to their premium plan.


