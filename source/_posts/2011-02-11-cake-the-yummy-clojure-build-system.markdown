---
layout: post
title: "Cake - the yummy Clojure build system"
date: 2011-02-11 19:56
comments: true
categories: clojure
---

About 10 minutes ago I heard about [cake](https://github.com/ninjudd/cake) clojure build system, and gave it a try. And 10 minutes later, it won me over! Wow, it addresses all the pain points of [leiningen](https://github.com/technomancy/leiningen).

## BLAZINGLY FAST!!!

Sorry for using all CAPS but I'm very excited about this improvement over leiningen -- OK, it may not be the fault of leiningen that JVM cold startup time is non-trivial but hey, someone came up with an idea of having a long running JVM process in the background, so subsequent clojure tasks reuse the same JVM instance. Cake folks integrated that nicely. It takes about 10-15 seconds to boot up a JVM but subsequent cake tasks or execution of clojure code is virtually instant! Comparing to leiningen, which doesn't take this approach and every single task (such as common ones like lein test) takes around 5 seconds. This adds up quickly and makes you less efficient. The speed improvement alone is enough for me to switch to cake.

## Advanced REPL functionalities: tab completion, history

It just works. Very useful for having instant feedbacks while exploring the language and API. No more manually adding jLine to your classpath or hack around tab completion wrapper...It just works! (I know I said it already)

## run clojure files directly

OK, leiningen can do this too, but through [plugin](https://github.com/sids/lein-run). I feel this is a very handy functionality, which probably should be included in the core.

## autotest

Detects your code change and automatically run your test suites! Sweet.

## compatible with leiningen project definition files

Cake understand `project.clj`, so I don't need to do anything for my existing leiningen projects. Change directory to the project and `cake` away :D

Overall, it just works out of the box. No more mucking around with dev-dependencies and other chores and let you focus on what you'd love to do.
