---
layout: post
title: "Building a GAE+GWT application using the best practices (Part 4)"
date: 2010-03-03 13:15
comments: true
categories: java, gwt, gae
---

## Building a GAE+GWT application using the best practices series
1. [Part 1](/2010/02/26/building-a-gae-plus-gwt-application-using-the-best-practices-part-1/)
2. [Part 2](/2010/03/01/building-a-gae-plus-gwt-application-using-the-best-practices-part-2/)
3. [Part 3](/2010/03/03/building-a-gae-plus-gwt-application-using-the-best-practices-part-3/)
4. Part 4
5. [Part 5](/2010/03/09/building-a-gae-plus-gwt-application-using-the-best-practices-part-5/)

In the last blog post, we went over how to write GWT-RPC handlers using GWT-dispatch and dependency injection (Guice). This section, we're going to see how the client side is set up.

## Dependencies
We need the following dependencies
* [Gin](http://code.google.com/p/google-gin/)
* [GWT-dispatch](http://code.google.com/p/gwt-dispatch/)
* [GWT-presenter](http://code.google.com/p/gwt-presenter/)
* [GWT-log](http://code.google.com/p/gwt-log/)

They need to be on the classpath when you compile your GWT code, but not under the war directory like the server dependencies need to be.

## Module definition
<a href="http://reminiscential.files.wordpress.com/2010/03/screenshot1.png"><img src="http://reminiscential.files.wordpress.com/2010/03/screenshot1.png?w=200" alt="" title="Screenshot" width="200" height="300" class="aligncenter size-medium wp-image-172" /></a>

The first step is to declare the inherited GWT modules in the module XML file:
RateChecker.gwt.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<module rename-to="ratechecker">
    <inherits name="com.google.gwt.user.User" />
    <inherits name="com.google.gwt.inject.Inject" />
    <inherits name="net.customware.gwt.dispatch.Dispatch" />
    <inherits name="net.customware.gwt.presenter.Presenter" />
    <inherits name="com.allen_sauer.gwt.log.gwt-log-DEBUG" />
    <inherits name="com.google.gwt.user.theme.chrome.Chrome"/>

    <entry-point class="ratechecker.client.RateChecker" />

    <source path="client" />
    <source path="shared" />

</module>
```

Here we specify the explicitly the packages need to be included for compiling - "client" and "shared". If not specified, GWT will by default compile every source file under the client package.

## Create a Gin Module

Gin and Guice implements the same API (or rather, Gin has the same interface as Guice), but Gin uses GWT deferred binding "magic". Similar to what we have on the server side, on the client side, we start by defining our module:
RateCheckerClientModule:

```java RateCheckerClientModule.java
public class RateCheckerClientModule extends AbstractPresenterModule {

    public RateCheckerClientModule() {
    }

    @Override
    protected void configure() {

        bind(EventBus.class).to(DefaultEventBus.class).in(Singleton.class);
        bind(PlaceManager.class).in(Singleton.class);

    }
}
```

To start up, we bind EventBus and PlaceManager in the singleton scope. They're both provided by GWT-mvp library.

## AppPresenter

There are different ways to facilitate the MVP pattern but the way I find the most convenient is to have an AppPresenter manage all subsequent presenters. The view the AppPresenter represents is the RootPanel of GWT.

```java AppPresenter.java
public class AppPresenter {

    private HasWidgets _container;

    private final MainPresenter _mainPresenter;


    @Inject
    public AppPresenter(final MainPresenter mainPresenter) {
        _mainPresenter = mainPresenter;
        _mainPresenter.bind();
    }


    public void go(final HasWidgets container) {
        _container = container;
        _container.clear();
        _container.add(_mainPresenter.getDisplay().asWidget());
    }
}
```

Here, MainPresenter is the actual UI. The go() method of AppPresenter is for the module entry point to call when the module first initializes. We need to add the bindings to the client module:

```java
        ...
    @Override
    protected void configure() {

        bind(EventBus.class).to(DefaultEventBus.class).in(Singleton.class);
        bind(PlaceManager.class).in(Singleton.class);
        bind(ILog.class).to(GwtLogAdapter.class).in(Singleton.class);
        bind(AppPresenter.class);

        bindPresenter(MainPresenter.class, MainPresenter.Display.class, MainView.class);
    }
        ...
```

Here we specify the explicitly the packages need to be included for compiling - "client" and "shared". If not specified, GWT will by default compile every source file under the client package.


## Ginjector

Similar to "Injector" interface on the server side, the client side needs to define a Ginjector that act as a gateway for Gin managed object instances. 

```java RateCheckerGinjector.java
@GinModules({RateCheckerClientModule.class, ClientDispatchModule.class})
public interface RateCheckerGinjector extends Ginjector {

    AppPresenter getAppPresenter();

}
```

Here the annotation `@GinModules({...})` makes the instances managed by `RateCheckerClientModule` and `ClientDispatchModule` available for the ginjector. ClientDispatchModule binds DispatchAsync interface, which is what we will use to interface with the web service methods.

## Entry Point

Finally, here's the module entry point:
```java
/**
 * Entry point classes define <code>onModuleLoad()</code>.
 */
public class RateChecker implements EntryPoint {

    RateCheckerGinjector _injector = GWT.create(RateCheckerGinjector.class);

    @Override
    public void onModuleLoad() {

        final AppPresenter appPresenter = _injector.getAppPresenter();
        appPresenter.go(RootPanel.get("root"));
    }

}
```

`GWT.create(...)` statement here creates the ginjector at runtime. Behind the scene, it generates a class (by the name of something like RateCheckerGinjector_Impl) that contains the code to instantiate the bound classes, and when Gin sees a @Inject annotation on a class's constructor, it provides the instances with the correct scope from the dependency injection container (Ginjector) to the constructor so that the said class can be instantiated.

The onModuleLoad() method doesn't do much. It simple binds the appPresenter with the RootPanel where the app's UI is going to be displayed.

I know a lot of the concrete UI creation has been left out of this post, but hopefully it will become clearer once the next post is in.

