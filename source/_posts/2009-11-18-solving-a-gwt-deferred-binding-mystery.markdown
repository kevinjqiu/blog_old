---
layout: post
title: "Solving a GWT deferred binding mystery"
date: 2009-11-18 10:55
comments: true
categories: gwt, java
---

This morning at work, I was running our GWT application with some deferred binding logic in it, and all of a sudden I got this ridiculous message:

```
[ERROR] Class 'mycompany.rebind.HistoryResourceGenerator' must derive from 'com.google.gwt.core.ext.Generator'
[ERROR] Failure while parsing XML
com.google.gwt.core.ext.UnableToCompleteException: (see previous log entries)
at com.google.gwt.dev.cfg.ModuleDefSchema$ObjAttrCvt.convertToArg(ModuleDefSchema.java:729)
at com.google.gwt.dev.util.xml.HandlerArgs.convertToArg(HandlerArgs.java:64)
at com.google.gwt.dev.util.xml.HandlerMethod.invokeBegin(HandlerMethod.java:214)
at com.google.gwt.dev.util.xml.ReflectiveParser$Impl.startElement(ReflectiveParser.java:257)
at com.sun.org.apache.xerces.internal.parsers.AbstractSAXParser.startElement(Abstract...)
```
It was running fine yesterday when I left work, and now it tells me that my generator isn’t a subclass of the GWT Generator??? Quickly pulled the source, and it’s as clear as day that my generator is properly written. Then what gives?

Searching for an explanation, I pulled the GWT’s source code, and opened `com.google.gwt.dev.cfg.ModuleDefSchema` and here’s the method in question:

```java
  public Object convertToArg(Schema schema, int lineNumber, String elemName,
      String attrName, String attrValue) throws UnableToCompleteException {

    Object found = singletonsByName.get(attrValue);
    if (found != null) {
      // Found in the cache.
      //
      return found;
    }

    Class<?> foundClass = null;
    try {
      // Load the class.
      //
      ClassLoader cl = Thread.currentThread().getContextClassLoader();
      foundClass = cl.loadClass(attrValue);
      Class<? extends T> clazz = foundClass.asSubclass(fReqdSuperclass);

      T object = clazz.newInstance();
      singletonsByName.put(attrValue, object);
      return object;
    } catch (ClassCastException e) {
      Messages.INVALID_CLASS_DERIVATION.log(logger, foundClass,
          fReqdSuperclass, null);
      throw new UnableToCompleteException();
    } catch (ClassNotFoundException e) {
      Messages.UNABLE_TO_LOAD_CLASS.log(logger, attrValue, e);
      throw new UnableToCompleteException();
    } catch (InstantiationException e) {
      Messages.UNABLE_TO_CREATE_OBJECT.log(logger, attrValue, e);
      throw new UnableToCompleteException();
    } catch (IllegalAccessException e) {
      Messages.UNABLE_TO_CREATE_OBJECT.log(logger, attrValue, e);
      throw new UnableToCompleteException();
    }
  }
}
```

Because the error message was from `Messages.INVALID\_CLASS\_DERIVATION`, I put a breakpoint on line 22:

```java
Messages.INVALID_CLASS_DERIVATION.log(logger, foundClass, fReqdSuperclass, null);
```

and started my app in debug mode. The breakpoint hits, and I noticed that the exception object is reporting `ClassCastException` from `JSONNull` to `JSONObject`. I quickly clicked: The error is from the class that fetches some resource over the net and plugged in to the generator. Following this lead, I found that the server resource the class was fetching wasn’t available any more, which caused the fetcher class to fail, and consequently, the `ClassCastException` is propagated to the generator and eventually `ModuleDefSchema`.

The mysterious error message, however, has nothing to do with the real problem. I spent a lot of time thinking that my `Generator` is wrong, because that’s what the compiler says, but in fact, it’s not. I think the confusion can be avoided if the GWT compiler can log the actual exception object, instead of interpreting the exception for the user. That’s what they’re doing in the `try/catch` clause in the method shown above: the exception object `e` is not used in the case of `ClassCastException`…

Well, the take-home message is: don’t always trust the GWT compiler message. Also, adding log more logging to your generator classes that can be indicative to where the actual problem is.

