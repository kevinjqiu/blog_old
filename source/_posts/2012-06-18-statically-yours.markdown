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

Static site generators are becoming increasingly popular among developers. Developer blogs don't need a ton of visuals, so content with basic styling is more than enough. Most importantly, it has to do code embedding well. A bit of research, it turns out that [Octopress](octopress.org) seems to be ubiquitous.

After a day of setting up, here's what I found:

## Installation

I'm primarily a Python developer, so setting up a Ruby project is a bit foreign to me, especially since Octopress only works with Ruby 1.9+, I need to setup [RVM](http://rvm.io), and even though it's well documented, it's not without speed bumps on my Ubuntu system. In particular, you need to install `libssl-dev` before you let RVM compile and install Ruby, otherwise you will get something like this:

```
no such file to load -- openssl (LoadError)
```

I had to 
```
sudo apt-get install libssl-dev
```

and re-install Ruby
```
rvm reinstall 1.9
```

## Setting up TLD

After the first deploy, the pages are already accessible via `<yourname>.github.com`, but pointing your domain to the page is just as easy. I followed [this](https://help.github.com/articles/setting-up-a-custom-domain-with-pages) official article from Github.

## Static sweetness

There we go! Sweet static pages:

* Page views are extremely fast
* [Markdown](http://daringfireball.net/projects/markdown/syntax)
* Version controlled blog posts
* Embedding code is [easy](http://octopress.org/docs/blogging/code/)
