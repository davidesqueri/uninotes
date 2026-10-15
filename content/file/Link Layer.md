---
number headings: auto, first-level 1, max 3, 1.1
---
#uni 
# 1 Introduction
In discussing the link layer, we’ll see that there are two fundamentally ­different types of link-layer channels. 
The first type are **broadcast channels**, which connect multiple hosts in wireless LANs, satellite networks, and hybrid fiber-coaxial cable (HFC) access networks. Since many hosts are connected to the same broadcast communication channel, a so-called medium access protocol is needed to coordinate frame transmission.
The second type of link-layer channel is the **point-to-point communication link**, such as that often found between two routers connected by a long-distance link, or between a user’s office computer and the nearby Ethernet switch to which it is connected.

In the following chapter we will refer to any device that run a link-layer as a **node** and to the communication channels that connect adjacent nodes as **links**. Over a given link, a transmitting node encapsulates the datagram in a **link-layer frame** and transmits the frame into the link.
#### Services Provided by the Link Layer (6.1.1)
We said the basic service of any link layer is to move a datagram from one node to an adjacent node over a single communication link. The link layer can also offer a variety of extra services, here follow some examples:
- **Framing**
- **Link Access**: A medium access control (MAC) protocol specifies the rules by which a frame is transmitted onto the link.
- **Reliable Delivery**
- **Error detection and correction**
#### Where the Link Layer is implemented (6.1.2)
The data layer is implemented in a **network interface card** (**NIC**), also called a **network adapter**, or on a chip, in each-and-every host. This NIC or chip attaches to the host's system buses. Because of the need for a NIC, the data link is a mix of hardware and software.
Ethernet, wifi card or chip implement both the data layer and the physical layer.

The sending side encapsulates a datagram in a frame and adds error checking bits, reliable data transfer, flow control eccetera and then transmits it onto the physical link. The link-layer controller on the receiving side looks for errors, manages flow control and extracts the datagram and passes it to the upper layer.
# 2 Error-Detection and Correction (EDC) Techniques
Typically, the data to be protected includes not only the datagram passed down from the network layer for transmission across the link, but also link-level addressing information, sequence numbers, and other fields in the link frame header.
The receiver tries to detect corrupted bits, at which point it can try to automatically correct the bits or ask the transmitter to re-send the corrupted frames.

Keep in mind that a link can be:
- **Half-Duplex**: nodes at both ends of link can transmit, but not at the same time: achieved over a physical link by putting at both end a **transceiver**, which is both a transmitter and a receiver.
- **Full-Duplex**: we put two lines of physical link, one is used to transmit by one end, the other line is used to transmit by the other end.

The link-layer frame includes **R**, the error detection bits (redundancy bits) typically included in the **trailer** (because these are calculated while transmitting the message), and **D**, the data protected by error checking, which may include header fields.
What we do is apply a certain *mathematical operation* to the data, and include the result in the encoded message. 
When the receiver gets the message, it applies the same operation to the D data bits, and then checks if the resulting redundancy bits are the same as the received redundancy bits.
So D and R are the sent bits and D' and R' are the received bits. If the result of the operation on D' is different from the received R' bits, the message changed and we have N errors. This could mean the D' bits are wrong, or the R' bits are wrong.

Here follow some of the commonly used operations.
## 2.1 Parity Checking
This method detects single bit errors.
Only one bit, the **parity bit**, is added as trailer to the D bits.
This parity bit can either be set when there is an even number of 1's, or an uneven number of 1's.

Parity checking can only detect single-bit errors, this means that if two bits flip, the error goes undetected.
If the probability of bit errors is small and errors can be assumed to occur independently from one bit to the next, the probability of multiple bit errors in a packet would be extremely small. In this case, a single parity bit might suffice. However, measurements have shown that, rather than occurring independently, errors are often clustered together in “**bursts**".

What we can do to mitigate this problem is divide the D bits into rows and columns, and compute the parity bit for each row and for each column. With this **two-dimensional parity** scheme, the parity of both the column and the row containing the flipped bit will be in error. The receiver can thus not only _detect_ the fact that a single bit error has occurred, but can use the column and row indices of the column and row with parity errors to actually identify the bit that was corrupted and _correct_ that error.
A single error in the parity bits themselves is also detectable and correctable. Two-dimensional parity can also detect (but not correct) any combination of two errors in a packet.
The ability of the receiver to both detect and correct errors is known as **forward error correction** (**FEC**) and is particularly valuable because it (1) decreases the number of sender retransmissions required and (2) allows for *immediate* error correction, avoiding the wait for the RTT propagation delay needed for retransmission.
## 2.2 Checksumming (6.2.2)
In checksumming techniques, the d bits of data are treated as a sequence of k-bit integers, then some sort of sum is applied to these integers and the result is included in the trailer.
The **Internet Checksum** is the most commonly used for internet applications.

Because transport-layer error detection is implemented in software, it is important to have a simple and fast error-detection scheme such as checksumming. On the other hand, error detection at the link layer is implemented in dedicated hardware in adapters, which can rapidly perform more complex (and useful) operations.
### 2.2.1 Internet Checksum
The sender treats the package as a sequence of 16-bit integers and sums them, the 1s complement of this sum then forms the internet checksum that is carried in the segment header.
The receiver itself calculates the internet checksum and compares it to the received one in the checksum field, if these are not equal an error is detected.
In the TCP and UDP protocols, the Internet checksum is computed over all fields (header and data fields included). In IP the checksum is computed over the IP header (since the UDP or TCP segment has its own checksum).
#### Cyclic Redundancy Check (CRC) (6.2.3)
**Cyclic Redundancy Check** (**CRC**) is a more powerful—and complex—error detecting technique.

Let **D** be the string of $d$ data bits and **G** be a bit pattern (**generator**) of r+1 bits agreed upon in advance by both sender and receiver (the MSB of G needs to be a $1$). For a given piece of data, _D_, the sender will choose _r_ additional bits, _R_, and append them to _D_ such that the resulting *d+r* bit pattern $D+R$ (interpreted as a binary number) is exactly divisible by _G_ (i.e., has no remainder) using modulo-2 arithmetic. The process of error checking with CRCs is thus simple: the receiver divides the *d+r* received bits by _G_. If the remainder is nonzero, the receiver knows that an error has occurred; otherwise the data is accepted as being correct.
All CRC calculations are done in modulo-2 arithmetic without carries in addition or borrows in subtraction. This means that addition and subtraction are identical, and both are equivalent to the bitwise exclusive-or (XOR) of the operands.
Therefore the formula for the *d+r* bit pattern is $D \cdot 2^r \ \text{XOR} \ R$. We can calculate *R* as $R=(D \cdot 2r)\%G$.

This operation can detect all burst errors that are less than r+1 bits and any *odd* number of bit errors.
Widely used in practice, in Ethernet, 802.11 Wifi and more.
# 3 Multiple Access Links and Protocols (6.3)
As we said in the [[#1 Introduction]], links can be point-to-point links or **broadcast links**: multiple sending and receiving nodes all connected to the same, single, shared broadcast channel.
Using a single shared Broadcast channel leads to **collisions**: a node receives two or more signals at the same time.
The problem of how to coordinate the access of multiple sending and receiving nodes to a shared broadcast channel is known as the **multiple access problem**, which to be resolved requires a **multiple access protocol**: an algorithm that determines how nodes share a channel.

Consider a multiple access channel (**MAC**) of rate $R$ bps, an ideal protocol should have the following characteristics:
1. when only one node wants to transmit, it can send at rate $R$.
2. *fair*: when M nodes want to transmit, each can send at an average rate of $R/M$.
3. *fully decentralized*, without a SPF.
4. *simple* (KISS, keep it stupid simple), so that it is inexpensive to implement.
## 3.1 MAC protocols taxonomy
We can classify just about any multiple access protocol as belonging to one of three categories: **channel partitioning protocols**, **random access protocols**, and **taking-turns protocols**.
### 3.1.1 Channel partitioning Protocols
Channel partitioning MAC protocols divide the channel into smaller pieces (time slots, frequency, code), then allocate each piece for exclusive use of a node.

In **time division multiple access** (**TDMA**) the channel is accessed in "rounds": each station gets a fixed length slot in each round (length = packet transmission time). 
In **frequency division multiple access** (**FDMA**) the channel is instead divided in different frequency slots.

Channel partitioning is appealing because it eliminates collisions and is perfectly fair: each node gets a dedicated transmission rate of $R/N$ bps. However, it has two major drawbacks. First, a node is limited to an average rate of $R/N$ bps even when it is the only node with packets to send. A second drawback in TDMA is that a node must always wait for its turn in the transmission sequence.

A third channel partitioning protocol is **code division multiple access (CDMA)**. CDMA assigns a different _code_ to each node. Each node then uses its unique code to encode the data bits it sends. If the codes are chosen carefully, CDMA networks have the wonderful property that different nodes can transmit _simultaneously_ and yet have their respective receivers correctly receive a sender’s encoded data bits (assuming the receiver knows the sender’s code)t. CDMA has been used in military systems for some time (due to its anti-jamming properties) and now has widespread civilian use, particularly in cellular telephony.
### 3.1.2 Random Access Protocols
In a random access protocol, a transmitting node always transmits at the full rate of the channel. When there is a collision, each node involved in the collision repeatedly retransmits its frame until it gets through without a collision. But when a node experiences a collision, it doesn’t necessarily retransmit the frame right away. _Instead it waits a random delay before retransmitting the frame_. Each node involved in a collision chooses independent random delays.

A random Access MAC protocol must specify:
1. how to detect collisions
2. how to recover from collisions
#### Slotted ALOHA
In the following description we make these assumptions: all frames are of exactly $L$ bits, time is divided into equal size slots (the time it takes to transmit 1 frame), nodes start to transmit only  at the beginning of a slot, *nodes are synchronized* (**slotted**) and, finally, if two or more nodes transmit in the same slot all nodes detect the collision.

In **slotted aloha** when a node obtains a fresh frame it transmits it in the next available slot: if there's no collision the node can send also the new frame in the next slot.
If, instead, there's a collision, the node at every new slot retransmits the frame with probability $p$, until it actually sends it: hopefully the two colliding nodes "choose" different slots to retransmit and all goes well.

The main advantage of slotted aloha over channel partitioning is, when only one node desires to transmit, it can do so at full link throughput. It is also decentralized—apart from the synchronization—and extremely simple.
##### Best value for $p$
Suppose: $N$ nodes with many frames to send, each transmits in a slot with probability $p$.
- probability that a given node has a succesful transmission in a slot: $p(1-p)^{N-1}$ 
- probability that any node has success: $Np(1-p)^{N-1}$
- so to find the $\overline p$ that maximizes $Np(1-p)^{N-1}$ for a large $N$
	- $\lim_{N \to \infty} N \overline p (1-\overline p)^{N-1}$
	- max efficiency: $\overline p = 1/e=0,37$ 

This means the actual best throughput one can achieve with slotted aloha is $0.37$ times the maximum rate for the link.
#### (Unslotted) ALOHA
Designed by professor Abrahamson. Aloha means "hello" in Hawaian.
**Unslotted aloha** is the original version of the aloha protocol, that doesn't need synchronization between nodes:
When a node obtains a fresh frame it transmits it immediately, the probability of collision increases without synchronization, in the slotted version only a full collision can happen, in the unslotted version only a small part of the frame may collide and invalidate the whole frame:
	- a frame sent at $t_0$ collides with another frame sent in $[t_0 -1 ,t_0+1]$ 
##### Best value for $p$
Redoing the math we find that the optimal $p$ is $\overline p = 0,18$, much worse than [[#Slotted ALOHA]].
#### CSMA
**Carrier sense multiple access** (**CSMA**)'s underlying idea is to listen to the channel before transmitting, so as to not interrupt others.
Problem is collisions can still occur with carrier sensing: the propagation delay means two nodes may not hear each other's just started transmission.

We need to introduce **collision detection**: in **CSMA/CD** nodes detected when they have collided and immediately abort their transmissions.
Collision detection is easy with a wired channel, difficult in wireless.
##### Algorithm
1. NIC receives a datagram from the network layer, and creates a frame.
2. if the NIC senses that the channel is idle it starts frame transmission, otherwise, if it is busy, it waits until the channel is idle, then transmits.
3. if the NIC transmits the entire frame without collision, the NIC is done with the frame and the transmission is considered successful.
4. if the NIC detects another transmission while sending, it aborts and sends a **Jam** signal (a signal stronger than normal transmissions)
5. after aborting, the NIC enters the ***binary exponential backoff***: after the $m$th collision, the NIC chooses a $K$ at random from $\{ 0,1,2,...,2^m-1\}$ (in ethernet, $m$ is capped at $10$). The NIC waits $k*512$ bit times, then returns to `step 2`.

This leads CSMA/CD to have an efficiency of $E=\frac{1}{1+5\frac{d_{prop}}{d_{trans}}}$.
### 3.1.3 Taking-Turns Protocols
**Taking-turn protocols** coordinate access to a shared channel by giving devices opportunities to transmit. Some assign turns explicitly, such as polling or token passing; others let devices contend for access.
#### Polling Protocol
In the **polling protocol** a master node polls each of the nodes in a round-robin fashion: it lets each of them know when and for how many frames they can transmit over the shared channel. The key difference between polling and TDMA is here no slots get wasted, and the initial delay due to the wait for the assigned slot is diminished, although not eliminated.

Polling presents the following problems: polling overhead, latency and SPF.
#### Token Ring
In the **token ring protocol** the control **token** is passed from one node to the next sequentially. When a node receives the token it can transmit on the channel up to a certain maximum number of frames, otherwise it must wait for its turn. When a node with the token finishes its transmission (or it has no data to transfer when the token arrives) it gives up the token to the next node in line.
The token ring protocol is highly decentralized but still presents problems: a node could have problems with the release of the token, causing the halt of transmission for the whole channel.
One solution could be to have the node tasked with emitting the token in the network at some point recognize the failure and emit another token in the network, restoring functionality.
# 4 LANs
**Local Area Networks** (**LANs**) are a broadcast medium that utilize **MAC protocol** addressing, where MAC stands for **Medium Access Control**.
These cover a limited area (home, building, uni campus) and have a high bit rate (100Mbps - 10 Gbps).
## 4.1 MAC addresses
A **MAC address** is a **48 bit** (for most LANs) address burned in the **NIC** ROM (sometimes software settable), it is used "locally" to get a frame from one interface to another physically-connected interface. "MAC addresses" are also known as "LAN addresses" and "Physical Addresses", all these therms refer to **link-layer addresses**. MAC addresses are typically expressed in hexadecimal notation, where each byte is expressed asa pair of hexadecimal numbers. 
Every interface in a LAN has a unique MAC address and a **locally** unique 32-bit IP address — apart from link-layer switches, which do not have link-layer addresses associated with their interfaces.
The MAC address allocation is administered by the [[IEEE]]: every manufacturer buys a chunk of the MAC address space, where each chunk is of $2^{24}$ addresses, to assure uniqueness. This means when the IEEE allocates a chunk for a company, the first 24 bits are fixed, and the company can change the other 24. Although MAC addresses were designed to be permanent, it is now possible to change an adapter’s MAC address via software.

When an adapter wants to send a frame to some destination adapter, the sending adapter inserts the destination adapter’s MAC address into the frame and then sends the frame into the LAN. An  adapter may receive a frame that isn’t addressed to it. Thus, when an adapter receives a frame, it will check to see whether the destination MAC address in the frame matches its own MAC address. If there is a match, the adapter extracts the enclosed datagram and passes the datagram up the protocol stack. If there isn’t a match, the adapter discards the frame, without passing the network-layer datagram up, without interrupting the destination host's work. Note that it exists a special **MAC broadcast address**: 48 consecutive 1s, i.e. `FF-FF-FF-FF-FF-FF`.
### 4.1.1 Address Resolution Protocol (ARP)
Because there are both network-layer addresses and link-layer addresses , there is a need to translate between them, enter the **Address Resolution Protocol** (**ARP**).
An ARP module in a host host takes any IP address *on the same LAN* as input, and returns the corresponding MAC address.
Each host and router has an **ARP table** in its memory, which contains mappings of IP addresses to MAC addresses, with their relative TTL (typically 20 minutes).
#### Resolving IP addresses in the same LAN
If the ARP table does not contain a certain mapping for a desired destination IP address, the sender uses the ARP protocol to resolve the address. First, the sender constructs a special packet called an **ARP packet**. An ARP packet has several fields, including the sending and receiving IP and MAC addresses. The purpose of the ARP query packet is to query all the other hosts and routers on the subnet to determine the MAC address corresponding to the IP address that is being resolved. The sender passes an ARP query packet to the adapter along with an indication that the adapter should send the packet to the MAC broadcast address. The adapter encapsulates the ARP packet in a link-layer frame, uses the broadcast address for the frame’s destination address, and transmits the frame into the subnet. 
The frame containing the ARP query is received by all the other adapters on the subnet, and (because of the broadcast address) each adapter passes the ARP packet within the frame up to its ARP module. Each of these ARP modules checks to see if its IP address matches the destination IP address in the ARP packet. The one with a match sends back to the querying host a response ARP packet with the desired mapping. Now the sender updates its ARP table with this newly found info and can finally send its original datagram to the now-known destination MAC address.
Note that both ARP query and response packets have the same format.
#### Sending a datagram off the Subnet
When the sender tries to send a datagram to a host in another subnet, what MAC destination address should it insert in the frame header? The answer is the MAC address off the first-hop router on the path to the destination. This first receiving NIC (the one on the first-hop router) recognizes its MAC address on the frame and therefore passes the included datagram to the network layer, which in turn encapsulates it in a frame with the MAC address of the next-hop router as MAC destination address, and so on and so forth.
Obviously whenever a MAC address is missing, ARP is used.
## 4.2 Ethernet
The **Ethernet protocol** is the dominant, and first widely used, technology for wired LAN, this because it is fast and a single ethernet NIC supports multiple speeds.
Ethernet is both a link-layer _and_ a physical-layer specification and is carried over a variety of physical media including coaxial cable, copper wire, and fiber.
## 4.3 Physical topology for LANs
The first widely used physical topology for LANs was a hub-based star topology.
A **hub** is a physical-layer device that acts on individual bits rather than frames. When a bit, representing a zero or a one, arrives from one interface, the hub simply re-creates the bit, boosts its energy strength, and transmits the bit onto all the other interfaces. In particular, if a hub receives frames from two different interfaces at the same time, a collision occurs and the nodes that created the frames must retransmit. A topology with all nodes in the same collision domain is known as a **bus**.

In the early 2000s Ethernet installations continued to use a star topology, but the hub at the center was replaced with a layer-2 **switch**, which is not only “collision-less” but is also a bona-fide store-and-forward packet switch — this is a **switched** topology.
## 4.4 Ethernet frame structure
The sending interface encapsulates the IP datagram (or some other network layer protocol's packet) in a **Ethernet Frame**:
```
| preamble | dest.address | src.address | type | data(payload) | CRC |
```
- **Preamble**: used to synchronize the receiver and the sender clock rates. It is made of 7 bytes of 10101010 followed by one byte of 10101011. This is needed because although two NICs would like to transfer at a certain speed, nothing is absolutely perfect and the sending adapter will not transmit the frame at exactly the target rate; there will always be some _drift_ from the target rate, a drift which is not known _a priori_ by the other adapters on the LAN. A receiving adapter can lock onto adapter A’s clock simply by locking onto the bits in the first 7 bytes of the preamble.
- **Source and Destination addresses**: these are 6 byte - 48 bit MAC addresses. If the receiver's adapter receives a frame with it's destination address, or with the broadcast address, it passes the data in the frame to the network layer.
- **type**: this field indicates the higher layer's protocol, mostly IP, and it is used to demultiplex up at the receiver (for example ARP uses a value of `0x0806` for this field).
- **data**: the dimension of the payload must be between 46 bytes and 1500 bytes (MTU), this field carries the IP datagram. If the datagram is too big it is fragmented, if instead it is too small **stuffing** is used (the length field indicates the actual length, not counting the stuffing).
- 4 bytes of [[#Cyclic Redundancy Check (CRC) (6.2.3)]]. Note that although Ethernet uses CRC, it is NOT reliable, this because when the receiving NIC receives a frame that fails the CRC check, it simply discards it.
## 4.5 Ethernet CSMA/CD protocol
Ethernets uses **unslotted CSMA/CD with binary backoff** as it's MAC protocol:
1. The NIC receives the data from the network layer and creates a frame
2. if the NIC senses that the channel is idle for **96 bit** it starts the transmission of the frame, otherwise it waits
3. if the NIC transmits the entire frame without collision, the NIC is done with the frame
4. if the NIC detects another transmission while transmitting it aborts and sends a **48 bit** jam signal
5. After aborting, the NIC enters **exponential backoff**:
	   after the $i$-th collision:
	   1. the NIC chooses $K$ at random from $\{ 0,1,2,...,2^{m}-1\}$, where $m=\max(i,10)$.
	   2. the NIC sets a backoff timer at $k\cdot 512 \ \text{bit times}$
	   3. the NIC waits until the backoff timer expires
	   4. NIC returns to step 2
- After 17 trials the frame is dropped.
- $\text{bit time} =  0,1 \mu \text{sec}$ for 10 Mbps ethernet
# 5 Link Layer Switches
The role of the switch is to receive incoming link-layer frames and forward them onto outgoing links, the switch itself is **transparent** to the hosts and routers in the subnet.
Switches' output interfaces may have buffers, much like routers.

To prevent the cycling of broadcast frames, the active topology of a switched network is restricted to a spanning tree. Also, a large switched network would require large ARP tables in the hosts and routers and would generate substantial ARP traffic and processing. Furthermore, switches are susceptible to broadcast storms—if one host goes haywire and transmits an endless stream of Ethernet broadcast frames, the switches will forward all of these frames, causing the entire network to collapse.
With routers instead packets are not restricted to a spanning tree and can use the best path between source and destination. Because routers do not have the spanning tree restriction, they have allowed the Internet to be built with a rich topology.
## 5.1 Ethernet Switch
The switch is a **link layer** device, so it takes an active role: it stores and forwards ethernet frames.
- it examines the incoming frame's MAC address and selectively forwards the frame to one or more outgoing links, if the frame is to be forwarded on the segment.
- it uses CSMA/CD to access the segment
- switches are transparent: the hosts are unaware of the presence of the switches
- switches are plug-and-play and self-learning
### 5.1.1 Forwarding and Filtering
**Filtering** is the switch function that determines whether a frame should be forwarded to some interface or should just be dropped. **Forwarding** is the switch function that determines the interfaces to which a frame should be directed, and then moves the frame to those interfaces. Switch filtering and forwarding are done with a **switch table**. The switch table contains entries for some, but not necessarily all, of the hosts and routers on a LAN. An entry in the switch table contains (1) a MAC address, (2) the switch interface that leads toward that MAC address, and (3) the time at which the entry was placed in the table.

When a frame is received at the switch:
1. the switch records incoming link and MAC address of the sending host.
2. indexes the switch table using the MAC destination address.
3. **if** the entry is found for the destination and the destination is on the interface from which the frame arrived **then** drop the frame.
4. **else if** the entry is found and there is no loop: forward the frame on the interface indicated by the entry
5. **else** (entry not found) flood: forward on all interfaces except the one the frame arrived from.
### 5.1.2 Self-Learning
We said the switch is self-learning, because it *learns* which hosts can be reached through which interfaces and builds its switch table by itself, without manual intervention needed: when a frame is received, the switch learns the location (connected interface) of the sender (sender's MAC address) and it records the sender/location pair in the switch table (together with a timestamp). This property means switches—as opposed to routers—are **plug-and-play**.
### 5.1.3 Multiple Simultaneous Transmissions
Every input port of the switch has a dedicated direct connection to every output port of the switch. The switches can buffer the packets.
The ethernet protocol is used on each incoming link, so the are no collisions and **full duplex** is supported: each link is its own collision domain.
All of this means that on a single switch there can be $N$ simultaneous transmissions (as long as every host included in said transmissions is in only one transmission).
# 6 Virtual LANs (VLANs)
Modern institutional LANs are often configured hierarchically, with each workgroup (department) having its own switched LAN connected to the switched LANs of other groups via a switch hierarchy. While such a configuration works well in an ideal world, three drawbacks can be identified: **lack of traffic isolation**, **inefficient use of switches**, **bad user management** — as LAN sizes grow we get efficiency, security and privacy issues.
We also wish to limit the broadcast to a subsection of a LAN.

Each of these difficulties can be handled by a switch that supports **virtual local area networks** (**VLAN**s). As the name suggests, a switch that supports VLANs allows multiple _virtual_ local area networks to be defined over a single _physical_ local area network infrastructure. Hosts within a VLAN communicate with each other as if they (and no other hosts) were connected to the switch.
In a **port-based VLAN**, the switch’s ports (interfaces) are divided into groups by the network manager. Each group constitutes a VLAN, with the ports in each VLAN forming a broadcast domain.
It is also possible to define VLANs based on MAC addresses: the network manager specifies the set of MAC addresses that belong to each VLAN. VLANs can also be defined based on network-layer protocols (e.g., IPv4, IPv6, or Appletalk) and other criteria.

We now have to make different VLANs able to communicate. One way could be to connect the switch ports of different VLANs to the same router, which will route the packets from different VLANs, this approach is called **router-on-a-stick**.
### 6.1.1 VLAN trunking and 802.1Q Ethernet
A more scalable approach to interconnect VLAN switches is known as **VLAN trunking**: a **Trunk Port** belongs to all VLANs and carries frames between VLANs defined over multiple physical switches.
We need a way to tell switches to what VLAN packets belong to, we cannot use normal Ethernet 802.1 frames, they must carry VLAN ID info.
The **802.1Q** frame consists of the standard Ethernet frame with a four-byte **VLAN tag** added into the header that carries the identity of the VLAN to which the frame belongs. The VLAN tag consists of a 2-byte **Tag Protocol Identifier** (**TPID**) field (with a fixed hexadecimal value of 81-00), a 2-byte **Tag Control Information** field that contains a 12-bit **VLAN identifier field**, and a 3-bit **priority field** that is similar in intent to the IP datagram TOS field.
The VLAN tag is added into a frame by the switch at the sending side of a VLAN trunk, parsed, and removed by the switch at the receiving side of the trunk.
# 7 Link Virtualization (6.5)
Virtualization is a powerful abstraction even in Computer Networks: in a **Virtual Link** the path from one node to another is regarded as a point-to-point virtual link, just like a physical point-to-point link.
## 7.1 MPLS
**Multiprotocol Label Switching** (**MPLS**) is a packet-switched virtual-circuit network. 
MPLS's goal is not to abandon the destination-based IP datagram-forwarding infrastructure for one based on fixed-length labels and virtual circuits, but to augment it by selectively labeling datagrams and allowing routers to forward datagrams based on fixed-length labels (rather than destination IP addresses) when possible. Importantly, this technique works hand-in-hand with IP, using IP addressing and routing.

A link-layer frame transmitted between MPLS-capable devices has a small MPLS header added between the layer-2 header and layer-3 header. RFC 3032 defines the format of the MPLS header for such links. The most importa field, obviously, is the **label**.
An MPLS-capable router is often referred to as a **label-switched router**, since it forwards an MPLS frame by looking up the MPLS label in its forwarding table and then immediately passing the datagram to the appropriate output interface.
# 8 Wide-Area Packet Switched Networks (WANs)
In **Wide-Area Packet Switched Networks** nodes get identified through a unique address, similar to the ethernet MAC address.
These types of WANs can be:
- **connectionless**: each packet is managed on an individual basis and does not arrive in order — also known as **datagram service**.
- **connection-oriented**: a **virtual circuit** is preliminarily established, after that all packets follow the same path and arrive in order
## 8.1 Connection Oriented Packet Switched WANs
In order to have a connectionful communication, we first need to establish (and define) the connection or, in our case, the **virtual circuit**.
##### Virtual Circuits
A virtual circuit is a path from source to destination described by **VC numbers**, one number for each link along the path. Formally a VC is made of entries in the forwarding tables of the routers (or switches) along the path.
A packet belonging to a VC carries the VC number, rather than the destination address. The VC number can be changed on each link and the new VC number comes from the forwarding table.

| Incoming interface | incoming VC # | outgoing interface | outgoing VC # |
| ------------------ | ------------- | ------------------ | ------------- |
| ...                | ..            | ..                 | ..            |
The switches must maintain some information on the state of the connection.
# 9 Datacenter Networks
Datacenters are locations with thousands of host in close proximity, that are not intended for use of clients, at least not directly.
Some challenges one can face when working with datacenter networks are:
- multiple applications, each serving a potentially massive number of clients
- reliability
- load balancing and load managing
- avoid all kinds of bottlenecks (processing, networking, eccetera)

Datacenter networks, top to bottom:
0. Border routers: these interconnect the tier-1 switches and connect the datacenter to the internet.
1. tier-1 switches: connect to (~16) Tier-2 switches below
2. tier-2 switches: connect to (~16) TOR switches below
3. Top of Rack (**TOR**) switches: these switches sit on top of server racks and connect to the server blades (the host).
4. Server Racks: inside a server rack are usually 20-40 server blades, these are the hosts.
# 10 Point-to-Point Protocol (PPP)
This is a protocol at the data link level (data layer) ([[Internet Protocol Stack]]). This protocol was defined in a period in which the [[TCP/IP stacks]] was only one of the possible internet architectures.
Features:
- packet framing
- error detection ([[#Cyclic Redundancy Check (CRC)]] algorithm)
- no error correction! no retransmission! -> *unreliable service* 
- **bit transparency**: no limitations on what bit patterns can be transmitted
- network layer address negotiation
- no point-to-multipoint communication
- no flow control
## 10.1 Frame
```
| flag | address | control | protocol | info | check | flag |
   1B       1B        1B       1,2B     varL    2,4B     1B
```
- flags are delimiters for the framing, and are set to `01111110`
- address is useless, and always set to $0xFF$. Historically address with every bit set means Broadcast.
- control does nothing, could be used in the future
- protocol specifies to which upper layer protocol to deliver the frame (e.g. IP)
- info is the data carried for the upper layer, and is so obviously of variable lenght
- check is for the [[#Cyclic Redundancy Check (CRC)]] for error detection

### 10.1.1 Byte stuffing/unstufing
How do we ensure bit transparency? How do we make sure that info is not confused with the escape flag
We use an escape sequence:
to transmit `01111110` the transmitter sends `01111101 01111110`.

So the sequence `01111101` followed by either `01111110` or `01111101` tells the receiver to not interpret these last two as escape or flag, just data.
#### Sender (byte stuffing)
- 01111110 -> 01111101 01111110
- 01111101 -> 01111101 01111101
#### Receiver (byte unstuffing)
- 01111101 01111110 -> 01111110
- 01111101 01111101 -> 01111101