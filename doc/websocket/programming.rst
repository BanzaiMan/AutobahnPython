Programming with WebSocket
==========================

This guide introduces WebSocket programming with |Ab|. You'll see how to create WebSocket server and client applications:

1. :ref:`creating-websocket-servers`
2. :ref:`creating-websocket-clients`

Related Information:

* :ref:`WebSocket Examples <websocket_examples>`


.. _creating-websocket-servers:

Creating Servers
----------------

Using |ab| you can create WebSocket servers that will be able to talk to any (compliant) WebSocket client, including browsers.

We'll cover how to define the behavior of your WebSocket server by writing protocol classes and show some boilerplate for actually running a WebSocket server using the behavior defined in the server protocol.


Server Protocols
~~~~~~~~~~~~~~~~

To create a WebSocket server, you need to write a protocol class to **specify the behavior** of the server. 

For example, here is a protocol for a WebSocket echo server that will simply echo back any WebSocket message it receives:

.. code-block:: python

   class EchoServerProtocol(WebSocketServerProtocol):

      def onMessage(self, payload, isBinary):
         ## echo back message verbatim
         self.sendMessage(payload, isBinary)

This is just three lines of code, but we will go through each one carefully, since writing protocol classes like above really is core to WebSocket programming using |ab|.

----------

**1. Protocol Base Class: either Twisted or asyncio**

The **first thing** to note is that you **derive** your protocol class from a base class provided by |ab|.

Depending on whether you write a Twisted or a asyncio based application, here are the base classes to derive from:

* :class:`autobahn.twisted.websocket.WebSocketServerProtocol`
* :class:`autobahn.asyncio.websocket.WebSocketServerProtocol`

So a Twisted-based echo protocol would import the base protocol from ``autobahn.twisted.websocket`` and derive from :class:`autobahn.twisted.websocket.WebSocketServerProtocol`

*Twisted:*

.. code-block:: python

   from autobahn.twisted.websocket import WebSocketServerProtocol

   class EchoServerProtocol(WebSocketServerProtocol):

      def onMessage(self, payload, isBinary):
         ## echo back message verbatim
         self.sendMessage(payload, isBinary)

while an asyncio echo protocol would import the base protocol from ``autobahn.asyncio.websocket`` and dervice from :class:`autobahn.asyncio.websocket.WebSocketServerProtocol`

*asyncio:*

.. code-block:: python

   from autobahn.asyncio.websocket import WebSocketServerProtocol

   class EchoServerProtocol(WebSocketServerProtocol):

      def onMessage(self, payload, isBinary):
         ## echo back message verbatim
         self.sendMessage(payload, isBinary)

.. note:: In this example, only the imports differs between the Twisted and the asyncio variant. The rest of the code is identical. However, in most real world programs you probably won't be able to or don't want to avoid using network framework specific code.

----------

**2. WebSocket Callbacks - receiving messages**

The **second thing** to note is that we **override a hook** ``onMessage`` which is called by |ab| whenever the hook related event happens.

In case of ``onMessage``, the hook (or callback) will be called whenever a new WebSocket message was received. We will come back to WebSocket related callbacks later, but for now, the ``onMessage`` hook is all we need.

When our server receives a WebSocket message, the :meth:`autobahn.websocket.interfaces.IWebSocketChannel.onMessage` will fire with the message ``payload`` received.

The ``payload`` is always a Python byte string. Since WebSocket is able to transmit **text** (UTF8) and **binary** payload, the actual payload type is signaled via the ``isBinary`` flag.

When the ``payload`` is **text** (``isBinary == False``), the bytes received will be an UTF8 encoded string. To process **text** payloads, the first thing you often will do is decoding the UTF8 payload into a Python string:

.. code-block:: python

   s = payload.decode('utf8')

.. note::

   You don't need to validate the bytes for actually being valid UTF8 - |ab| does that already when receiving the message.

----------

**3. WebSocket Methods - sending messages**

The **third thing** to note is that we **use methods** like ``sendMessage`` provided by the base class to perform WebSocket related actions, like sending a WebSocket message.

As there are more methods for performing other actions (like closing the connection), we'll come back to this later, but for now, the ``sendMessage`` method is all we need.

:meth:`autobahn.websocket.interfaces.IWebSocketChannel.sendMessage` takes the ``payload`` to send in a WebSocket message as Python bytes. Since WebSocket is able to transmit payloads of **text** (UTF8) and **binary** type, you need to tell |ab| the actual type of the ``payload`` bytes. This is done using the ``isBinary`` flag.

Hence, to send a WebSocket text message, you will usually *encode* the payload to UTF8:

.. code-block:: python

   payload = s.encode('utf8')
   self.sendMessage(payload, isBinary = False)

.. note::

   |ab| will NOT validate the bytes of a text ``payload`` for actually being valid UTF8. You MUST ensure that you only provide valid UTF8 when sending text messages. If you produce invalid UTF8, a conforming WebSocket peer will close the WebSocket connection due to the protocol violation.

----------


Running a Server
~~~~~~~~~~~~~~~~

Now that we have defined the behavior of our WebSocket server, we need to actually start one that listens on a specific TCP port.

Here is one way of doing that when using Twisted

*Twisted:*

.. code-block:: python
   :emphasize-lines: 9-11

   if __name__ == '__main__':

      import sys

      from twisted.python import log
      from twisted.internet import reactor
      log.startLogging(sys.stdout)

      from autobahn.twisted.websocket import WebSocketServerFactory
      factory = WebSocketServerFactory()
      factory.protocol = EchoServerProtocol

      reactor.listenTCP(9000, factory)
      reactor.run()

What we are doing here is

1. Setup Twisted logging
2. Create a ``WebSocketServerFactory`` factory and set our ``EchoServerProtocol`` on the factory (the highlighted lines)
3. Start a server using the factory, listening on TCP port 9000

Similar, here is the asyncio way

*asyncio:*

.. code-block:: python
   :emphasize-lines: 9-11

   if __name__ == '__main__':

      try:
         import asyncio
      except ImportError:
         ## Trollius >= 0.3 was renamed
         import trollius as asyncio

      from autobahn.asyncio.websocket import WebSocketServerFactory
      factory = WebSocketServerFactory()
      factory.protocol = EchoServerProtocol

      loop = asyncio.get_event_loop()
      coro = loop.create_server(factory, '127.0.0.1', 9000)
      server = loop.run_until_complete(coro)

      try:
         loop.run_forever()
      except KeyboardInterrupt:
         pass
      finally:
         server.close()
         loop.close()

As can be seen, the boilerplate to create and run a server differ from Twisted, but again, the core code of creating a factory and setting our protocol (the highlighted lines) are identical (other than the differing import for the WebSocket factory).

You can find complete code for above examples here:

* `WebSocket Echo (Twisted-based) <https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/websocket/echo>`_
* `WebSocket Echo (Asyncio-based) <https://github.com/tavendo/AutobahnPython/tree/master/examples/asyncio/websocket/echo>`_


WebSocket Callbacks
-------------------

As we have seen above, |ab| will fire *callbacks* on your protocol class whenever the event related to the respective hook occurs.

It it in these hooks that you will implement application specific code.

The core WebSocket interface :class:`autobahn.websocket.interfaces.IWebSocketChannel` provides the following *callbacks*:

* :meth:`autobahn.websocket.interfaces.IWebSocketChannel.onConnect`
* :meth:`autobahn.websocket.interfaces.IWebSocketChannel.onOpen`
* :meth:`autobahn.websocket.interfaces.IWebSocketChannel.onMessage`
* :meth:`autobahn.websocket.interfaces.IWebSocketChannel.onClose`

**onConnect**

Whenever a new client connects to the server, a new protocol instance will be created and the :meth:`autobahn.websocket.interfaces.IWebSocketChannel.onConnect` hook fires as soon as the WebSocket opening handshake is begun by the client.

In this hook you can do thing like

* checking or setting cookies or other HTTP headers
* verifying the client IP address
* checking the origin of the WebSocket request
* negotiate WebSocket subprotocols

For a WebSocket server protocol, ``onConnect()`` will fire with 
:class:`autobahn.websocket.protocol.ConnectionRequest` providing information on the client wishing to connect via WebSocket.

On the other hand, for a WebSocket client protocol, ``onConnect()`` will fire with 
:class:`autobahn.websocket.protocol.ConnectionResponse` providing information on the WebSocket connection that was accepted by the server.

For example, a WebSocket client might offer to speak several WebSocket subprotocols. The server can inspect the offered protocols in ``onConnect()`` via the supplied instance of :class:`autobahn.websocket.protocol.ConnectionRequest`. When the server accepts the client, it'll chose one of the offered subprotocols. The client can then inspect the selectec subprotocol in it's ``onConnect()`` hook in the supplied instance of :class:`autobahn.websocket.protocol.ConnectionResponse`.


**onOpen**

The :meth:`autobahn.websocket.interfaces.IWebSocketChannel.onOpen` hook fires when the WebSocket opening handshake has been successfully completed. You now can send and receive messages over the connection.

**onClose**

When the WebSocket connection has closed, the :meth:`autobahn.websocket.interfaces.IWebSocketChannel.onClose` fires. From now on, no messages will be received anymore and you cannot send messages also. The protocol instance won't be reused. It'll be garbage collected. When the client reconnects, a completely new protocol instance will be created.

**onMessage**

In any case, the :meth:`autobahn.websocket.interfaces.IWebSocketChannel.onMessage` hook is the most important. It is here where you implement what should happen when a new (incoming) WebSocket message was received.

Here is an example that overrides all of above callbacks:

.. code-block:: python

   class MyServerProtocol(WebSocketServerProtocol):

      def onConnect(self, request):
         print("Client connecting: {}".format(request.peer))

      def onOpen(self):
         print("WebSocket connection open.")

      def onMessage(self, payload, isBinary):
         if isBinary:
            print("Binary message received: {} bytes".format(len(payload)))
         else:
            print("Text message received: {}".format(payload.decode('utf8')))

         ## echo back message verbatim
         self.sendMessage(payload, isBinary)

      def onClose(self, wasClean, code, reason):
         print("WebSocket connection closed: {}".format(reason))


WebSocket Methods
-----------------

The core WebSocket interface :class:`autobahn.websocket.interfaces.IWebSocketChannel` provides the following *methods*:

* :meth:`autobahn.websocket.interfaces.IWebSocketChannel.sendMessage`
* :meth:`autobahn.websocket.interfaces.IWebSocketChannel.sendClose`

**sendMessage**

We have already seen the :meth:`autobahn.websocket.interfaces.IWebSocketChannel.sendMessage` method for sending WebSocket messages.

**sendClose**

The :meth:`autobahn.websocket.interfaces.IWebSocketChannel.sendClose` will initiate a WebSocket closing handshake. After starting to close a WebSocket connection, no messages can be sent. Eventually, the :meth:`autobahn.websocket.interfaces.IWebSocketChannel.onClose` hook will fire.

After a WebSocket connection has been closed, the protocol instance will get recycled. Should the client reconnect, a new protocol instance will be created and a new WebSocket opening handshake performed.


.. _creating-websocket-clients:

Creating Clients
----------------

Write me.



Upgrading
---------

Starting with release 0.7.0, |ab| now supports both Twisted and asyncio as the underlying network library. This required changing module naming, e.g.

|ab| **< 0.7.0**:

.. code-block:: python

     from autobahn.websocket import WebSocketServerProtocol

|ab| **>= 0.7.0**:


.. code-block:: python

     from autobahn.twisted.websocket import WebSocketServerProtocol

or

.. code-block:: python

     from autobahn.asyncio.websocket import WebSocketServerProtocol

Two more small changes (also see the `interface definition <https://github.com/tavendo/AutobahnPython/blob/master/autobahn/autobahn/websocket/interfaces.py>`_ now available):

1. ``WebSocketProtocol.sendMessage``: renaming of parameter ``binary`` to ``isBinary`` (for consistency with `onMessage`)
2. ``ConnectionRequest`` no longer provides ``peerstr``, but only ``peer``, and the latter is a plain, descriptive string (this was needed since we now support both Twisted and asyncio, and also non-TCP transports)
