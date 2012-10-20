MessageStorm
============

MessageStorm is a cloud-based messaging system experiment. This project is a proof of concept which aims at checking the feasibility of a cloud-based messaging system able to handle numerous concurrent clients connections (+1M).

The server applicative is written in node and is supposed to run on a single VM hosted on Amazon cloud (XL instance), publicly available from the Internet (no firewall should be used). It offers websocket connectivity to clients that should access the server from another network (OVH cloud for instance).


This project is roughly composed of two node applications:

1. the server application which is responsible for
	* offering socket.io connectivity
	* routing messages to clients
	* offering a basic monitoring API

2. a sample client application which is responsible for
	* connecting to the server
	* logging all incoming messages (for monitoring purpose)
	* relaying distributed messages to other clients (through the server)


Messaging Protocol
------------------

MessageStorm protocol is quite simple since server <-> client communication, since it relies on two types of message:

### Server to client `poke` message

This is the base message of the protocol. This message is used by the server to send orders to clients. `poke` messages are composed of several fields:
* `ts` [mandatory] field, the UNIX timestamp (at which the server sent the message).
* `depth` [optional] field. If this field is present, it must be a power of two. If depth>1, the client has to reply a `pokeBack` message with a depth field equals to depth/2. The server will then randomly picks up two clients and send them a `poke`message with the new depth and so on and so forth. This mecanism simulate a messaging chain to heavily load the system.
* `treeId` [optional] field. This field identifies the tree. The client has to send back this id along with the depth so that the server knows which chain is concerned.

### Client to server `pokeBack` message

This message must be systematically sent by the client each times it receives a `poke` message. Here are the fields that this message can contain:
* `delay` [mandatory] field. This field is the difference between the ts sent by the server and the local (client) UNIX timestamp. We assume that server and clients are syncronized with the same NTP server.
* `serverTs` [mandatory] field. This field is the ts given by the server (it should be sent back by the client so that the server can compute message RTT more accurately)