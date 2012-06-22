---
layout: post
title: "Spectrum.vim - My first Vim plugin"
date: 2011-04-20 20:09
comments: true
categories: vim, vimscript
---

ver the past few months, I've been using Vim as my primary development tool at work and at home, and I have to say, I'm addicted to it! I'm thinking about writing a blog post of why I get hooked on [walking without crutches](http://walking-without-crutches.heroku.com/), but for this post, I'm just going to introduce you to my first plugin in Vim - [Spectrum](https://github.com/kevinjqiu/spectrum.vim).

## Introduction
Spectrum is a vim colorscheme roulette. Ever getting tired of staring at the same colorscheme every day? Having hundreds of colorschemes in your repo but too lazy to deterministically pick one? Spectrum helps you by randomly pick colorschemes from your vim runtime path or from the [web](http://inspiration.sweyla.com/code/). From there, you have the chance of voting up a colorscheme so Spectrum will have a higher probability to pick it or voting down a colorscheme so you wouldn't see it again.

## Development
To be honest, I'm not a fan of [Vimscript](http://en.wikipedia.org/wiki/Vimscript) - the language is not very expressive and doesn't have a lot of object oriented features. Semi-fortunately, since Vim 7, they have added support for Python, Ruby and Perl scripts. I said 'semi-fortunately' because the support isn't too comprehensive. For most core vim features, you still have to resort to calling Vim commands to achieve them, but at least I don't have to use Vim script for the most part.

Spectrum is written in Python, and use the `vim` module to interact with the hosting Vim instance. There is a bit of bootstrapping to do if you want to separate most of the Python code out of the entry point vim script (see [here](https://github.com/kevinjqiu/spectrum.vim/blob/master/plugin/spectrum.vim)). Many vim plugins written in Python require you to install the python code into your Python runtime before you can use them, but for a simple module like Spectrum, I opted for monkey patching syspath to include modules in the plugin folder.

Anyhow, give it a try and hope you like it.

[https://github.com/kevinjqiu/spectrum.vim](https://github.com/kevinjqiu/spectrum.vim)
