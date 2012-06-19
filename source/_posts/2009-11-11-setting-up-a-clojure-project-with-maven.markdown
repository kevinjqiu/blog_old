---
layout: post
title: "Setting up a Clojure Project with Maven"
date: 2009-11-11 11:00
comments: true
categories: clojure
---

In this blog post I’m going to record my recent experience in setting up a Clojure project using the [clojure-maven-plugin](http://github.com/talios/clojure-maven-plugin).

## Clojure-Maven-Plugin

First you need to compile the plugin from source:

```
git clone git://github.com/talios/clojure-maven-plugin.git
cd clojure-maven-plugin
mvn install
```

Of course, you will need to have Maven2 installed already.

After that, the compiled plugin jar will be in your maven local repository. Create a `pom.xml` file to use the plugin. I’m using the pom.xml from my project [Programming Collective Intelligence](http://github.com/kevinjqiu/pci-clj) as an example:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>pci</groupId>
    <artifactId>pci</artifactId>
    <name>Programming Collective Intelligence</name>
    <version>0.0.1-SNAPSHOT</version>

    <build>
        <plugins>
            <plugin>
                <groupId>com.theoryinpractise</groupId>
                <artifactId>clojure-maven-plugin</artifactId>
                <version>1.2-SNAPSHOT</version>
            </plugin>
        </plugins>
    </build>

</project>
```

Also, setting up clojure-lang and clojure-contrib (optional, but nice to have) as dependencies as follows:

```xml
<dependencies>
    <dependency>
        <groupId>org.clojure</groupId>
        <artifactId>clojure-lang</artifactId>
        <version>1.1.0-alpha-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>org.clojure</groupId>
        <artifactId>clojure-contrib</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
</dependencies>
```

Clojure builds haven’t reached Maven central repository yet, so you need to build Clojure from source, and install it into your local repository by using

```bash
mvn install:install-file -Dfile=clojure-lang.jar -DgroupId=org.clojure -DartifactId=clojure-lang -Dversion=1.1.0-alpha-SNAPSHOT -Dpackaging=jar
```

Do the same for clojure-contrib.

You can also create a directory in under your project root and have pom.xml looking for artifacts in that directory as well. I found this to be useful for artifacts that are not in the central repository. To do so, add repository definition in pom.xml:

```xml
<repositories>
    <repository>
        <id>lib-repo</id>
        <name>lib-m2-repository</name>
        <url>file://${basedir}/lib-repo</url>
        <layout>legacy</layout>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
        <releases>
            <checksumPolicy>ignore</checksumPolicy>
        </releases>
    </repository>
</repositories>
```

Afterwards, you need to setup your directory structure as follows:
```
${project_root}
-- lib-repo
    -- org.clojure
        -- jars
            * clojure-lang-1.1.0-alpha-SNAPSHOT.jar
            * clojure-contrib-1.1.0-alpha-SNAPSHOT.jar
        -- poms
            * clojure-lang-1.1.0-alpha-SNAPSHOT.pom
            * clojure-contrib-1.1.0-alpha-SNAPSHOT.pom
```

The pom files can be found in your local repository after you’ve done `mvn install:install-file`.

If you want to run `clojure:repl` goal, you’d better add jline to your dependency as well:

```xml
<dependencies>
  ...
        <dependency>
            <groupId>jline</groupId>
            <artifactId>jline</artifactId>
            <version>0.9.94</version>
        </dependency>

  ...
</dependencies>
```

By convention, the plugin compiles everything under `${project\_root}/src/main/clojure` and `${project\_root}/src/test/clojure`.

It’s hard to imagine working in a language like Clojure without automated unit tests. Fortunately, clojure-maven-plugin has the `clojure:test` goal which runs unit tests. All you need to do is telling the plugin where’s the entry point of your unit test. Add the following configuration in the build section:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.theoryinpractise</groupId>
            <artifactId>clojure-maven-plugin</artifactId>
            <version>1.2-SNAPSHOT</version>
            <configuration>
                <testScript>src/test/clojure/pci/test.clj</testScript>
            </configuration>
        </plugin>
    </plugins>
</build>
```

The test script looks like the following:
```clojure
(ns my-namespace
  (:use clojure.contrib.test-is
          test-module-1
          test-module-2))
(run-tests 'test-module-n)
```

There you have it! The sample files can be found in my [pci-clj](http://github.com/kevinjqiu/pci-clj) project on GitHub.

