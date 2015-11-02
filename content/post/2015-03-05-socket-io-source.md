---
layout: post
title: Tried to read socket.io source code
category: [tech]
tags: [javascript, socket.io]
---
Some important parts of socket.io.
<!--more-->

### Manager
This is the main interface of socket.io. ```require('socket.io').listen()``` returns a Manager (the io object).

Manager holds a server, which is the instance of nodejs server; a namespaces, which is the collections of all namespaces; a settings, which holds the customizable setting info. a sockets, which is the default namespace.

#### ```new exports.Manager(server, options)``` workflow:

- init all the members including server, namespaces, sockets, settings.
- ```server.on('request'...``` use ```handleRequest``` to handle server request.
- sockets is a bad name, because is actually refers to "" namespace, as ```this.sockets = this.of('')``` shows.
- init store
  - here the store subscribes lots of messages like handshaken, connect, open, join, leave, etc. and call the corresponding method
  - init ```handshaken```, ```connected```, ```open```, ```closed```, ```rooms```, ```roomClients```. These are important controll objects.
- init transports, transports are underlying data transporation modules.



### handshake
The first step of websocket is shandshake. client inits a http request inicating itself as a websocket request, the server indentifies it and start websocket communication.
Here is how it works in socket.io:

- handleRequest: when request comes, this method is called. In this method, ```checkRequest``` is used to extract messsages from rquest data. When request data **doesn't** have an id, then it is considered as a handshake, and handleHandshake will be called.
- handleHandshake: This method first extracts handshake data from request data with ```handshakeData``` method.
  - authorize method can be used to authoriztion. authoriztion method can be passed from outside with set method.
  - If authorization succeeded, then success is returned. Here the unique id for this connection is created, and be used everywhere.
  - onHandshake: It is called after response, sets handshaken object with id as key, and handshake data as value.
  - Also, ```handshake``` message is published.
  - Now connection is established.


### send and receive message
To receive messages from client, the workflow is the same as handshake, except that ```handleHTTPRequest``` is used instead of ```handleHandshake```.

#### handleHTTPRequest
```handleClient``` will do the actual work.

#### handleClient
##### why onHandshake for second time here??
Because when client get disconnected, it will try to reconnect for sometime, and will enter handleClient.

Important parts are:
Here is where connect event of client is fired:
```javascript
if (i === '') {
  this.namespaces[i].handlePacket(data.id, { type: 'connect' });
}
```

Here is how message of specific connection can be listend:
```js
this.store.subscribe('message:' + data.id, function (packet) {
   self.onClientMessage(data.id, packet);
});
```
I think we can use this to communicate with connection from outside of nodejs. For example, when redis store is used, we can publish ```message:111``` from other program, say a php batch, to notify something to the connection with id 111. **Update: too simple, sometimes naive**: After a close look at the code, I found this is difficut as the data of message is the packet itself. 

The real code to send packet back to user is done by namespaces.

### SocketNamespace
The packet is handled by ```handlePacket``` method.
The important member is ```socket```, which is the socket we got in ```io.sockets.on('connection', funtion(socket) {}```. socket.io has one socket for each connection identified by id. In handlePacket method, itdoes different things based on ```pacekt.type```. Here we can find why ```socket.on(xxx, function() {})``` can work, in fact it calls ```socket.$emit.apply(socket, params)```.

#### connect
In ```packet``` method, it does different things based on packet.type. When type is connect, then connection event is emitted, and our ```io.sockets.on('connection')``` will be called. Also here, we will call ```socket.packet({ type: 'connect'})``` to send connect packet back to client, so that client's 'on connect' will be called.

**TODO** ack

The ```packet``` method of SocketNamespace calls Manager's onDispath which in turn calls transport's onDispatch to send the packet.
### Transport
Transport is where the transition of data is carried out. We only focus on websocket. There're several subclass represents different sub protocal of websocket, the ```onDispatch``` called write method of its sub class. I only searched in default.js. In default.js, there's a ```write``` method finally writes data buffer into socket.







