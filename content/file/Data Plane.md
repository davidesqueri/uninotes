---
number headings: auto, first-level 1, max 3, 1.1
---
#uni
- [1 Introduction](#1%20Introduction)
- [2 Network Layer: overview (4.1)](#2%20Network%20Layer:%20overview%20(4.1))
	- [2.1 Network service Model (4.1.2)](#2.1%20Network%20service%20Model%20(4.1.2))
- [3 Routers (4.2)](#3%20Routers%20(4.2))
	- [3.1 Forwarding](#3.1%20Forwarding)
	- [3.2 Switching Fabrics](#3.2%20Switching%20Fabrics)
	- [3.3 Queueing](#3.3%20Queueing)
	- [3.4 Packet Scheduling](#3.4%20Packet%20Scheduling)
- [4 IP: the internet protocol (4.3)](#4%20IP:%20the%20internet%20protocol%20(4.3))
	- [4.1 IPv4 Datagram format (4.3.1)](#4.1%20IPv4%20Datagram%20format%20(4.3.1))
	- [4.2 Fragmentation (4.3.2)](#4.2%20Fragmentation%20(4.3.2))
	- [4.3 IPv4 Addressing (4.3.3)](#4.3%20IPv4%20Addressing%20(4.3.3))
		- [4.3.1 Obtaining an Address](#4.3.1%20Obtaining%20an%20Address)
	- [4.4 Network Address Translation (4.3.4)](#4.4%20Network%20Address%20Translation%20(4.3.4))
	- [4.5 IPv6 (4.3.5)](#4.5%20IPv6%20(4.3.5))
		- [4.5.1 IPv6 datagram format](#4.5.1%20IPv6%20datagram%20format)
	- [4.6 Transition from IPv4 to IPv6](#4.6%20Transition%20from%20IPv4%20to%20IPv6)
	- [4.7 ICMP protocol](#4.7%20ICMP%20protocol)
- [5 Generalized Forwarding and SDN (4.4)](#5%20Generalized%20Forwarding%20and%20SDN%20(4.4))
	- [5.1 OpenFlow](#5.1%20OpenFlow)
		- [5.1.2 OpenFlow abstraction](#5.1.2%20OpenFlow%20abstraction)
- [6 Middlebox](#6%20Middlebox)
- [7 Where's the intelligence](#7%20Where's%20the%20intelligence)
# 1 Introduction
The Data Plane Functions of the **network layer** are the *per-router* functions in the network layer that determine how a datagram arriving on one of a router's input links is forwarded to one of that router's output links.

Traditionally, these control-plane routing protocols and data-plane forwarding functions have been implemented together, monolithically, within a router. **Software-defined networking** (**SDN**) explicitly separates the data plane and control plane by implementing these control plane functions as a separate service, typically in a **remote controller**.
# 2 Network Layer: overview (4.1)
This layer receives segments from the transport layer and transports it from the sending to the receiving host: the sender encapsulates segments into datagrams and passes them to the link layer, the receiver delivers these segments to the transport layer protocol.
Two important network-layer functions can be identified: **forwarding** (**data plane** — moving packets from a router's input link to the appropriate router output link) and **routing**  (**control plane** — determining the route taken by packets in their path).

**Switches** are **link-layer devices**: they base their forwarding decision on values in the fields of the link-layer frame and the only function they offer is transferring a packet from an input link to an output link — so called **packet switchers**.
**Routers** also implement the network layer, they are therefore called **network-layer devices**. A router examines the header fields in all IP datagrams that pass through it and according to these fields it moves these datagrams from input ports to the correct output ports. A key element in every router is its **forwarding table**, which determines which packet gets forwarded to which link.
## 2.1 Network service Model (4.1.2)
Services can be offered for *individual* datagrams, or for a *flow* of datagrams.
Here are some examples of services that the network layer *could* offer: guaranteed delivery, guaranteed delivery with a bounded delay, in-order packet delivery, guaranteed minimal bandwidth and security.

The **Internet Protocol** (**IP**) offers a **best effort service**: it gives no guarantees and does not offer security, in-order delivery or any other service.

Interestingly, in spite of these well-developed alternatives, the Internet’s basic best-effort service model combined with adequate bandwidth provisioning have arguably proven to be more than “good enough” to enable an amazing range of applications.
# 3 Routers (4.2)
Four router components can be identified:
- **routing processor**: responsible for the control plane (software): in can either execute routing protocols himself of communicate with the remote SDN controller. The processor also performs network management functions.
- **switching fabric**: responsible for the data plane (hardware): it connects input ports to output ports.
- **input** and **output** ports: these perform the necessary link-layer and physical-layer functions to transmit and receive packets to and from the physical links.

> router input port:![[routerinputport.svg]]

## 3.1 Forwarding
The forwarding can either be **destination-based** (the forward is based only on the destination IP address) or **generalized** (the forward is based on any set of header field values).

In destination-based forwarding, when looking for the entry of the forwarding table for a given destination address, we use the *longest* address prefix that matches the destination address.
This is often performed using **ternary content addressable memorie**s (**TCAM**s) — present the address to TCAM and in one clock cycle retrive the address, regardless of table size.
## 3.2 Switching Fabrics
The switching Fabric transfers a packet from an input link to the appropriate output link. The **switching rate** is the rate at which packets can be transferred from input links to output links, it is often measured as a multiple of the input/output line rate (rate R).

There are 3 types of switching fabrics: 
- **Switching via memory**: the switching is under direct control of the CPU. The packet is first copied to the system's memory. The speed is limited by the memory bandwidth.
- **Switching via bus**: **Bus contention**: packets are transferred from input ports to output ports over a shared bus: all ports receive the packet but only the right one will keep it. The switching speed is limited by the bus bandwidth — a 32 Gbps bus is sufficient for access routers.
- **Switching via interconnection network**: one way to pass the bandwidth limitation of a single bus is to utilize a crossbar switch consisting of $2N$ buses that connect $N$ input ports to $N$ output ports. Using this technology only the right port receives the packet and multiple packets can be forwarded in parallel, if the destination ports differ.

It is also possible to run multiple switching fabrics in parallel and break packets in smaller chunks, each traveling on a different switching fabric before getting reassembled into the original datagram at the end of the packet forward.
## 3.3 Queueing
Packet queues may form at both the input ports and the output ports and the extent of it will depend on **traffic load**, the **relative speed of the switching fabric** and the **line speed**.
Queueing can cause delay and packet loss due to buffer overflow.

If the switching fabric is slower than the input ports' combined throughput, queueing at the input ports may occur.
Another reason for input port queueing could be **Head of line (HOL) blocking**: the queued datagram at the front of the queue—maybe waiting for other packets destined to the same output port—prevents others in the queue from moving forward. This type of queue can unfortunately grow to unbounded lengths as soon as the packet arrival rate on the input links reaches only 58% of their capacity.

**Buffering**—a type of output queueing—instead happens when datagrams arrive from the fabric faster thank the output link's transmission rate — if such a situation continues, the buffer will not have enough memory to store new incoming packets and a decision must be made to either drop the arriving packet (**drop-tail**policy) or remove one or more already-queued packets to make room for the new one. It is some times better to preemptively drop—or mark—a packet when nearing the limit, to signal congestion: policies employing this strategies are known as **active queue management** (**AQM**) algorithms.

The **scheduling discipline** chooses among the queued datagrams for transmission -> **priority scheduling**.
## 3.4 Packet Scheduling
Packet scheduling is the problem of determining the order in which queued packets are transmitted over an outgoing link.
There are many possible policies for packet scheduling, here are some examples:
- **FCFS**/**FIFO**: Packets are sent in the same order they were buffered.
- **Priority**: The arriving traffic is classified and queued by class. Inside a priority queue, FCFS is used. It is always sent a packet from the currently not empty highest priority queue.
- **Round Robin**: The arriving traffic is classified and queued by class and the sending is alternated between classes. A so-called **work-conserving queueing** discipline will never allow the link to remain idle whenever there are packets of any class queued for transmission.
- **Weighted fair queueing** (**WFQ**): This is a generalized Round Robin, where the queues are not handled the same: every queue has a weight, that determines it's available percentage of traffic (See [[Network Neutrality]]).
# 4 IP: the internet protocol (4.3)
The **Internet Protocol** (**IP**) is the network layer communications protocol in the Internet protocol suite for relaying datagrams across network boundaries. Its routing function enables **internetworking**, and essentially establishes the Internet.
*IP has the task of delivering packets from the source host to the destination host solely based on the IP addresses in the packet headers*. For this purpose, IP defines packet structures that encapsulate the data to be delivered. It also defines addressing methods that are used to label the datagram with source and destination information.
There are two versions currently in use: IP protocol version 4 and 6 — IPv4 and IPv6 for short.
The IP protocol uses as support ICMP ([[#3.7 ICMP protocol]]): this handles error reporting and router signaling.
## 4.1 IPv4 Datagram format (4.3.1)
![[Pasted image 20251028152916.png]]

- **header length** in bytes: this is need because the **options** field can be of variable length although not often used, leading to a typical IP datagram header length of 20 bytes.
- **type of service** (**ToS**) bits: these allow different types of IP datagrams to be distinguished from each other (without looking at the payload). It can for example be useful to distinguish real-time from non-real-time datagrams. Bits 0 to 5 represent diffserv (priority). Bits 6 and 7 are used for **ECN** (not really used) (see [[Transport Layer#3.8.3 ECN explicit congestion notification echo (3.7.2)]]).
- **identifier**, **flags** (3 bits) and **fragment offset** (13 bits) all have to do with IPv4 fragmentation. Fragmentation does not exist in IPv6. Fragment Offset is the offset of the first segment in the original datagram. `fragflag=1` means the current datagram is NOT the last fragment
- **TTL**: time to live, maximum number of forwards in routers before being dropped: decremented at each router.
- **upper layer protocol**: a value of 6 indicates TCP, 17 indicates instead UDP.
- **header checksum**: computer as Internet Checksum (treating each 2 bytes in the header as a number and summing them using 1s complement arithmetic). Routers perform integrity checks on every incoming packet, when an error is detected the packet is dropped. Note that the checksum for a packet must be computed and updated at each hop, because IP header values can change in transit — TTL for example.
- **options**: this field allows IPv4 headers to be extended. Although meant to be used rarely, its sole existence causes problems: because of its variable length the exact start of the payload in a datagram varies, also since the use of options in datagrams varies, the processing time of an IP datagram at routers can vary greatly. Because of these issues the options field is not present in IPv6 headers.
- **data**: this field usually contains the transport layer segment and is obviously of variable length, though it can carry other types of data, such as ICMP messages.
## 4.2 Fragmentation (4.3.2)
As we said, different network links have a different **MTU** (maximum transfer unit): the largest possible size for a link-level frame, typically 1500 bytes (for ethernet for example).
When a router receives a datagram too big for the desired output link, it fragments it in smaller datagrams, which then get reassembled only at the final destination (this keeps complexity at the network's edge). If the next link has a smaller MTU, the new router has to again fragment the fragments.

If not every fragment arrives in a determined timeframe, every fragment is dropped and the original datagram is therefore lost.

This means that when a destination host receives a series of datagrams from the same source, it needs to determine whether any of these datagrams are fragments of a larger datagram and if so it needs to determine how these fragments must be put together again.
This is the reason the IPv4 header contains the fragmentation fields: **identification**, **flags** and **fragmentation offset**. 
When a host creates a datagram, it inserts in it an identification number (typically incrementing it with each new datagram), together with source and destination IP addresses. When a router fragments a datagram, the new resulting datagrams all include this ID and addresses and the last fragment has a flag bit set to 0 (all the others have it set to 1). Furthermore every fragment of the datagram includes a fragmentation offset, that tells the host where the fragment fits within the original IP datagram.
## 4.3 IPv4 Addressing (4.3.3)
Every interface—the boundary between the host and the physical link—of a host or router is associated with an **IP address**, which is a 32-bit (4 bytes) number that uniquely (not actually, see [[#3.4 Network Address Translation]]) identifies it on the internet.
IP addresses are usually written in so-called **dotted-decimal notation**, in which each byte of the address is written in its decimal form and is separated by a dot from other bytes in the address.

A portion of an interface's IP address is determined by the **subnet** to which it is connected. In particular, IP addresses have **subnet masks** which indicate how many of the first bits in the IP address are determined by the subnet: the `x` most significant bits in a `a.b.c.d/x` address constitute the network portion of the IP address and are often referred to as the **prefix** of the address.
The routers that are outside the destination subnet of a datagram only consider the prefix of the address, the remaining `32-x` bits compose the identifier of the interface in its subnet and are used by routers inside it.
This address assignment strategy is known as **Classless Interdomain Routing** (**CIDR**), before it there was an addressing scheme known as **classful addressing** which constrained the network portions of an IP address to be 8,16 or 24 bits, respectively **class** A,B and C. Classful addressing led to a quick depletion of IPv4 addresses because small sized organizations still had to use class B network portions, since class C ones could only support up to $2^8-2=254$ hosts (interfaces), obviously too little.

IPv4 offers a special type of address: the **broadcast address** `255.255.255.255`, datagrams with this destination address are delivered to every host on the same subnets — some routers also forward broadcast messages into neighboring subnets.
### 4.3.1 Obtaining an Address
The **ICANN** (Internet Corporation for Assigned Names and Numbers) allocates IP addresses, through 5 **regional registries** (RRs), who may then allocate to local registries.
These local registries assign blocks of IP address to ISPs, which in turn allocate a portion of their block to organizations.

Organizations can at this point finally assign IP addresses from their subnet to hosts, this can be done manually by network administrator but is usually done automatically through [[DHCP]].

ICANN allocated the last chunk of IPv4 addresses to RRs in 2011...
NAT helps IPv4 address space exhaustion.
IPv6 has a 128 bit (instead of 32) address space.
#### DHCP
The **Dynamic Host Configuration Protocol** (**DHCP**) is a client-server protocol: when a host joins a network it can ask a DHCP server for a temporary IP address, this way IP address are dynamically and automatically assigned.
This assigned address is only held while connected to the network (IP addresses have a due date, so the hosts need to renew their lease), this leads to the possibility of reuse of IP addresses, so it is better for mobile users who join and leave often a network.
(See [[DHCP]] for more details on the interaction).
## 4.4 Network Address Translation (4.3.4)
With **Network Address Translation** (**NAT**) all devices in a local network share just **one** IPv4 address from the outside world point of view.
This means every datagram that leaves a local network that uses NAT has the same source NAT IP address, but different source port numbers.

Every device in a local network has a 32-bit address in a "private" IP address space, that can only be used in the local network, since these private IP address spaces only work in the local network and don't comunicate directly with the outer internet, only through NAT.
These private IP addresses spaces are: `10.0.0.0/8`, `172.16.0.0/12` and `192.168.0.0/16`.

NAT offers several Advantages: all devices in a local network can use a single IP address, the ISP can change without changing addresses of devices in the local network and **security**: devices inside a local network are not directly accessible by the outside world, only through the NAT router.

The NAT router must now keep a **NAT translation table** and do the following:
1. replace every source IP address and port number of every outgoing datagram with the NAT IP address and a new port number.
2. remember every `(source IP address, port number)` to `(NAT IP address, new port number)` translation pair.
3. for the incoming datagrams perform the inverse translation.

NAT is widely deployed but still has some issues (or critiques):
- port numbers should be used to identify processes, not hosts
- routers should only process up to layer 3, instead it changes layer 4 information such as address and port
- address shortage should be solved by IPv6, NAT is only a bandaid
- it violates end-to-end argument, information should be exchanged end-to-end, nobody in the middle should change this information, instead the router does
- NAT traversal: what if a client wants to connect to server behind NAT? We need to use **Port Forwarding**: we have to insert manually lines in the NAT table so that incoming messages directed to ports used by the server are directed to the server.
## 4.5 IPv6 (4.3.5)
**IP protocol version 6** (**IPv6**) was introduced because the 32 bit address space was not enough (IPv6 uses 128 bits, allowing for many more addresses) and it was rapidly being all used up.
The designers took this opportunity to speed up processing/forwarding: instead of a variable length header, IPv6 uses a 40 byte fixed length header. It also enables different network-layer treatments of "flows".
(Why not IPv5? It was already taken by "string protocol", but it failed).

The most important changes introduced with version 6 are:
- 128-bit addresses and a **anycast address**: with IPv6 every grain of sand on planet Earth can be addressed. An anycast address allows a datagram to be delivered to a group of hosts.
- *fixed* 40 byte header.
- introduction of a 20-bit **flow label** which allows for labeling of packets belonging to particular flows for which the sender requests special handling, such as non-default quality of service or real-time service.
### 4.5.1 IPv6 datagram format

> IPv6 datagram format:![[Pasted image 20251031120717.png]]

These are the fields of this new fixed-length header:
- 20-bit **flow label**: this identifies to which "flow" a datagram belongs.
- 8-bit **traffic class** (or priority): this indicated the priority of this datagram inside its flow.
- hop limit: this is a different name for the TTL field but works exactly the same.
- **next header**: in IPv4 we had a field that indicated the upper layer protocol, this is the same thing: inside the payload there is the header for the transport layer protocol, the "next header" field indicates what this header inside the payload is. 

This means that to speed up processing the following was removed: (1) the checksum (because it had to be computed at every hop and is redundant), (2) the options field (because of its variable length, in IPv6 this field is inside the payload, pointed to from the `next header` field) and (3) fragmentation: if a datagram is to big for the next link, it gets discarded, leading to the need for a new type of message that can notify the sender of the MTU problem (when the sender receives this message tries again with smaller datagrams). For IoT instead we introduce a new kind of fragmentation, because these links typically have really low MTUs.
## 4.6 Transition from IPv4 to IPv6
Let's now consider a practical manner: how will the internet be transitioned from version 4 to version 6: new IPv6 capable systems can be made backward-compatible but already deployed IPv4 systems cannot handle IPv6 datagrams.
A **flag day** (a given time and date when all internet machines would be turned off and upgraded) is unthinkable because we now have billions of devices, so we must turn to **tunneling**: the IPv6 datagram gets carried as a payload in a IPv4 datagram among IPv4 routers ("packet within a packet"), these IPv4 routers between two IPv6 routers are referred to as a **tunnel**.
An IPv4 datagram carrying a IPv6 datagram in its payload is signaled by **protocol number 41**: this protocol number lets IPv6 nodes know that the original datagram must be extrapolated from the IPv4's payload.
Note that when an IPv6 node begins tunneling with another IPv6 node and creates an IPv4 datagram, it puts the IP address of the next IPv6 router in this datagram's destination address, not the original destination address.
## 4.7 ICMP protocol
The **Internet Control Message protocol** (**ICMP**) is used by hosts and routers to communicate *network-level* information.
The ICMP protocol is **above** the IP protocol: ICMP messages are carried *inside* IP datagrams, as IP payload.
But the IP protocol itself heavily relies on the ICMP protocol.

The ICMP messages contain:
- type
- code
- the header and the first 8 bytes of the IP datagram that caused ICMP message

| Type | Code | Description               | notes                                                                                                      |
| ---- | ---- | ------------------------- | ---------------------------------------------------------------------------------------------------------- |
| 0    | 0    | echo reply (ping)         |                                                                                                            |
| 3    | 0    | dest. network unreachable |                                                                                                            |
| 3    | 1    | dest host unreachable     |                                                                                                            |
| 3    | 2    | dest protocol unreachable |                                                                                                            |
| 3    | 3    | dest port unreachable     |                                                                                                            |
| 3    | 6    | dest network unknown      |                                                                                                            |
| 3    | 7    | dest host unknown         |                                                                                                            |
| 4    | 0    | source quench (not used)  | not used anymore: used for rate control, for congestion ("you are going to fast, I'm gonna lose packets!") |
| 8    | 0    | echo request (ping)       |                                                                                                            |
| 9    | 0    | route advertisement       |                                                                                                            |
| 10   | 0    | router discovery          | router announces itself as router                                                                          |
| 11   | 0    | TTL expired               | when TTL (hop limit) expires                                                                               |
| 12   | 0    | bad IP header             | IP header can get corrupted -> checksum fails                                                              |
# 5 Generalized Forwarding and SDN (4.4)
What we have seen until now can be named "Destination Based Forwarding": the only thing it cares about is the destination IP address.
**Generalized Forwarding** is instead a significantly more general "*match plus action*" approach: we can use every information included in a datagram and take different actions based on said information. Examples of actions can be: **forward**, **drop**, **copy**, **modify** and **log**. Since generalized forwarding is **software defined networking**, in this approach routers are better referred to as **packet switches**, with their forwarding tables being computed, installed, and updated by remote controller.
## 5.1 OpenFlow
Our following discussion of generalized forwarding will be based on **[[OpenFlow]]**, a highly visible and successful standard that has pioneered the notion of the match-plus-action forwarding abstraction and controllers, as well as the SDN revolution more generally.

To implement this new approach we can't use forwarding tables anymore, we need a new, more powerful tool: the **flow table**.
A flow table's entry—also known simply as **flow**—contains a **set of header field values** to which an incoming packet will be matched, a **set of counters** that are updated as packets are matched to flow table entries and a **set of actions to be taken** on match.

Flow table:

| match                        | action               |
| ---------------------------- | -------------------- |
| `src=*.*.*.*, dest=3.4.*.*`  | `forward(2)`         |
| `src=1.2.*.*, dest=*.*.*.*`  | `drop`               |
| `src=10.0.0.3, dest=*.*.*.*` | `send to controller` |
#### Match
OpenFlow 1.0 packet matching fields can be grouped by layer: there are link layer fields (SRC MAC, DST MAC, ETH TYPE, VLAND ID, VLAN PRI), network layer fields (IP SRC, IP DST, IP PROTOCOL, IP TOS) and transport layer fields (TCP/UDP SRC PORT, TCP/UDP DST PORT). Obviously flows can have **wildcards** (`*`).
#### Action
Each flow table entry has a list of zero or more actions that determine the processing that is to be applied to a packet that matches a flow table entry. If there are multiple actions, they are performed in the order specified in the list.
The absence of actions in a flow table entry means the packet is to be dropped.
### 5.1.2 OpenFlow abstraction
This match+action abstraction unifies different kinds of devices:

| device   | match                                 | action                   |
| -------- | ------------------------------------- | ------------------------ |
| router   | longest destination IP prefix         | forward out a link       |
| switch   | destination mac address               | forward of flood         |
| firewall | IP addresses and TCP/UDP port numbers | permit or deny           |
| NAT      | IP address and port                   | rewrite address and port |

# 6 Middlebox
By the definition in RFC 3234: “**middlebox**: any intermediary box performing functions apart from normal, standard functions of an IP router on the data path between a source host and destination host”.
From this definition these are middleboxes:
- caches
- load balancers
- firewalls, IDS
- NAT
- application specific
- ecc..

Middleboxes were initially proprietary hardware solutions, now there are programmable middleboxes: middleboxes that expose APIs.
We are moving towards software programmable networks.
# 7 Where's the intelligence
- Traditional phone net: intelligence and computing are in the center, at the network switches
- Internet (pre 2005): intelligence and computing are at the edge
- Internet (post 2005): now we have programmable network devices, intelligence and computing are both at the network switches and at the edge.