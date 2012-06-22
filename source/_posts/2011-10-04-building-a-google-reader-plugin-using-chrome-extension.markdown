---
layout: post
title: "Building a Google Reader plugin using Chrome extension"
date: 2011-10-04 20:14
comments: true
categories: javascript, chrome extension, google reader
---

OK, ok, I understand. The title is a bit misleading. [Google Reader](http://reader.google.com) isn't open for 3rd party plugins, and there's no indication that Google will ever. However, with Google Chrome extension, we can build such local "plugins".

## What are we going to achieve?

Anyone uses Google Reader to read [DZone](http://www.dzone.com) feeds? I do. DZone is a very good tech news aggregator and you can vote and comment on stories. With Google Reader, you get DZone feeds like the following. I don't know about you but for me, sometimes I just want to read the original story without going to the DZone page. It'd be nice if they have a "click through" action (like the following) on the action bar that brings you to the original story.

<a href="http://reminiscential.files.wordpress.com/2011/10/screenshot-google-reader-539-chromium.png"><img src="http://reminiscential.files.wordpress.com/2011/10/screenshot-google-reader-539-chromium.png?w=300" alt="End goal" title="Screenshot-Google Reader (539) - Chromium" width="300" height="100" class="aligncenter size-medium wp-image-262" /></a>

## Basic strategy

So, how are we going to implement this feature? Chrome Extension code can be injected into the running page, and have full access to its DOM. Therefore, we can write code such that if the currently opened entry is a DZone entry, we insert 'Click Through' action into the entry action bar (action bar is the bar underneath the main entry, where 'Add Star', 'Like', 'Share' actions are). The 'Click Through' action, when clicked, will read the feed URL, fetch it in the background, parse it and get the URL of the original story, and open the original URL in a separate tab.

<a href="http://reminiscential.files.wordpress.com/2011/10/untitled-1.png"><img class="aligncenter size-medium wp-image-261" title="strategy" src="http://reminiscential.files.wordpress.com/2011/10/untitled-1.png?w=300" alt="" width="300" height="231" /></a>

### Create a manifest

A Chrome extension must have a manifest.json file containing the metadata of the extension.

```javascript
{
    "name":"GReader",
    "version":"1.0",
    "description":"Enhanced Google Reader experience",
    "permissions": [
        "http://*.dzone.com/"
    ],
    "background_page":"background.html",
    "content_scripts":[{
        "matches":[
            "*://www.google.com/reader/view/*"
        ],
        "js":[
            "lib/jquery-1.6.4.min.js",
            "src/greader.js"
        ],
        "run_at":"document_idle"
    }]
}
```

Here we

* specify that the extension needs to access any URL on dzone.com including its subdomains.
* specify the background page
* specify the content script

The "run_at" property will dictate when the content script is going to be run. Because Google Reader is a full AJAX application, we want our script to be run when the document is fully rendered.

We also specify the "matches" property, so our content script is only activated when the URL matches.

### The content script
We start with:
```javascript
(function($) {

})(jQuery);
```

This creates a function scope, which separates our `$` variable apart from the current page's `$` variable. Google Reader (I assume, is using Google's own closure library), already defines $ and it's not the jQuery object. This idiom gives `$` as jQuery.

We want to insert the "Click through" action in the entry action bar. To achieve this, we will need to listen on `DOMNodeInserted` event, and when such event happens and the node inserted is of the right CSS class name ("entry-action" here), we proceed to manipulate the DOM to add our customized actions.

```javascript
    $("#entries").live('DOMNodeInserted', function(e) {
        if (!e.target.className.match(/entry\-actions/))
            return;

        var entryAction = new EntryAction($(e.target));
        if (entryAction.entry.url.match(/^http\:\/\/feeds\.dzone\.com/)) {
            entryAction.addAction({
                'name':'Click Through',
                'fn':function(entry) {
                    chrome.extension.sendRequest({"type":"fetch_entry", "url":entry.url}, function(response) {
                        var matched = /<div class="ldTitle">(.*?)<\/div>/.exec(response.data);
                        var href = ($(matched[1]).attr("href"));
                        if (href !== null) {
                            chrome.extension.sendRequest({"type":"open_tab", "url":href}, function(response) {
                                // TODO: do something afterwards?
                            });
                        }
                    });
                }
            });
        }
    });
```

Here I built a little bit of abstraction around the raw entry action bar. It's encapsulated in EntryAction class, which I'll show in a moment. Basically, if the current displaying entry's feed URL starts with feed.dzone.com, I'll build the "click through" action, and set the click handler. It sends the feed URL to the background script. The background script will do the cross-site request to fetch the feed content and send it back. Then the content script will regex match the content to get the original story's URL, and ask chrome to open the URL in a new tab.

Here's the code for `EntryAction`:
```javascript
    var EntryAction = function(element) {
        this.element = element;
        var entryElmt = this.element.parent(".entry");
        var url = $(entryElmt).find(".entry-title-link").attr('href');
        this.entry = {
            "url" : url
        };
    };

    EntryAction.prototype.addAction = function(action) {
        var that = this;
        var onclick = function(e) {
            var actionFunc = action['fn'];
            actionFunc(that.entry);
        }

        this.element.append($("<span>")
            .addClass("link unselectable")
            .text(action['name'])
            .click(onclick));
    };
```

I won't delve too much into this code. It makes assumptions about the structure of the DOM that Google Reader renders into. This does make the extension brittle but that's the reality we have to deal with for client-side scripting. Luckily, Google Reader markup doesn't change very often. For people new to object-oriented Javascript, this is one way to create a "class" (prototype) and put "instance" methods on a "class".

### Background.html
Unlike content scripts, which is injected and runs in the target page, the background page runs in its own process (the extension's process) and keeps running while the extension is active. It's comparable to the "server" side of the extension. For our extension's purpose, we're using the background script to make requests to 3rd party web sites (DZone).

```html
<html>
<script type="text/javascript" src="lib/jquery-1.6.4.min.js"></script>
<script>
    (function($) {
        chrome.extension.onRequest.addListener(
            function(request, sender, sendResponse) {
                if (request.type === 'fetch_entry') {
                    $.get(request.url, function(data) {
                        sendResponse({"data":data});
                    });
                } else if (request.type === 'open_tab') {
                    chrome.tabs.create({'url':request.url});
                    sendResponse({"status":"ok"});
                }
            }
        );
    })(jQuery);
</script>
</html>
```

We register handlers for events that our "client" (the content script) is able to raise. Here we deal with 2 kinds of events: fetch_entry and open_tab.

* Although Chrome 13 allows cross-site requests from content scripts, I'm actually quite fond of this pattern of delegating requests to the background page.
* chrome.tabs isn't accessible in the content script. That's why open_tab is an event the client (the content script) can raise and delegate chrome specific API calls to the background script.

## After Thoughts
That's it! That's my first Chrome extension. It's not earth shattering or anything but I learned quite a lot. I like Chrome extension development -- it's straightforward and simple. The architecture is quite simple yet powerful. The code is on [Github](https://github.com/kevinjqiu/greader) and I plan to expand it to a framework for customizing Google Reader experience. Here are a few things we can do with the extension:

* Link "Share" action to twitter/Google+
* Click on "Like" action to automatically vote up on DZone (or any other news aggregator)
* Share with comment on Google Reader puts the comment on the entry on DZone (or any other news aggregator)
* endless opportunities...

