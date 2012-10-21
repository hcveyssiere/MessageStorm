MessageStorm
============

MessageStorm is a cloud-based messaging system experiment. This project is a proof of concept which aims at checking the feasibility of a cloud-based messaging system able to handle numerous concurrent clients connections (+1M).

The server applicative is written in node and is supposed to run on a single VM hosted on Amazon cloud (XL instance), publicly available from the Internet (no firewall should be used). It offers websocket connectivity to clients that should access the server from another network (OVH cloud for instance).


This project is roughly composed of two node applications:

* the server application which is responsible for
	* offering socket.io connectivity
	* routing messages to clients (implementing Messaging Protocol)
	* offering a basic monitoring API (implementing Monitoring Protocol)
* a sample client application which is responsible for
	* connecting to the server
	* logging all incoming messages (for monitoring purpose)
	* relaying distributed messages to other clients (through the server)


Messaging Protocol
------------------

MessageStorm protocol is quite simple since server <-> client communication relies on two types of messages:

### Server to client `poke` message

This is the base message of the protocol. This message is used by the server to send orders to clients. `poke` messages are composed of several fields:
* `id` [mandatory] field, that identifies the message (and will let the server compute RTT statistics).
* `ts` [mandatory] field, the UNIX timestamp (at which the server sent the message).
* `treeLeaves` [optional] field. If this field is present, it must be a power of two. If treeLeaves>1, the client has to reply a `pokeBack` message with a treeLeaves field equals to treeLeaves/2. The server will then randomly picks up two clients and send them a `poke` message with the new treeLeaves and so on and so forth. This mecanism simulate a messaging tree to heavily load the system.
* `treeId` [optional] field. This field identifies the tree. The client has to send back this id along with the treeLeaves so that the server knows which tree is concerned.
* `parentId` [optional] field. This field identifies the parent (socket.io id) of the poke (in case of a tree poke) or -1 if the node is the root of the tree.

### Client to server `pokeBack` message

This message must be systematically sent by the client each times it receives a `poke` message. Here are the fields that this message can contain:
* `id` [mandatory] field, this is the exact copy of the id sentby the server.
* `delay` [mandatory] field. This field is the difference between the ts sent by the server and the local (client) UNIX timestamp. We assume that server and clients are syncronized with the same NTP server.
* `serverTs` [mandatory] field. This field is the ts given by the server (it should be sent back by the client so that the server can compute message RTT more accurately)
* `treeLeaves` [optional] field. In reply to a `poke` message the client sends back treeLeaves/2 (if received treeLeaves is greater or equal to 2). The server will forward this treeLeaves to two clients.
* `treeId` [optional] field. This field identifies the tree. This is the exact copy of the treeId sent by the server.


Monitoring Protocol
-------------------

In addition to messaging protocol, the server implements a monitoring protocol. This protocol involves the server and a single client (hereafter called the monitor). Using this protocol, the monitor can initiate and monitor server <-> clients messages and get information about server statistics. Websockets (socket.io) is also used as the underlying protocol.


### Monitor to server `initiatePoke` message

This message is sent by the monitor to initiate a poke message to client. Depending on the parameters of the message, the server can initiate different types of messages:
* unicast messages: a single `poke` message will be sent to a single client. The receiver can be specified by the monitor, otherwise a client will be chosen randomly. The monitor will be informed by the server of the response of the client.
* broadcast messages: the server will iterate over all client and send them a single `poke` message. The monitor will be informed of the response of all clients when the last client will have answered. The monitor can also request the server in the meantime to get information about the number of reponses the server already received.
* tree messages: the server will send a `poke` message to a single client (randomly chosen) with a treeLeaves (the monitor must provide this value). This will cause the creation of a tree of messages. The monitor will then be informed of the reception by the server of the last message of treeLeaves 1 (the last message of the leaves of the tree). The monitor can also request the server in the meantime to get the status of the tree (the number of messages already received)

Here are the different fields the monitor can add to any `initiatePoke` message:
* `type` [mandatory] field, this is the type of action he wants the server to execute. It can be `unicast`, `broadcast` or `tree`.
* `destId` [mandatory, only if type = unicast] field. This is the id (socket.io id) of the client the server should reach.
* `treeLeaves` [mandatory, only if type = tree] field. This parameter will be transmitted as is to the first client of the tree. It must be a power of 2 and it represents the number of leaves of the tree (2*treeLeaves-1 is the the total number of clients involved in the tree).


### Monitor to server `statusRequest` message

This message can be sent by the monitor to get the status of the server. There is no parameter.

### Server to monitor `status` message

This message will be sent by the server to the monitor in response to `statusRequest` message or automatically when an important event has occured (unicast poke answer received, last broadcast poke answser received, last tree poke answer received). Here is the detail of a `status` message structure (all fields are optional and depends on server's context):
* `unicast` field. This field is present when the server wants to notify the monitor of the end of an unicast poke. It contains three fields:
	* `clientId` the id (socket.io id) of the poked client
	* `pingDelay` is the time taken by the poke to reach the client (not very accurate since this time is computed assuming that server and client are synchronized)
	* `rtt` is the round trip time of the poke (more accurate since computed by the server)
* `broadcast` field. This field is present to notify the monitor of the status of a broadcast poke. This field is an array of couples (pingDelay, rtt). broadcast[i] indicated the ping delay and the RTT of the ith client, length(broadcast) indicates the number of answers already received.
* `tree` field. This field is an array representing all the client involved in the tree. tree[i] is a dictionary containing the parent, the rtt and the pingDelay of the ith client involved in the tree.


