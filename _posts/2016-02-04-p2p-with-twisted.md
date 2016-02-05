---
layout: post
title:  "A simple p2p network in Python twisted"
date:   2016-02-04 15:38:56 +0100
categories: dev
---

Let's do a simple p2p network in Python. It should be able to discover
other nodes and ping them over the network. We will be using
[twisted][twisted] for building the network. 

I mostly used the [Bitcoin Developer Documentation][bitcoindoc] to
teach me how to write a p2p network. I wrote an implementation of the model described here in {% include icon-github.html username="benediktkr" %} /
[ncpoc](https://github.com/benediktkr/ncpoc). 


Since this is a peer-to-peer network, every instance has to be both
the a server and a client. The first clients need some way to find
each other, and recognize itself in the event it tried to connect to
it self.

### UUID to identify peers

Since outgoing TCP connections are assigned random ports, we cannot
rely on `ip:port` as identifies for instances of the p2p network. We
are going assign each instance a random UUID for the session. A UUID
is 128 bits, seems like a reasonably small overhead.

{% highlight python %}
>>> from uuid import uuid4
>>> generate_nodeid = lambda: str(uuid4())
'a46de8d6-177e-4644-a711-63d182fdbade'
{% endhighlight %}

### Defining a simple protocol in Twisted

We will only talk about Twisted to the extent that is nessacary to
build this. For more information on how to build servers and protocols
with twisted,
[Writing Servers in the Twisted documenation][twistedserver] is a good
place to start.

The `Factory` class instance is persistent between connections, so
it's where we store things like the peer list and the UUID for this
session. A new instance of `MyFactory`is created for each connection.

{% highlight python %}
from twisted.internet.endpoints import TCP4ServerEndpoint
from twisted.internet.protocol import Protocol, Factory
from twisted.internet import reactor

class MyProtocol(Protocol):
    def __init__(self, factory):
        self.factory = factory
        self.nodeid = self.factory.nodeid

class MyFactory(Factory):
    def startFactory(self):
        self.peers = {}
        self.nodeid = generate_nodeid()

    def buildProtocol(self, addr):
        return NCProtocol(self)
        
endpoint = TCP4ServerEndpoint(reactor, 5999)
endpoint.listen(MyFactory())
{% endhighlight %}

This will define a listener for a completely empty protocol on
`localhost:5999`. Not very useful so far. We want the nodes to be able
to send each other messages and taalk.We will use JSON strings to
create the messages, it is easy to serialize and very readable.

Let's begin by creating a `hello` message for new nodes to introduce
themselves to each other. We also include what kind of a message it
is, so peers can easily figure out how to handle it.

{% highlight python %}
{
    'nodeid' : 'a46de8d6-177e-4644-a711-63d182fdbade',
    'msgtype': 'hello',
}
{% endhighlight %}

Now we need to edit our code to be handle getting a hello from a new
conncetion. We also need to keep a list of connected UUIDs (the
`peers` property of the `Factory` class.

{% highlight python %}
import json

class MyProtocol(Protocol):
    def __init__(self, factory):
        self.factory = factory
        self.state = "HELLO"
        self.remote_nodeid = None
        self.nodeid = self.factory.nodeid

    def connectionMade(self):
        print "Connection from", self.transport.getPeer()

        def connectionLost(self, reason):
        if self.remote_nodeid in self.factory.peers:
            self.factory.peers.pop(self.remote_nodeid)
        print self.nodeid, "disconnected"

    def dataReceived(self, data):
        for line in data.splitlines():
            line = line.strip()
            if self.state == "HELLO":
                self.handle_hello(line)
                self.state = "READY"

    def send_hello(self):
        hello = json.puts({'nodeid': self.nodeid, 'msgtype': 'hello'})
        self.transport.write(hello + "\n")
   
    def handle_hello(self, hello):
        hello = json.loads(hello)
        self.remote_nodeid = hello["nodeid"]
        if self.remote_nodeid == self.nodeid:
            print "Connected to myself."
            self.transport.loseConnection()
        else:
            self.factory.peers[self.remote_nodeid] = self
{% endhighlight %}

This will be capable of sending and reading a `hello` message, and
keep track of connected peers.

### Talking to the protocol

So now we have something that looks like a "server" thats able to
handle a `hello` message. We need to make this also act as a
"client". Twisted is a nice-batteries included library, so this is
almost for free with Twisted's callbacks.

{% highlight python %}
from twisted.internet.endpoints import TCP4ClientEndpoint, connectProtocol

def gotProtocol(p):
    """The callback to start the protocol exchange. We let connecting
    nodes start the hello handshake""" 
    p.send_hello()

point = TCP4ClientEndpoint(reactor, "localhost", 5999)
d = connectProtocol(point, MyProtocol())
d.addCallback(gotProtocol)
{% endhighlight %}

### Bootstrapping the network

If we run both of those parts, a succesful handshake will hopefully be
performed and the listening factory will output something like

    Connection from 127.0.0.1:58790
    Connected to myself. 
    a46de8d6-177e-4644-a711-63d182fdbade disconnected.

We need more than one instance to have a p2p network. The simplest way
to get the first clients in the network to find each other is to
simply have a list with `host:port` combinations and then initially
loop through this list and try connecting to them, starting each
instance with a different listening port.

{% highlight python %}
BOOTSTRAP_LIST = [ "localhost:5999"
                 , "localhost:5998"
                 , "localhost:5997" ]

for bootstrap in BOOTSTRAP_LIST:
    host, port = bootstrap.split(":")
    point = TCP4ClientEndpoint(reactor, host, int(port))
    d = connectProtocol(point, MyProtocol())
    d.addCallback(gotProtocol)
reactor.run()
{% endhighlight %}
-
And now we can bootstrap a network with a couple of different
instances of our program.

### Ping

Let's make them do something useful and add a `ping` message, with an
empty payload. We also need a `pong` message for the response. 

{% highlight python %}
{
     'msgtype': 'ping',
}
{% endhighlight %}
{% highlight python %}
{
    'msgtype': 'pong',
}
{% endhighlight %}

When we send a `ping` to a connected node, we note when we got the
`pong` reply. This is useful to detect dead clients, and gives our
simple `ping`/`pong` message flow some purpose. We also use Twisted's
`LoopingCall` to ping nodes on a regular interval. Since each
connection has it's own `MyProtocol` instance, we don't have to loop
over all of them, we can think in the abstraction of connections.

Again, twisted does most of the heavy-lifting for us. Edit the
`MyProtocol` class to operate the mechanics. 

{% highlight python %}
from time import time

from twisted.internet.task import LoopingCall

class MyProtocol(Protocol):
    def __init__(self, factory):
        self.factory = factory
        self.state = "HELLO"
        self.remote_nodeid = None
        self.nodeid = self.factory.nodeid
        self.lc_ping = LoopingCall(self.send_ping)
        self.lastping = None

    def connectionMade(self):
        print "Connection from", self.transport.getPeer()

    def connectionLost(self, reason):
        if self.remote_nodeid in self.factory.peers:
            self.factory.peers.pop(self.remote_nodeid)
            self.lc_ping.stop()
        print self.nodeid, "disconnected"

    def dataReceived(self, data):
        for line in data.splitlines():
            line = line.strip()
            msgtype = json.loads(line)['msgtype']
            if self.state == "HELLO" or msgtype == "hello":
                self.handle_hello(line)
                self.state = "READY"
            elif msgtype == "ping":
                self.handle_ping()
            elif msgtype == "pong":
                self.handle_pong()

    def send_hello(self):
        hello = json.puts({'nodeid': self.nodeid, 'msgtype': 'hello'})
        self.transport.write(hello + "\n")

    def send_ping(self):
        ping = json.puts({'msgtype': 'ping'})
        print "Pinging", self.remote_nodeid
        self.transport.write(ping + "\n")

    def send_pong(self):
        ping = json.puts({'msgtype': 'pong'})
        self.transport.write(pong + "\n")

    def handle_ping(self, ping):
        self.send_pong()
   
   def handle_pong(self, pong):
        print "Got pong from", self.remote_nodeid
        ###Update the timestamp
        self.lastping = time()
        
    def handle_hello(self, hello):
        hello = json.loads(hello)
        self.remote_nodeid = hello["nodeid"]
        if self.remote_nodeid == self.nodeid:
            print "Connected to myself."
            self.transport.loseConnection()
        else:
            self.factory.peers[self.remote_nodeid] = self
            self.lc_ping.start(60)
{% endhighlight %}

Now we have a bootstrapped peer-to-peer network where the peers ping
each other every minute. But it can only find other nodes by bootstrapping.

### Discovering nodes

The current version just has the ability to connect to a list of
predefined nodes. We need to add som extra functionality to turn it
into a true p2p network.

 * We need to distinguish between a peer that we are connected to and
   a peer we have a connection from. Outgoing TCP connections are
   assigned a random port number that we can't connect to. To avoid
   the inevitable confusing that names in strings will bring, let's
   use the integer `1` for the first type of peer and `2` for the
   second type described.
 * We need to advertise our own listening `ip:port` pair. (In the
   practical reality, NAT will get in the way a lot of the time, but we
   will ignore it here).
 * Define an `addr` messages to advertise peers and `getaddr` to
   request new peers.

After these changes, our `Protocol` instance looks like this

{% highlight python %}
class MyProtocol(Protocol):
    def __init__(self, factory, peertype):
        self.factory = factory
        self.state = "HELLO"
        self.remote_nodeid = None
        self.nodeid = self.factory.nodeid
        self.lc_ping = LoopingCall(self.send_ping)
        self.peertype = peertype
        self.lastping = None

    def connectionMade(self):
        remote_ip = self.transport.getPeer()
        host_ip = self.transport.getHost()
        self.remote_ip = remote_ip.host + ":" + str(remote_ip.port)
        self.host_ip = host_ip.host + ":" + str(host_ip.port)
        print "Connection from", self.transport.getPeer()

    def connectionLost(self, reason):
        if self.remote_nodeid in self.factory.peers:
            self.factory.peers.pop(self.remote_nodeid)
            self.lc_ping.stop()
        print self.nodeid, "disconnected"

    def dataReceived(self, data):
        for line in data.splitlines():
            line = line.strip()
            msgtype = json.loads(line)['msgtype']
            if self.state == "HELLO" or msgtype == "hello":
                self.handle_hello(line)
                self.state = "READY"
            elif msgtype == "ping":
                self.handle_ping()
            elif msgtype == "pong":
                self.handle_pong()
            elif msg_type == "getaddr":
                self.handle_getaddr()
            

    ###The methods for ping and pong remain unchanged and are omitted
    ###for brevity

    def send_addr(self, mine=False):
        now = time()
        if mine:
            peers = [self.host_ip]
        else:
            peers = [(peer.remote_ip, peer.remote_nodeid)
                     for peer in self.factory.peers
                     if peer.peertype == 1 and peer.lastping > now-240]
        addr = json.puts({'msgtype': 'addr', 'peers': peers})
        self.transport.write(peers + "\n")

    def send_addr(self, mine=False):
        now = time()
        if mine:
            peers = [self.host_ip]
        else:
            peers = [(peer.remote_ip, peer.remote_nodeid)
                     for peer in self.factory.peers
                     if peer.peertype == 1 and peer.lastping > now-240]
        addr = json.puts({'msgtype': 'addr', 'peers': peers})
        self.transport.write(peers + "\n")

    def handle_addr(self, addr):
        json = json.loads(addr)
        for remote_ip, remote_nodeid in json["peers"]:
            if remote_node not in self.factory.peers:
                host, port = remote_ip.split(":")
                point = TCP4ClientEndpoint(reactor, host, int(port))
                d = connectProtocol(point, MyProtocol(2))
                d.addCallback(gotProtocol)
        
    def handle_getaddr(self, getaddr):
        self.send_addr()
        
    def handle_hello(self, hello):
        hello = json.loads(hello)
        self.remote_nodeid = hello["nodeid"]
        if self.remote_nodeid == self.nodeid:
            print "Connected to myself."
            self.transport.loseConnection()
        else:
            self.factory.peers[self.remote_nodeid] = self
            self.lc_ping.start(60)
            ###inform our new peer about us
            self.send_addr(mine=True)
            ###and ask them for more peers
            self.send_getaddr()
{% endhighlight %}

### Summary

And now we have a simple p2p network capable of the following

 * Boostrap by way of a preconfigured list
 * Connect to other nodes and make a handshake.
 * Ask peers for nodes and connect to them
 * Advertise it's own peers as well as it self to other peers
 * Ping peers

[twisted]: https://twistedmatrix.com/
[bitcoindoc]: https://bitcoin.org/en/developer-documentation
[twistedserver]: http://twistedmatrix.com/documents/current/core/howto/servers.html
