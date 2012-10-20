MessageStorm
============

MessageStorm is a cloud-based messaging system experiment. This project is a proof of concept which aims at checking the feasibility of a cloud-based messaging system able to handle numerous concurrent clients connections (+1M).

The server applicative is written in node and is supposed to run on a single VM hosted on Amazon cloud (XL instance), publicly available from the Internet (no firewall should be used). It offers websocket connectivity to clients that should access the server from another network (OVH cloud for instance).


This project is roughly composed of two node applications :

* the server application which is responsible for
** offering socket.io connectivity
** routing messages to clients
** offering a basic monitoring API

* a sample client application which is responsible for
** connecting to the server
** logging all incoming messages
** relaying distributed messages to other clients (through the server)


Messaging Protocol
------------------

MessageStorm protocol is quite simple since server <-> client communication, since it relies on two types of message :

### server to client "poke" message #

tzettze