---
layout: post
title: "Integrating SmartGWT with ASP.NET JSON web service"
date: 2009-03-19 09:08
comments: true
categories: java, gwt
---

Many web API authors are using third party libraries like JayRock to convert ASP.NET web service method return values to JSON.  ASP.NET does have the ability to return JSON objects for web methods.  It’s not a very well-known feature (I guess) as there are only a few [places](http://encosia.com/2008/03/27/using-jquery-to-consume-aspnet-json-web-services/) mentioning it. 

For example, suppose the web method we are calling is `/ItemService.asmx/GetList`, and it’s expecting a parameter “uid”. For JSON web service to work, we have to set the request content type to “application/json” and provide the request parameter as a JSON object and send it through HTTP POST data (`{uid:'12345'}`).

In the current project I'm working on, I'm hooking up [SmartGWT](http://code.google.com/p/smartgwt/)'s Data source to use JSON web service of ASP.NET. SmartGWT’s data source allows OperationBinding to be set for each of the four CRUD operations. For ItemDS, my fetch operation looks like the following:

```java
    OperationBinding createFetchOperation() {

        final OperationBinding retval = new OperationBinding(DSOperationType.FETCH, Urls.ISSUE_FETCH);
        retval.setDataFormat(DSDataFormat.JSON);

        retval.setDataProtocol(DSProtocol.POSTPARAMS);

        retval.setRecordXPath("/result");

        return retval;

    }
```

Since we're doing a bunch of customized stuff, we’re bound to override `transformRequest()` method in the `DataSource` class:

```java
class ItemDS extends RestDataSource {
    /* data source setup */
    @Override
    protected void transformRequest(DSRequest req) {
        // TODO
    }
}
```

We need to do two things in transformRequest:
1. set request header to `application/json; charset=utf-8`
2. set the request payload to be the JSON object that contains parameters to the web service
So here we go:

```java
protected void transformRequest(DSRequest req) {
    dsRequest.setContentType("application/json; charset=utf-8");
    final JavaScriptObject params = JSOHelper.createObject();
    JSOHelper.setAttribute(params, "uid", "12345");
    req.setData(params);
    return req.getData();
}
```

Does that work? A simple test proved that it didn't. Because remember we set data protocol on the fetch operation to `POSTPARAMS`? It turns out that even though `req.data` is sent via POST body, it’s URL form-encoded, not JSON encoded as you would expect since we already set data format to JSON. A bit digging on the SmartGWT forum turned out that data format setting only affects the parsing of the return value, not the outbound parameters.

Okay…let me try again…This time, I’m setting `DSProtocol` to `POSTMESSAGE`, as this seems to be a more logical choice (as the parameters are going to be sent out as POST message/body). Does it work? I wish…From the DevConsole, it turns out that SmartGWT’s RPC Manager sends out “[Object]” in POST body…Whaaat? Looking at SmartGWT’s API, it doesn’t allow setting request data as a simple string. It has to be a `JavaScriptObject` instance. I digged inside SmartGWT’s source code, setting the attribute `data` directly to the JSON object that I want to use, but GWT shell complains it cannot cast from `java.lang.String` to `JavaScriptObject`…Sighhhh

It starts to get frustrating. This evening, as I was going through some JavaScript stuff, it dawns to me that the notion `[Object]` seems to be the string representation of a generic JavaScript object. (i.e., the return of an object’s `toString()` method). Looking at `JavaScriptObject` class in GWT confirms this. So, what if I just override `toString()` method and just return a JSON object?

This is like an epiphany! I went over some articles on how to do GWT JSNI and quickly come up with this:

```java
/* Modify OperationBinding, set to use POSTMESSAGE protocol */
setDataProtocol(DSProtocol.POSTPARAMS);

/* in ItemDS */
static native void setValue(JavaScriptObject jso, String val) /*-{
        jso.toString = function() {
                    return val;
                        }
}-*/;
/* in transformRequest */
/* ... setup content type ... */
final JavaScriptObject data = JSOHelper.createObject();
setValue(data, "{uid:'12345'}");
req.setData(data);
return data;
```

Et voilà!!! The DevConsole shows that RPC manager sends out {uid:12345} as the HTTP POST data and I got right what I want.

