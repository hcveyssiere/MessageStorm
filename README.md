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

This is the base message of the protocol. This message is used by the server to send orders to clients. `poke` messages are composed, at least of these fields:
* `id` field, that identifies the message (and will let the server compute RTT statistics)
* `params` field, that carries details about the actions that the client must take when it receives this message