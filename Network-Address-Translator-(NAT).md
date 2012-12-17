## Introduction 
In this lab assignment you will be writing a simple NAT that can handle ICMP and TCP. It will implement a subset of the functionality specified by RFC5382 and RFC5508. Expect to refer often to these RFCs.

Before beginning this lab, it is crucial that you:
* Understand how NATs work. Consider re-watching the NAT lectures.
* Understand TCP handshake and teardown packet sequences. Consider working through the TCP state diagram.
* Understand NAT Endpoint Independence.
NAT builds on "Simple Router". You should start with your static router code and extend it to include NAT functionality.

As with "Simple Router", we will create a NAT that sits in Mininet between the app servers and the client.  The internal interface of the NAT faces the client, while the external interfaces are connected to app servers. The app servers are "outside" the NAT, while the client is "inside."  

The topology of NAT is as follows, where the NAT's internal interface (eth1) faces the client and its external interface (eth2) has two application servers connected with a switch:

![alt text](http://yuba.stanford.edu/~huangty/mininet/nat_topo.png "Topology for NAT")


A correct implementation should support the following operations from the emulated client host:

* Pinging the NAT's internal interface from the emulated client host
* Pinging any of the app servers (e.g. 172.64.3.21, 172.64.3.22 above)
* Downloading files using HTTP from the app servers
All packets to external hosts (app servers) should appear to come from eth2's address (e.g. 172.64.3.1 above).


##General NAT Logic
There are three major parts to the assignment:

* translating ICMP echo messages (and their corresponding replies),
* translating TCP packets
* cleaning up defunct mappings between internal addresses and the external address.
Note that your NAT is not required to handle UDP. It is entirely up to you whether you drop or forward UDP traffic.

##Simple Router
Your NAT builds on the "simple router". You must add a new command-line flag, -n, which controls whether the NAT is enabled. If the -n flag is not passed, then the router should act following the requirements of "simple router". For example, it should be possible to traceroute across the router when the -n flag is not passed. All of the ICMP errors in "simple router" still apply. For example, trying to open a TCP port on the router should cause an ICMP port unreachable reply (with the caveat of TCP requirement 4 below). More precisely:

* Your NAT MUST generate and process ICMP messages as per the "simple router".

## ICMP Echo Request/Reply
The first four bytes of an ICMP echo request contain a 16-bit query identifier and a 16-bit sequence number. Because multiple hosts behind the NAT may choose the same identifier and sequence number, the NAT must make their combination globally unique. It needs to maintain the mapping between a globally unique identifier and the corresponding internal address and internal identifier, so that it can rewrite the corresponding ICMP echo reply messages. The first three requirements for your NAT are:

* Your NAT MUST translate ICMP echo requests from internal addresses to external addresses, and MUST correctly translate the corresponding ICMP echo replies.
* ICMP echo requests MUST be external host independent: two requests from the same internal host with the same query identifier to different external hosts MUST have the same external identifier.
* An ICMP query mapping MUST NOT expire less than 60 seconds after its last use. This value MUST be configurable, as described below.

Other "simple router" ICMP behavior should continue to work properly (e.g. responding to an ECHO request from an external host addressed to the NAT's external interface).

## TCP Connections
When an internal host opens a TCP connection to an external host, your NAT must rewrite the packet so that it appears as if it is coming from the NAT's external address. This requires allocating a globally unique port, under a set of restrictions as detailed below. The requirements for your NAT are a subset of those in specified in [RFC5382](https://tools.ietf.org/html/rfc5382); in some cases they are more restrictive. Refer to the RFC for details on the terms used. Your NAT has the following requirements:

* Your NAT MUST have an "Endpoint-Independent Mapping" behavior for TCP.
* Your NAT MUST support all valid sequences of TCP packets (defined in [RFC0793](http://tools.ietf.org/html/rfc793)) for connections initiated both internally as well as externally when the connection is permitted by the NAT. In particular, in addition to handling the TCP 3-way handshake mode of connection initiation, A NAT MUST handle the TCP [simultaneous-open mode of connection initiation](http://ttcplinux.sourceforge.net/documents/one/tcpstate/tcpstate.html).
* Your NAT MUST have an "Endpoint-Independent Filtering" behavior for TCP.
* Your NAT MUST NOT respond to an unsolicited inbound SYN packet for at least 6 seconds after the packet is received. If during this interval the NAT receives and translates an outbound SYN for the connection the NAT MUST silently drop the original unsolicited inbound SYN packet. Otherwise, the NAT MUST send an ICMP Port Unreachable error (Type 3, Code 3) for the original SYN.
If your NAT cannot determine whether the endpoints of a TCP connection are active, it MUST abandon the session if it has been idle for some time. In such cases, the value of the "established connection idle-timeout" MUST NOT be less than 2 hours 4 minutes. The value of the "transitory connection idle-timeout" MUST NOT be less than 4 minutes. This value MUST be configurable, as described below. EDIT: Note you *MUST* timeout idle connections.
* Your NAT MUST NOT have a "Port assignment" behavior of "Port overloading" for TCP.

NOTE: hairpinning for TCP is NOT required. It is up to you whether you support it, or other behavior not required here.


