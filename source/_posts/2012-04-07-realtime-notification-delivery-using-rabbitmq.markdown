---
layout: post
title: "Realtime notification delivery using rabbitmq"
date: 2012-04-07 16:50
comments: true
categories: websocket, rabbitmq, python
---

Our company has "hack-off" days once a while, where we developers get to choose whatever we would like to work on and present it to the entire company by the end of the day. I have been hearing this [websocket](http://en.wikipedia.org/wiki/WebSocket) buzz for a while now and would like to build something interesting with it.

## WebSocket

Websocket is a persistent bi-directional connection between the browser and the server. With websocket, web browser can post message to the server, but what's more interesting is that the server is able to push messages to the client (browser). This breaks away from the traditional web application request/response model. Traditionally, the client makes the request and waits for the server to give an answer. AJAX is revolutionary, but essentially, it's still the same model: the client asks the server whether there's anything interesting, but not the other way around. With websocket, the server suddenly becomes more involved and able to deliver more engaged user experience.

[Our company](http://www.freshbooks.com/) provides web application for online invoicing. The web application allows users to create clients, create invoices, send them to clients, and so on. Each one of these are "events" which gets sent to [RabbitMQ](http://www.rabbitmq.com/). We then have a plethora of RabbitMQ consumers that read messages off the queue and do interesting stuff with them.

## Proof of concept

For this hack-off, my goal is to write a RabbitMQ consumer that reads the messages off the message queue, and deliver (notify) them to the front-end using websocket.

<a href="http://reminiscential.files.wordpress.com/2012/04/websocket-1.png"><img src="http://reminiscential.files.wordpress.com/2012/04/websocket-1.png?w=300" alt="" title="architecture" width="300" height="181" class="aligncenter size-medium wp-image-292" /></a>

I've heard good things about [Tornado](http://www.tornadoweb.org). Having read their docs on [websocket request handler](http://www.tornadoweb.org/documentation/websocket.html), I felt it's straightforward enough for me, so I chose Tornado as my backend.

## Pika

One problem arises, though: The tornado server will run as a regular server, waiting for incoming websocket connections. The RabbitMQ consumer also needs to be in the same process event loop, waiting for incoming messages from the message queue. I looked into a few solutions such as [sparkplug](http://pypi.python.org/pypi/sparkplug/) and [stormed-amqp](http://pypi.python.org/pypi/stormed-amqp/0.1), neither seem to be a good hit here. Finally, I stumbled on [Pika](https://github.com/pika/pika). It comes with a Tornado event loop adapter, which allows rabbitmq consumer and websocket handlers to run inside the same event loop. Perfect.

The entry point looks like this:

```python
application = tornado.web.Application([
    (r'/ws', handlers.MyWebSocketHandler),
])

def main():
    pika.log.setup(color=True)

    io_loop = tornado.ioloop.IOLoop.instance()

    # PikaClient is our rabbitmq consumer
    pc = client.PikaClient(io_loop)
    application.pc = pc
    application.pc.connect()

    application.listen(8888)
    io_loop.start()
```

```python
class MyWebSocketHandler(tornado.websocket.WebSocketHandler):

    def open(self, *args, **kwargs):
        pika.log.info("WebSocket opened")

    def on_close(self):
        pika.log.info("WebSocket closed")
```

That was straightforward. However, I'm faced with the problem of how to make the amqp consumer notify websocket handlers when we receive a message from the message queue. We cannot get the handler instances from the tornado application object. Note, each websocket connection has a corresponding `MyWebSocketHandler` instance. The instances are not available from the application object. Maybe there's a way to get them by other means, but I'm not familiar with the tornado API enough to know that.

However, from the handler, we do get the `application` object, and because we attached pika_client (our amqp consumer) to the application, we have access to it inside our socket handler. Hey, how about registering the handler with the client when the websocket is connected, and let the client "notify" the handler when events are received? Hey, isn't that the [observer pattern](http://en.wikipedia.org/wiki/Observer_pattern)?

Here's the code:

```python
class MyWebSocketHandler(websocket.WebSocketHandler):

    def open(self, *args, **kwargs):
        self.application.pc.add_event_listener(self)
        pika.log.info("WebSocket opened")

    def on_close(self):
        pika.log.info("WebSocket closed")
        self.application.pc.remove_event_listener(self)
```

Now, our `PikaClient` object need to support `add_event_listener()` and `remove_event_listener()` methods.

```python
class PikaClient(object):

    def __init__(self, io_loop):
        pika.log.info('PikaClient: __init__')
        self.io_loop = io_loop

        self.connected = False
        self.connecting = False
        self.connection = None
        self.channel = None

        self.event_listeners = set([])

    def connect(self):
        if self.connecting:
            pika.log.info('PikaClient: Already connecting to RabbitMQ')
            return

        pika.log.info('PikaClient: Connecting to RabbitMQ')
        self.connecting = True

        cred = pika.PlainCredentials('guest', 'guest')
        param = pika.ConnectionParameters(
            host='localhost',
            port=5672,
            virtual_host='/',
            credentials=cred
        )

        self.connection = TornadoConnection(param,
            on_open_callback=self.on_connected)
        self.connection.add_on_close_callback(self.on_closed)

    def on_connected(self, connection):
        pika.log.info('PikaClient: connected to RabbitMQ')
        self.connected = True
        self.connection = connection
        self.connection.channel(self.on_channel_open)

    def on_channel_open(self, channel):
        pika.log.info('PikaClient: Channel open, Declaring exchange')
        self.channel = channel
        # declare exchanges, which in turn, declare
        # queues, and bind exchange to queues

    def on_closed(self, connection):
        pika.log.info('PikaClient: rabbit connection closed')
        self.io_loop.stop()

    def on_message(self, channel, method, header, body):
        pika.log.info('PikaClient: message received: %s' % body)
        self.notify_listeners(event_factory(body))

    def notify_listeners(self, event_obj):
        # here we assume the message the sourcing app
        # post to the message queue is in JSON format
        event_json = json.dumps(event_ostener in self.event_listeners:
            listener.write_message(event_json)
            pika.log.info('PikaClient: notified %s' % repr(listener))

    def add_event_listener(self, listener):
        self.event_listeners.add(listener)
        pika.log.info('PikaClient: listener %s added' % repr(listener))

    def remove_event_listener(self, listener):
        try:
            self.event_listeners.remove(listener)
            pika.log.info('PikaClient: listener %s removed' % repr(listener))
        except KeyError:
            pass
```

I left out the queue setup code here for brevity. `on_message` callback is called when the consumer gets a message from the queue. The client, in turn, notifies all registered websocket handlers. Obviously, in real applications, you may want to do some kind of credentials and filtering, so the right message get to the right receiver. Then we simply call `handler.write_message()`, so the message gets relayed to the front-end's websocket.onmessage callback.

Here's some front-end code:

```javascript
(function($){
    $(document).ready(function() {
        var ws = new WebSocket('ws://localhost:8888/ws');
        ws.onmessage = function(evt){
            alert(evt.data);
        }
    });
})(jQuery);
```

Yes, we simply echo the message back. For the hackoff, I did parse the data, render a slightly more detailed notification message, and display the notification using jquery-toaster.

## Conclusion

This is my first stab at websocket and the tornado web framework. I'm not an expert on either subject, so chances are there are better ways to achieve the same result.

I think websocket is a very interesting technology. It opens a wide range of possibilities for more interactive and engaging web applications. Our web application is of traditional architecture: server renders most of the page, and every request involves page loads. Having a websocket may not be very beneficial as the application doesn't have that much of user interaction. My hackoff is more of a proof of concept. However, if the application is a one-page web app (no full page reloads), the websocket model works very well. 
