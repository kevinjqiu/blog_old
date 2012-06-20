---
layout: post
title: "Building a GAE+GWT application using the best practices (Part 1)"
date: 2010-02-26 00:39
comments: true
categories: java, gwt, gae
---
This is the first installment of the series [Building a GAE+GWT application using the best practices](#TODO).

## Introduction

In the next few blog posts, I'm going to present my experience building a simple (but non-trivial) web application using GWT and Google App Engine, while applying the best practices introduced by Ray Ryan in his excellent [GWT best practices](http://code.google.com/events/io/2009/sessions/GoogleWebToolkitBestPractices.html) at Google IO last year.

The application I'm going to build is called RateChecker. It's simply a tool that goes fetches the posted USD/CAD exchange rate from a bank website, persists the rate to the data store, and present the user with the recent rates when requested. It can potentially do more (like generating a histogram of the rates, etc), but for the purpose of blog series, I'm going with the basics to illustrate the implementation pattern without losing my bearings in the embellishing.

Here's a screenshot of the finished application:
<p style="text-align:center;"><a href="http://reminiscential.files.wordpress.com/2010/02/screenshot2.png"><img class="size-full wp-image-145 aligncenter" title="RateChecker" src="http://reminiscential.files.wordpress.com/2010/02/screenshot2.png" alt="" width="259" height="216" /></a></p>

## The Goals

I intend to use this minimal application to practice using the following technologies/techniques:
* Server/Persistence tier:
    * Google Guice as dependency injection container on the server side
    * JDO as the persistence layer
    * AppEngine Cron task
    * Command pattern for request handling
* Client tier:
    * Google GIN as dependency injection container on the client side
    * Model-View-Presenter pattern on the client side
    * EventBus to decouple components
    * Command pattern for remote service calls
* Unit Testing on both Client/Server side with mock objects

Also I plan to rewrite the service layer with one of the alternate JVM language, such as Scala or Clojure, if I my schedule allows and I have enough motivation.
