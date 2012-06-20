---
layout: post
title: "Building a GAE+GWT application using the best practices (Part 3)"
date: 2010-03-03 01:02
comments: true
categories: java, gae, gwt
---

## Building a GAE+GWT application using the best practices series
1. [Part 1](/2010/02/26/building-a-gae-plus-gwt-application-using-the-best-practices-part-1/)
2. [Part 2](/2010/03/01/building-a-gae-plus-gwt-application-using-the-best-practices-part-2/)
3. Part 3
4. [Part 4](/2010/03/03/building-a-gae-plus-gwt-application-using-the-best-practices-part-4/)
5. [Part 5](/2010/03/09/building-a-gae-plus-gwt-application-using-the-best-practices-part-5/)

In this part of the series, we're going to explore the designing of the web services for RateChecker and coding them using the command pattern from [GWT-dispatch](http://code.google.com/p/gwt-dispatch/) based on Ray Ryan's [presentation](http://www.youtube.com/watch?v=PDuhR18-EdM).

## The big picture

<a href="http://reminiscential.files.wordpress.com/2010/03/classdiagram.png"><img class="aligncenter size-medium wp-image-166" title="ClassDiagram" src="http://reminiscential.files.wordpress.com/2010/03/classdiagram.png?w=300" alt="" width="300" height="272" /></a>

To correctly implement web services using the command pattern, we first have to get the big picture. There are three "actors" involved in this: an action, a result and a handler.

### Action
An action is used to store the parameters of the web service call (if any). For instance, a CheckRate action needs to know what type of rate the user is checking: buying rate? selling rate? currency?

### Result
A result object stores the result (duh...) of the web service call. In the case of CheckRate method, the result is the rate object containing the details of the rate.

### Handler
A handler is the actual "worker" that actually does the work of checking the rate. In this case, the check rate method fetches the posted rate page , parses the text (if needed) to get the rate information.

### Dispatch and DispatchAsync
A Handler is executed on the server side (by the DispatchServlet we saw in the last post). On the client side, there's a counterpart "DispatchAsync", which is the asynchronous interface that the client code calls.

## Implementing web service methods
Now that we have the big picture in place, we're going to look into how to actually implement them.
The first step is to define a domain model. In this case, it's our Rate class:

```java
@PersistenceCapable(identityType=IdentityType.APPLICATION)
public class Rate implements Serializable {

    private static final long serialVersionUID = -4415279469780082174L;

    @PrimaryKey
    @Persistent(valueStrategy=IdGeneratorStrategy.IDENTITY)
    private Long id;

    @Persistent
    private RateType type;

    @Persistent
    private Date timeFetched;

    @Persistent
    private Double rate;

    public Rate() {
    }
    // ... getters and setters omitted
}
```

```java
public enum RateType {
    Selling,
    Buying,
}
```

In our example application, we are going to define three simple web methods:

* Check rate: use Url fetch to get the posted rate page and return a rate object from that.
* Save rate: persist the rate object into the data store.
* Get rate: get the rates from the data store.

### Check rate

As we have shown in the big picture, every action needs three pieces: action, result (both in shared package, as they will be used by both the client and the server) and the handler (lives in the server package).

For CheckRate action, we need to specify the type of rate it needs to check. For simplicity, I'm always dealing with USD/CAD rate. The parameter here is only for whether to check for buying rate or selling rate.

```java
public class CheckRate implements Action<CheckRateResult> {

    private static final long serialVersionUID = -1716760883016361503L;

    private RateType _type;

    public CheckRate() {
    }

    public CheckRate(final RateType type) {
        _type = type;
    }

    public void setType(final RateType type) {
        _type = type;
    }

    public RateType getType() {
        return _type;
    }

}
```

The result is designed to hold the returned Rate object.

```java
public class CheckRateResult implements Result {

    private static final long serialVersionUID = -9099789297842581458L;

    private Rate _rate;

    public CheckRateResult() {
    }

    public CheckRateResult(final Rate rate) {
        _rate = rate;
    }

    public void setRate(final Rate rate) {
        _rate = rate;
    }

    public Rate getRate() {
        return _rate;
    }
}
```

Word of caution: because both action and result are serialized and sent over the wire as part of GWT-RPC call, they are required to have a default public constructor.

Now, on to the handler:
```java
public class CheckRateHandler implements ActionHandler<CheckRate, CheckRateResult> {

    public static final String URL_BUY = "http://www.ingdirect.ca/en/datafiles/rates/usbuying.html";

    public static final String URL_SELL = "http://www.ingdirect.ca/en/datafiles/rates/usselling.html";

    public CheckRateHandler() {
    }

    @Override
    public CheckRateResult execute(final CheckRate action, final ExecutionContext ctx) throws ActionException {
        final CheckRateResult retval = new CheckRateResult();

        String strUrl = null;
        switch (action.getType()) {
        case Buying:
            strUrl = URL_BUY;
            break;
        case Selling:
            strUrl = URL_SELL;
            break;
        }

        try {
            final URL url = new URL(strUrl);

            BufferedReader br = null;
            try {
                br = new BufferedReader(new InputStreamReader(url.openStream()));

                final double dRate = Double.parseDouble(br.readLine());

                final Rate rate = new Rate();
                rate.setRate(dRate);
                rate.setType(action.getType());
                rate.setTimeFetched(new Date());

                retval.setRate(rate);

            } finally {
                if (br != null)
                    br.close();
            }
        } catch (final MalformedURLException e) {
            e.printStackTrace();
            throw new ActionException(e);
        } catch (final IOException e) {
            e.printStackTrace();
            throw new ActionException(e);
        } catch (final NumberFormatException e) {
            e.printStackTrace();
            throw new ActionException(e);
        }

        return retval;
    }
    // ... other methods omitted
}
```

As you can see, the handler does the actual work of fetching the rate using URL Fetch service offered by Google App Engine.

The other two web method implementations are similar. You can follow the project on Github [here](http://github.com/kevinjqiu/ratechecker). In the next section, I'm going to go over the building of the UI in GWT, as well as making AJAX calls from GWT to the server.

