---
layout: post
title: "Building a GAE+GWT application using the best practices (Part 2)"
date: 2010-03-01 00:44
comments: true
categories: java, gwt, gae
---

In Part 2, we're going to go over project setup for GAE and GWT applications, and wire the server (servlet) using [Guice](http://code.google.com/p/google-guice/) and [GWT-Dispatch](http://code.google.com/p/gwt-dispatch/).

## Project setup

I'm using Eclipse as my development environment. Install [Google Eclipse plugin](http://code.google.com/eclipse/), and install the provided GWT (2.0.2) and GAE (1.3.1) with the plugin. Create a new project in Eclipse using the "New web application project" wizard, and create a sample project.

Now, because during deployment, GAE applications are executed within its own servlet container, all dependencies have to be placed inside the directory /war/WEB-INF/lib. Go ahead, download [Guice](http://code.google.com/p/google-guice/downloads/list), [GWT-dispatch](http://code.google.com/p/gwt-dispatch/downloads/list), [GWT-log](http://code.google.com/p/gwt-log/downloads/list), [commons-logging](http://commons.apache.org/downloads/download_logging.cgi) and [log4j](http://logging.apache.org/log4j/1.2/download.html). Put the jar files inside /war/WEB-INF/lib directory. Then in Eclipse, select the jars you just placed, right click and select "Add to build path". Your lib directory should look something like this:

<a href="http://reminiscential.files.wordpress.com/2010/03/screenshot.png"><img class="aligncenter size-medium wp-image-158" title="Screenshot" src="http://reminiscential.files.wordpress.com/2010/03/screenshot.png?w=168" alt="" width="168" height="300" /></a>

## "Wiring" the server
Now that the project is setup, we need to wire the server to utilize the dependency injection container Guice. The details can be found [here](http://code.google.com/p/google-guice/wiki/GoogleAppEngine) but in short, we need to do the following:

### Modify web.xml
Find web.xml in /war/WEB-INF. In traditional GWT-RPC development, every service needs to be written as a servlet and declared in web.xml. For Guice + GWT-dispatch, we only need a filter and a listener (as the entry point).

```xml
<webapp>
        [...]
    <filter>
        <filter-name>guiceFilter</filter-name>
        <filter-class>com.google.inject.servlet.GuiceFilter</filter-class>
    </filter>

    <filter-mapping>
        <filter-name>guiceFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

    <listener>
        <listener-class>ratechecker.server.guice.GuiceServletConfig</listener-class>
    </listener>
        [...]
</webapp>
```

Here, our listener is ratechecker.server.guice.GuiceServletConfig. It reads like the following

```java
public class GuiceServletConfig extends GuiceServletContextListener {

    private ServletContext _ctx;

    @Override
    public void contextDestroyed(final ServletContextEvent servletContextEvent) {
        _ctx = null;
        super.contextDestroyed(servletContextEvent);
    }

    @Override
    public void contextInitialized(final ServletContextEvent servletContextEvent) {
        _ctx = servletContextEvent.getServletContext();
        super.contextInitialized(servletContextEvent);
    }

    @Override
    protected Injector getInjector() {
        return Guice.createInjector(new GuiceServerModule(), new DispatchServletModule());
    }

}
```

This code is borrowed from [Hupa](http://james.apache.org/hupa/project-info.html). The responsibility of this servlet context listener is to construct an injector (achieved by the last method). Here, our injector contains two modules, `ratechecker.server.guice.GuiceServerModule` and `ratechecker.server.guice.DispatchServletModule`.

```java DispatchServletModule.java
public class DispatchServletModule extends ServletModule {
    @Override
    protected void configureServlets() {
        super.configureServlets();
        serve("/ratechecker/dispatch").with(RateCheckerDispatchServlet.class);
    }
}
```

This module has a mapping of URIs and its serving classes. It serves "/ratechecker/dispatch" with `RateCheckerDispatchServlet`, which is the entry point for GWT-dispatch.

```java GuiceServerModule.java
public class GuiceServerModule extends ActionHandlerModule {

    public GuiceServerModule() {
    }

    @Override
    protected void configureHandlers() {
            // declare bindings
    }

}
```


This is where you declare your bindings for the application. We'll come back to this file frequently as the application develops.

```java RateCheckerDispatchServlet.java
@Singleton
public class RateCheckerDispatchServlet extends DispatchServiceServlet {

    private static final long serialVersionUID = 4895255235709260169L;

    private final Log _logger;

    @Inject
    public RateCheckerDispatchServlet(final Dispatch dispatch, final Log logger) {
        super(dispatch);
        _logger = logger;
    }

    @Override
    public Result execute(final Action<?> action) throws ActionException {
        try {
            _logger.info("executing: " + action.getClass().getName());
            final Result res = super.execute(action);
            _logger.info("finished: " + action.getClass().getName());
            return res;
        } catch (final ActionException ae) {
            _logger.error(ae.getMessage());
            ae.printStackTrace();
            throw ae;
        } catch (final Exception e) {
            _logger.error("Unexpected exception: " + e.getMessage());
            e.printStackTrace();
        }
        return null;
    }
}
```


This servlet extends from GWT-dispatch's DispatchServiceServlet. It's main responsibility is to provide unified logging.

Notice you cannot run the application, because Guice is complaining that there's no binding for `org.apache.commons.logging.Log`, which we declared as a dependency for RateCheckerDispatchServlet. We go ahead write our `LogProvider` (to provide lazy initialization for the Log object to its users)

```java LogProvider.java
public class LogProvider implements Provider<Log> {

    @Override
    public Log get() {
        return new Log4JLogger("RateCheckerLogger");
    }

}
```

Now the binding should be added to `GuiceServerModule`:
```java GuiceServerModule.java
[...]
    @Override
    protected void configureHandlers() {
            bind(Log.class).toProvider(LogProvider.class).in(Singleton.class);
    }
[...]
```

Now everytime Guice sees `Log.class` declared as a dependency in the constructor, it uses `LogProvider.get()` method to retrieve an instance of the log if there's none, and uses the existing log instance if it's been initialized (because of the singleton scope).

In the end, your server package should look like this:
<a href="http://reminiscential.files.wordpress.com/2010/03/screenshot-1.png"><img src="http://reminiscential.files.wordpress.com/2010/03/screenshot-1.png?w=300" alt="" title="Screenshot-1" width="300" height="154" class="aligncenter size-medium wp-image-159" /></a>

We haven't covered PersistenceManagerProvider but it's the same idea as LogProvider. It provides an instance of PersistenceManager, which is used by the data store related action handlers to deal with data persistence.

That's it for server wiring. In the next blog post, I'll go through designing and writing GWT-RPC services using Guice and GWT-dispatch.

