---
number headings: auto, first-level 1, max 3, 1.1
---
#uni 
- [1 Introduction](#1%20Introduction)
- [2 Multiplexing and Demultiplexing (3.2)](#2%20Multiplexing%20and%20Demultiplexing%20(3.2))
- [3 Connectionless transport: UDP (3.3)](#3%20Connectionless%20transport:%20UDP%20(3.3))
	- [3.1 UDP segment header](#3.1%20UDP%20segment%20header)
	- [3.2 UDP checksum (3.3.2)](#3.2%20UDP%20checksum%20(3.3.2))
- [4 Reliable Data Transfer protocols (3.4)](#4%20Reliable%20Data%20Transfer%20protocols%20(3.4))
	- [4.1 Stop-and-Go protocols](#4.1%20Stop-and-Go%20protocols)
	- [4.2 Pipelining](#4.2%20Pipelining)
		- [4.2.1 Error recovering strategies](#4.2.1%20Error%20recovering%20strategies)
- [5 Connection-Oriented transport: TCP (3.5)](#5%20Connection-Oriented%20transport:%20TCP%20(3.5))
	- [5.1 TCP segment structure (3.5.2)](#5.1%20TCP%20segment%20structure%20(3.5.2))
	- [5.2 Sequence Numbers and ACKs](#5.2%20Sequence%20Numbers%20and%20ACKs)
	- [5.3 Round-Trip Time Estimation and Timeout (3.5.3)](#5.3%20Round-Trip%20Time%20Estimation%20and%20Timeout%20(3.5.3))
	- [5.4 TCP reliable data transfer (3.5.4)](#5.4%20TCP%20reliable%20data%20transfer%20(3.5.4))
		- [5.4.1 Doubling the Timeout Interval](#5.4.1%20Doubling%20the%20Timeout%20Interval)
		- [5.4.2 TCP fast retransmit](#5.4.2%20TCP%20fast%20retransmit)
	- [5.5 TCP flow control (3.5.5)](#5.5%20TCP%20flow%20control%20(3.5.5))
	- [5.6 TCP connection Management (3.5.6)](#5.6%20TCP%20connection%20Management%20(3.5.6))
		- [5.6.1 The 3-way handshake](#5.6.1%20The%203-way%20handshake)
		- [5.6.2 Closing a TCP connection](#5.6.2%20Closing%20a%20TCP%20connection)
	- [5.7 Congestion Control (3.6)](#5.7%20Congestion%20Control%20(3.6))
		- [5.7.1 Approaches to congestion control (3.6.2)](#5.7.1%20Approaches%20to%20congestion%20control%20(3.6.2))
	- [5.8 TCP Congestion Control (3.7)](#5.8%20TCP%20Congestion%20Control%20(3.7))
		- [5.8.2 TCP cubic](#5.8.2%20TCP%20cubic)
		- [5.8.3 ECN: explicit congestion notification echo (3.7.2)](#5.8.3%20ECN:%20explicit%20congestion%20notification%20echo%20(3.7.2))
		- [5.8.4 TCP fairness (3.7.1)](#5.8.4%20TCP%20fairness%20(3.7.1))
- [6 Evolution of transport-layer functionality](#6%20Evolution%20of%20transport-layer%20functionality)
	- [6.1 Quic: quick UDP internet connections](#6.1%20Quic:%20quick%20UDP%20internet%20connections)

# 1 Introduction
The transport Layer manages the logical communication *between application processes running on different hosts*, whereas the network layer manages the logical communication *between hosts.

The Sender in transport protocols breaks application messages into **segments** and passes them to the *network layer*, which is going to form packets with them and make them arrive at destination. At the receiving end the network layer passes the received packets to the transport layer, which reassembles them into messages and demultiplexes them to the *application layer* via sockets.

Since the transport layer uses the network layer to actually bring its segments to the right hosts, there are certain services it cannot provide if they aren't enabled by the underlying network protocol — for example, delay or bandwidth guarantees. There are instead some services that even when not offered by the network, can be offered by the transport layer: reliable data transfer and encryption are only two examples.
Keep in mind that the internet protocol is a **best-effort delivery service**: it does not guarantee segment delivery, orderly delivery of segments and integrity of segment data on arrival.

The internet offers two transport layer protocols: TCP and UDP, which we are going to analyze later.
# 2 Multiplexing and Demultiplexing (3.2)
The fact that there are multiple processes running on the same host calls for the need for multiplexing and demultiplexing: which is the implementation of extending host-to-host delivery to process-to-process delivery.

Multiplexing is done at the sender side: data from multiple sockets is handled and put into segments, to which the respective transport layer header is added.
This transport header info is then used to deliver the received segment to the correct socket—the correct process.

Processes in a host use different ports, which are identified by unique **port numbers**: each port number is a 16 bit number ranging from 0 to 65535, where the port numbers from 0 to 1023 are called **well-known port numbers** and are restricted to well-known application protocols.
#### Connectionless demultiplexing
A UDP socket is fully identified by a two-tuple consisting of a destination IP address and a destination port number.
The host receives IP datagrams: each datagram carries one transport-layer segment, has a source IP address and a destination IP address and has source and destination port numbers.
At the receiving host, the transport layer uses the  *port number* to direct the message extracted from the segment to the appropriate socket.
#### Connection-Oriented demultiplexing
A TCP socket gets identified by a four-tuple:
- source IP address
- source port number
- destination IP address
- destination port number
The receiver uses *all four* of these values to direct the message extracted from the received segment to the appropriate socket.

This means that two arriving TCP segments with same destination but different source get routed to different sockets.
# 3 Connectionless transport: UDP (3.3)
UDP (**User datagram protocol**) is a "best effort" service, like IP.
Segments may be lost and delivered out of order to the application.
This because UDP is ***connectionless***: there is no handshake between UDP sender and receiver, each UDP segment is handled independently of others.

Some applications are better suited for UDP for the following reasons:
- no connection: no establishment delay, no connection state memory, more concurrent users
- no congestion control: faster sending rate
- smaller packet header overhead

This leads to UDP being useful for streaming and multimedia, [[DNS]], network management like SNMP and more.
## 3.1 UDP segment header
> UDP segment header: 
> ![[UDP_segment_structure.svg | 300]]

Since the Message (the application layer data) can be of variable length, we need to introduce a Length field, which states the size—in bytes—of the UDP segment, both header and message.
## 3.2 UDP checksum (3.3.2)
UDP needs to introduce error detection on an end-to-end basis since there is no guarantee that all links between source and destination provide error checking, some bits could even flip while being stored in routers.

The sender treats the contents of the UDP segment (header and IP address included) as a sequence of 16-bit integers, then the checksum is computed as the sum of these 16-bit integers (with overflows being wrapped around) and the 1s complement of this checksum value is put into the UDP checksum field.

The receiver computes the checksum as well and compares it to the checksum included in the header: if these are not equal and an error is detected, but nothing is done to correct it by UDP: some implementations discard this segment, others still deliver it to the application but with a warning.
# 4 Reliable Data Transfer protocols (3.4)
Before presenting TCP, let us analyze how we can achieve a **reliable data transfer** over a lossy and unreliable channel.
## 4.1 Stop-and-Go protocols
1. Reliable data transfer over a perfectly reliable channel
2. reliable data transfer over a channel with bit errors: 2.0(ACKs) 2.1(seq. numbers) 2.2(no NACK)
3. reliable data transfer over a lossy channel with bit errors: timer
#### RDT3.0 "Stop and Go" Performance evaluation
"Stop and Go": sending one packet and waiting for ACK.
$$
\begin{matrix}
D_{trans}= \frac{L}{R} \\
RTT= D_{trans}+ping \\
t=RTT+D_{trans} \\ 
U=\frac{D_{trans}}{t}
\end{matrix}
$$
But since $D_{trans} << ping$ we get that $U$ is extremely low.
## 4.2 Pipelining
The answer to the stop-and-go protocol's performance problem is sending multiple packets back to back, without waiting for an ACK for every packet.
To allow for pipelining we must increase the range of sequence number and give sender and receiver buffers.
### 4.2.1 Error recovering strategies
#### Go-back-N – GBN
In this protocol (**GBN**) the sender is allowed to transmit multiple packets (when available) without waiting for an acknowledgement, but is limited to a maximum of **N** unacknowledged packets in the pipeline.
We define **base** the sequence number of the oldest not acknowledged packet and **nextseqnum** the smallest unused sequence number (the sequence number of the next packet to be sent). So:
- the sequence numbers in $[0,\text{base}-1]$ correspond to acknowledged packets
- the interval $[\text{base},\text{nextseqnum}-1]$ corresponds to packets sent but not acknowledged
- the sequence numbers in the interval $[\text{nextseqnum},\text{base}+N-1]$ can be used for packets that can be sent immediately if data arrives from the upper layer
- the sequence numbers $\geq \text{base}+N$ cannot be used until an unacknowledged packet currently in the pipeline (the one with sequence number $\text{base}$) has been acknowledged.
This means the range of permissible sequence numbers can be viewed as a **sliding window** of size $N$ over the range of sequence numbers.
For this reason $N$ is called **window size** and the Go-Back-N protocol gets referred to as a **sliding window protocol**.

A packet's sequence number is carried in a fixed-length field in the packet header, suppose this field is $k$ bits in size, we get $2^k-1$ possible sequence numbers. When working with a finite range of sequence number, every operation must be done in modulo-2 arithmetic.

As we said the sender is allowed to send packets only while the windows is not full, one must choose what to do with this restriction: **buffering**, telling the upper layer to retry later or a **synchronization** mechanism that allows the upper layer to send data only when it's possibile are examples.

As for the receipt of ACKSs: an acknowledgement for a packet with sequence number $n$ will be taken to be a **cumulative acknowledgement** for every packet up until $n$.

The protocol's name is derived from the sender's behavior in the presence of lost or delayed packets: the sender keeps a timer for the oldest unACKed packet, if an ACK is received but there are still additional transmitted unACKed packets it restarts the timer, if every packet gets ACKed the timer is stopped, instead, when the timer expires, the sender *resends every unacked packet*.

When the receiver receives a packet, if it is the one it expected (the one following the last ACKed packet), it sends an ACK for it and delivers the data to the upper layer, in every other case the packet gets discarded and an ACK for the last accepted packet is sent.
The receiver *could* buffer out-of-order packets while waiting for the next in line, but if the desired packet gets lost, every packet up until the latter gets resent, so the buffering becomes useless.

GBN can however still stuffer from performance problems, for example while working with a large window and a big $\text{bandwidth} \cdot \text{delay}$ product whenever a package error happens a large amount of packets gets retransmitted, many unnecessarily.
#### Selective repeat – SR
Selective-Repeat protocols aim to solve retransmission-related problems in GBN protocols: the sender sends a maximum of $N$ packets starting from the **base** (the oldest not ACKed sent packet) in the pipeline and the receiver *ACKs individual packets*.
*The window slides on arrival of in-order packets* and out-of-order packets get buffered until any preceding missing packets have arrived, so to deliver them in order to the upper layer.

The sender implements a timer for every unacked packet: if the timer expires it retransmits all unACKed packets.
Note that on arrival of a duplicate packet, it must still be ACKed, since the sender could have not received the ACK for it and therefore resent it.

This is more efficient although it may happen (for example with a maximum sequence number of 4, if the sender sends 4 packets and the receiver sends 4 acks, all lost in transmission) that the sender sends a duplicate packet and the receiver accepts it, because the sequence number goes from 0 to windowSize.
A solution for this problem could be to make the max sequence number at least double the window size.
# 5 Connection-Oriented transport: TCP (3.5)
TCP (**transport control protocol**) is a connection-oriented (since an handshake is required before sending data) transport layer protocol. It introduces the **point-to-point** concept: one sender, one receiver, multicasting is not allowed.
It achieves a reliable, in-order *byte stream* from the application standpoint and provides a **Full duplex** communication: the flow is bidirectional in the same connection.

After a connection has been established, TCP passes the application's data to the TCP send buffer, which was allocated during the three-way-handshake, from where it will eventually be passed to the network layer and sent, at TCP's convenience (depending on flow control, congestion eccetera). When arriving at the receiver host, these segments will momentarily be stored at the TCP receive buffer, from where the application will read the byte stream of data from the sender.

Every segment has a **maximum segment size** (**MSS**): the maximum size of a segment's data field, typically set by first determining the largest link-layer frame that can be sent by the local sending host (called **maximum transmission unit** — **MTU**) and subtracting 20 bytes for the IP header and another 20 bytes for the TCP header, considering that both ethernet and PPP link-layer protocols have an MTU of 1500 bytes a typical MSS is 1460 bytes. Obviously when sending more data than what the MSS allows, the message will be broken into different segments, each respecting the maximum data field size.

TCP's congestion and flow Control sets the window size of the **pipelining** ([[Direct Connection Networks#3.2 Pipelining]]).
## 5.1 TCP segment structure (3.5.2)
> TCP segment structure: ![[TCP_segment_structure.svg|600]]

The **receive window** field is used for flow control: it indicates the number of bytes a receiver is willing to accept. The **options** field is used to negotiate MSS, used as a scaling factor, timestamping or used for other things, and it is of variable size (hence the header length field).
The flag field contains 8 bits:
- the **ACK** field is used to indicate that the acknowledgement field is valid
- the **RST**, **SYN** and **FYN** fields are for used for connection setup and teardown
- the **CWR** and **ECE** bits are used in explicit congestion notification
- the **PSH** bit indicates that the receiver should pass the data to the upper layer immediately
- the **URG** bit is set when the sender's upper layer entity has marked the data in the segment as urgent, it indicates that the **urgent data pointer** field is significant
## 5.2 Sequence Numbers and ACKs
TCP uses a Go-Back-$N$ error recovering approach with a little difference: the **sequence number** in a segment is the *byte stream number* of the first byte in the segment's data, because as we said TCP views data as an unstructured—but ordered—stream of bytes.
The **acknowledgement number** is instead the sequence number of the next byte it is expecting from the sender. Because TCP only acknowledges bytes up to the first missing byte in the stream, TCP is said to provide **cumulative acknowledgments**.
To minimize the possibility that a segment from a previous, terminated connection gets mistaken for a valid segment, *both sides of a TCP connection randomly choose an initial sequence number*.

TCP doesn't specify how the receiver must handle out-of-order segments, that is up to the implementor, it could either immediately discard out-of-order bytes or buffer them up until arrival of previous bytes—which is the approach taken in practice.
## 5.3 Round-Trip Time Estimation and Timeout (3.5.3)
Because TCP uses a timeout/retransmit mechanism to recover from lost segments, it needs to actually determine a suitable length for the timeout intervals, for example, it needs to be longer than the connection's RTT to avoid unnecessary retransmission.

This estimate is done circa once every RTT: TCP simply measures the time from the sending of a segment to the arrival of the corresponding ACK, not for every sent segment but only once at a time. Obviously this is done only on not-duplicate segments.

Because of the variation in the results TCP computes the **exponential weighted moving average** (**EWMA**): $$\text{estimatedRTT}_{n+1} = (1-\alpha)\cdot\text{estimatedRTT}_{n}+\alpha \cdot \text{sampleRTT}_{n}$$
where $\text{sampleRTT}_n$ is the last measured sampleRTT. The influence of past samples decreases exponentially fast.
A typical value for $\alpha$ is $0.125$ ($\frac{1}{8}$). The bigger $\alpha$ the more responsive the estimate (but less stable) — when computing the first ERTT ($\text{ERTT}_1$) we consider $\alpha = 0$.

In addition to an estimate of the RTT, it is also valuable to hold an estimate of the variability of the RTT as a EWMA of the difference between $\text{SampleRTT}$ and $\text{EstimatedRTT}$ (the recommended value for $\beta$ is $0.25$):
$$
\text{DevRTT}=(1-\beta)\cdot \text{DevRTT}+\beta \cdot \left | \text{SampleRTT}-\text{EstimatedRTT} \right |
$$

We can use this new value to determine a suitable timeout interval: since it must be greater than RTT and also depends on the variability of the latter we can compute it as $\text{TimeoutInterval}=\text{EstimatedRTT}+4\cdot \text{DevRTT}$. Note that an initial $\text{TimeoutInterval}$ of 1 second is recommended in RFC 6298.
If a timeout occurs, the value of $\text{TimeoutInterval}$ is doubled to avoid a premature timeout occurring for a subsequent segment that will soon be acknowledged. However as soon as a segment is received and $\text{EstimatedRTT}$ is updated, the $\text{TimeoutInterval}$ is again computed as above.
## 5.4 TCP reliable data transfer (3.5.4)
As we said the TCP protocol creates an RDT service on top of IP's unreliable service. For more info on RDT see [[#4 Reliable Data Transfer protocols (3.4)]], here we analyze TCP's RDT implementation.

| EVENT AT RECEIVER                                                                              | TCP RECEIVER ACTION                                                                |
| ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| arrival of in-order segment with expected seq# . All data up to expected seq# is already ACKed | delayed ACK. Wait up to 500ms for the next segment. If no next segment, send ACK   |
| arrival of in-order segment with the expected seq# . One other segment has pending ACK         | immediately send a single cumulative ACK, ACKing both the in-order segments        |
| arrival of out-of-order segment, higher than the expected seq# . *A Gap is detected*           | immediately send a *duplicate* ACK, indicating the seq# of the next expected byte  |
| arrival of segment that partially or completely fills the Gap                                  | immediately send ACK, provided that the segment starts at the lower end of the gap |
TCP uses a GBN error recovering approach, keeping a single timer for the oldest unACKed segment. 
Lets see two common modifications to TCP.
### 5.4.1 Doubling the Timeout Interval
A modification that most TCP implementations employ is as follows: whenever the timeout event occurs, TCP retransmits the not-yet-acknowledged segment with the smallest sequence number, but each time it retransmits it sets the next timeout interval to twice the previous value. The timeout is then set again as we have seen before when a segment normally arrives. This modification provides at least some sort of congestion control.
### 5.4.2 TCP fast retransmit
If the sender receives 3 additional ACKs for the same data ("triple duplicate ACKs"), resend the unACKed segment with the smallest sequence number, because it likely got lost, this way we don't have to wait for the timeout.
## 5.5 TCP flow control (3.5.5)
Recall that each host sets aside a receive buffer for the connection, when the TCP connection receives bytes that are correct and in-order it places them in the buffer. The associated application process will read the data from this buffer, but not necessarily immediately upon arrival.
When the network layer delivers data faster than the speed at which the application layer removes data from the socket buffers we get an overflow and we can lose segments:

We need to introduce some sort of **flow control**: the receiver controls the sender, so that no **overflow** occurs: the TCP receiver advertises the free buffer space in the `rwnd` ("receive window") field in the TCP header ([[#3.1 TCP segment structure (3.5.2)]]).
The sender limits the amount of unACKed ("in flight") data to the received `rwnd`, this way we guarantee that the buffer will not overflow: $\text{LastByteSent} - \text{LastByteAcked} \leq \text{rwnd}$.
We compute `rwnd` as follows: $\text{rwnd}=\text{RcvBuffer}-(\text{LastByteRcvd} - \text{LastByteRead})$, indicating the spare room in the buffer.

The receiver application process sets `RcvBuffer` (the size of the receive buffer) via socket options (the default is typically 4096 bytes), many systems auto-adjust `RcvBuffer`.

There is an issue with this scheme though: suppose the buffer becomes full so that $\text{rwnd}=0$, after advertising $\text{rwnd}=0$ to the sender, the application slowly empties the buffer, but if the receiver has no segments to send to the sender, the latter will never know that the is again room the in the receive window. To solve this issue the TCP specification requires the sender to continue to send segments with one data byte even while the receiver's receive window is 0. This way as soon as there is place in the receive buffer, the sender will know it and send new data.

> receiver buffer: ![[receiverWindow.svg]]

## 5.6 TCP connection Management (3.5.6)
Before exchanging data, the sender and the receiver must perform a handshake: they agree to establish a connection and agree on the connection parameters (for example the starting sequence numbers).
### 5.6.1 The 3-way handshake
When the sender wants to initiate a new connection, it sends a TCP segment — with no application-layer data — with the SYN bit set; for this reason, this special segments is called the **SYN segment**. Furthermore the client chooses — randomly — a initial sequence number and includes it in the segment.

Once this SYN segment arrives at the receiver, it allocates the TCP buffers and variables to the connection, and sends a connection-granted segment to the sender TCP. This new segment still has the SYN bit set, acknowledges the arrived SYN segment and advertises the newly — randomly — chosen receiver initial sequence number. Because this segment purely exists to acknowledge the connection request, it is called a **SYNACK segment**.

Upon receiving the SYNACK, the sender also allocates buffers and variables to the connection and sends another TCP segment, this time with the SYN bit *unset* since the connection is already established — this new segment can already include application-layer data.

Note that if the initial sequence number was always the same, we would risk mixing segments from two different connections, the solution is to start with a random initial sequence number.
### 5.6.2 Closing a TCP connection
Closing a connection means deallocating the data needed for it — let us see how two hosts decide on doing it:
1. the client sends a TCP FIN control segment
2. the server receives the FIN and replies with an ACK
3. the server closes the connection and sends a FIN
4. the client receives the FIN, replies with an ACK and starts a **timed wait** (designed to close the connection shortly after the supposed arrival of the ACK to the server — typically 30 seconds)
5. the server receives the ACK. The connection is now closed.
   If the server doesn't receive the ACK (it timeouts), it resends the FIN, this is the reason of the timed wait of the client.
## 5.7 Congestion Control (3.6)
Congestion is informally defined as follows: too many sources are sending too much data too fast for the *network* to handle — at the router's input ports are arriving more datagrams for time unit than how many datagrams for time unit the router can send.
This may happen either because the receiver has a too small input capacity, or because of internal congestion.

We want to avoid congestion because it brings long delays (because of queueing in router's buffers) and packet loss (buffer overflow at routers).
When too many senders overflow routers with segments, delay rises and as soon as the buffer is full, packet loss happens — which leads to retransmission, making congestion worse.

The goal of congestion-control is to throttle the transmission rate just enough to avoid congestion.
### 5.7.1 Approaches to congestion control (3.6.2)
We can distinguish between congestion-control approaches based on wether or not the network layer provides explicit support to the transport layer for congestion-control purposes.

In **network-assisted congestion control** routers provide direct feedback to every sending and receiving host with flows passing through congested routers. This feedback may indicate the congestion level or explicitly set a sending rate.
This approach is implemented by: ATM, DECbit and TCP ECN protocols.
Routers provide senders with feedback either through **choke packets** or — more typically — by marking incoming packets with congestion notifications before forwarding them, when a host receives a marked packet it will tell the sender that the network is congested. This latter method of notification takes a full RTT.

The preferred approach is **end-to-end congestion control**: the network core does not provide direct feedback, congestion is ***inferred*** from observed loss and delay — this is the approach taken by TCP.
## 5.8 TCP Congestion Control (3.7)
TCP uses (and must use) **end-to-end congestion control**, since the IP layer provides no explicit feedback to the end systems regarding network congestion.
The approach taken by TCP is to have each sender limit the rate at which it sends traffic into its connection as a function of perceived network congestion.
To do this TCP keeps track of another variable: the **congestion window** (**cwnd**). This variable (and `rwnd`) imposes a limit to the amount of unACKnowledged data: $\text{LastByteSent} - \text{LastByteAcked} \leq \min \{\text{cwnd},\text{rwnd}\}$. To simplify the discussion on congestion control we will assume `rwnd` to not be a problem and that the sender always has data to send.

![[cwnd.svg]]

Congestion is detected through packet loss: the sender learns that the network is congested either by receiving 3 duplicate ACKs from the receiver  or through a timeout. In TCP delay in ACK receipt is only used in determining how much it should increase the congestion window when not having packet loss.
Since TCP uses ACKs to trigger its increase in congestion window size it is said to be **self-clocking**.

So to summarize, TCP increments its congestion window based on ACK receipt delay up until packet loss occurs, at which point it decreases its transmission rate: this approach is called **bandwidth probing**.

TCP Congestion control is made of three phases: (1) **slow start**, (2) **congestion avoidance** and (3) **fast recovery**.
#### TCP's congestion control's 3 phases
When a connection begins, the sending rate is increased *exponentially* until the first loss event:
we start with $\text{cwnd}=1 \text{MSS}$ (resulting in a sending rate of $\frac{MSS}{RTT}$) and `cwnd` is then *doubled* every RTT (if an ACK is received).

When TCP detects a packet loss, it sets the variable `ssthresh` to half the `cwnd` when the loss was detected, then enters **slow start**.

**Slow start** goes as follows: starting from $\text{cwnd}=1\text{MSS}$, `cwnd` gets doubled once every `RTT`. When `cwnd` gets to the threshold value of `ssthresh`, slow start stops and TCP transitions into **congestion avoidance**.

When TCP enters congestion avoidance `cwnd` is half the size of what it was when the last packet loss was detected, this means we could be approaching congestion again, thus we need to be more careful and cannot continue doubling `cwnd`.
During **congestion avoidance** TCP increases `cwnd` by `MSS` bytes every `RTT`, resulting in a linear increase. This can be done by increasing `cwnd` on every ACK receipt: if at some point we send `n` segments every `RTT`, on ACK receipt we must increase `cwnd` by $\frac{1}{n}$ MSS, resulting in an increase of one `MSS` very `RTT`, up until a timeout happens (to which TCP responds by setting `ssthresh` at half the current value of `cwnd` and switching back to slow start) or a fast-recovery is triggered.

On triple duplicate ACKs receipt TCP halves the value of `cwnd` (and adds 3 `MSS`, accounting for the triple duplicate ACKs), records the value of `ssthresh` to be half the value of `cwnd` when the duplicate ACKs were received and finally enters **fast-recovery**.

In **fast-recovery** the value of `cwnd` is increased by 1 `MSS` for every duplicate ACK received for the missing segment that caused the fast-recovery, up until receipt of an ACK for the missing segment, which triggers TCP into entering **congestion-avoidance** again.
If instead a timeout is detected TCP returns to **slow-start**.
##### AIMD: Additive increase, multiplicative decrease
TCP congestion control is often referred to as an **additive-increase, multiplicative-decrease** (**AIMD**) form of congestion control.
**Additive increase**: the sending rate is increased by 1 maximum segment size every RTT, until loss is detected.
**Multiplicative Decrease**: the sending rate is cut in half when loss is detected by triple duplicate ACK , otherwise the sending rate is cut to 1 MSS when loss is detected by timeout.
AIMD has been found to have desirable stability properties, is distributed and asynchronous.
### 5.8.2 TCP cubic
There is a better way to "probe" for usable bandwidth:
We define as $W_{\max}$ the sending rate at which the congestion loss was detected, we suppose the congestion state of the bottleneck link probably hasn't changed much. 
After a loss we cut as we said the windows in half, then we initially ramp to $W_\max$ *faster* and we slow down the more we approach it.
This leads to the integral of the rate of transmission over time in TCP cubic is larger than the same integral in classic TCP, this means in average more segments are transmitted in the same time, leading to TCP cubic being the default in [[linux]] and the most popular TCP for popular Web Servers.

> transmission rate over time: ![[TCPcubicVSclassicTCP.svg]]
### 5.8.3 ECN: explicit congestion notification echo (3.7.2)
Modern TCP deployments often implement **network-assisted** congestion control, in the form of **explicit congestion notification**: 
1. the ECN bits in the IP datagram header's ToS (Type of Service) field get marked by the network router to indicate congestion — the policy for determining congestion is chosen by the network operator.
2. This congestion indication is carried to the destination.
3. The receiver sets the **ECE** (**explicit congestion echo**) bit on the TCP ACK segment, to notify the sender of the congestion.
4. The sender on receipt of a set ECE bit, halves `cwnd` and sets `CWR` (**control window reduced**) in the next TCP segment header.
### 5.8.4 TCP fairness (3.7.1)
Since rate increase is linear and rate decrease is multiplicative, the rate at which $n$ senders transmit segments through a same node gets slowly evened out to $\frac{\text{ max node rate}}{n}$, this means that TCP, under idealized assumptions, is fair (these assumptions are: same RTT for every sender, and fixed number of sessions only in congestion avoidance).
#### Transport services and fairness
Since UDP does not provide flow and congestion control, apps using it can send data at a constant, desired rate — this is not fair.
TCP applications in response can open **multiple parallel connections** between two hosts. Because of this lots of network administrators limit the amount of parallel connections between two hosts.
# 6 Evolution of transport-layer functionality
TCP and UDP have been the principal transport protocols for over 40 years. In this time the functionality of these protocols has been expanded with new application layer protocols.
## 6.1 Quic: quick UDP internet connections
This is an application-layer protocol on top of UDP.
It increases performance of HTTP by avoiding the TCP three way handshake and is deployed on many Google servers and apps.

QUIC employs:
- error and congestion control
- connection establishment: reliability, authentication, encryption
- multiple application-level "streams" multiplexed over a single QUIC connection

> TLS vs QUIC: ![[QUICvsTLS.svg]]