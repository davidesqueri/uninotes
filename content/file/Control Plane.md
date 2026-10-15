---
number headings: auto, first-level 1, max 3, 1.1
---
#uni
- [1 Introduction](#1%20Introduction)
- [2 Routing Algorithms (5.2)](#2%20Routing%20Algorithms%20(5.2))
	- [2.1 Routing protocols classification](#2.1%20Routing%20protocols%20classification)
	- [2.2 Link state Algorithms](#2.2%20Link%20state%20Algorithms)
		- [2.2.1 Dijkstra's algorithm](#2.2.1%20Dijkstra's%20algorithm)
	- [2.3 Distance Vector Algorithm](#2.3%20Distance%20Vector%20Algorithm)
- [3 Autonomous Systems (5.3)](#3%20Autonomous%20Systems%20(5.3))
	- [3.1 Intra-AS routing: OSPF](#3.1%20Intra-AS%20routing:%20OSPF)
	- [3.2 Inter-ASs routing: BGP (5.4)](#3.2%20Inter-ASs%20routing:%20BGP%20(5.4))
		- [3.2.1 Advertising BGP router information (5.4.2)](#3.2.1%20Advertising%20BGP%20router%20information%20(5.4.2))
		- [3.2.2 Determining the Best Routes (5.4.3)](#3.2.2%20Determining%20the%20Best%20Routes%20(5.4.3))
		- [3.2.3 IP-Anycast (5.4.4)](#3.2.3%20IP-Anycast%20(5.4.4))
# 1 Introduction
The **control plane** functions of the network layer compose the _network-wide_ logic that controls (1) how a datagram is routed among routers along an end-to-end path from the source host to the destination host and (2) how network layer components and devices are configured and managed.

As we said, **software-defined networking** explicitly separates data plane and control plane: the **remote controller** interacts with a **control agent** (**CA**) in each router and manages its flow tables.
This is opposed to the traditional way where the control is per-router distributed: the routing algorithms run in each router instead of on separated controllers and the CA of each router communicates with other CAs.
# 2 Routing Algorithms (5.2)
Routing algorithms must determine "**good**" (least expensive, least congested, fastest) **paths** (sequence of routers) from the sender to the receiver, through the network of routers.

This problem is abstracted via **graphs**, where `G=(N,E)` is the graph of nodes `N` connected by the edges `(x,y)` in `E`, where each edge has a different `c(x,y)` **cost** (representing physical length, speed, ecc). Edges not in `E` are indicated by a infinite cost: $c(x,y)=\infty$, two connected nodes are said to be *neighbors*. We will only consider *undirected graphs*.
Today the most accurate way to determine the link's cost is based on the retransmission needed on said link.
$$
\begin{matrix}
N: \text{router nodes}=\{ u,v,w,x,y,z \} \\
E: \text{links}=\{ (u,v),(u,x),(v,x), \text{ecc} \}
\end{matrix}
$$
## 2.1 Routing protocols classification
Routing Algorithms can be classified based on three criteria: (1) centralized versus decentralized, (2) static versus dynamic and (3) load-sensitive versus load-insensitive.

A **centralized routing algorithm** computes the least-cost path between source and destination using complete, global knowledge about the network. Algorithms with global state information are often referred to as **link-state (LS) algorithms**, since the algorithm must be aware of the cost of each link in the network.
In a **decentralized routing algorithm**, such as distance-vector algorithms, the calculation of the least-cost path is instead carried out in an *iterative*, *distributed* manner by the routers. No node has complete information about the costs of all network links. Instead, each node begins with only the knowledge of the costs of its own directly attached links. Then, through an iterative process of calculation and exchange of information with its neighboring nodes, a node gradually calculates the least-cost path to a destination or set of destinations.

In **static routing algorithms**, routes change very slowly over time, often as a result of human intervention (for example, a human manually editing a link costs). **Dynamic routing algorithms** change the routing paths as the network traffic loads or topology change. A dynamic algorithm can be run either periodically or in direct response to topology or link cost changes.

In a **load-sensitive algorithm**, link costs vary dynamically to reflect the current level of congestion in the underlying link. If a high cost is associated with a link that is currently congested, a routing algorithm will tend to choose routes around such a congested link. Modern routing algorithms are load-insensitive.
## 2.2 Link state Algorithms
In practice the global knowledge of link costs and topology is accomplished by having each node broadcast link-state packets to _all_ other nodes in the network, with each link-state packet containing the identities and costs of its attached links: we need a **link-state broadcast algorithm**.
### 2.2.1 Dijkstra's algorithm
This algorithm computes the least expensive route between a source node and every other, this means it returns the **forwarding table** for the source node. This is a iterative algorithm, after $k$ iterations we know the least expensive path from the source node to $k$ destinations.
Normally Dijkstra's algorithm has $O(n^2)$ complexity, but with the aid of the *heap* we can get it as low as $O(n\cdot \log n)$.
We also need to take into account the broadcast phase, this has $O(n^2)$ complexity.

- $D(v)$: current cost estimate for the path between the source node and the destination $v$ 
- $p(v)$: predecessor node along the path to $v$ 
- $N'$: set of nodes for which the least expensive path is known
## 2.3 Distance Vector Algorithm
Whereas the LS algorithm is an algorithm using global information, the **distance-vector (DV)** algorithm is iterative, asynchronous, and distributed. It is _distributed_ in that each node receives some information from one or more of its _directly attached_ neighbors, performs a calculation, and then distributes the results of its calculation back to its neighbors. It is _iterative_ in that this process continues on until no more information is exchanged between neighbors.

This algorithm is based on the **Bellman-Ford** (**BF**) equation (dynamic programming): 
$$
\begin{matrix}
D_x(y):\text{cost of the least-cost path from } x \text{ to } y. \\
\implies D_x(y)=\min_v \{ c_{x,v}+D_v(y) \} \quad \forall y \in N\\
\min_v: \text{minimum taken between }x\text{'s neighbors } v 
\\
c_{x,v}: \text{direct link cost of }x,v
\\
D_v(y):\text{least-cost path from } v \text{ to } y \text{ estimate}
\end{matrix}
$$
The idea is each node begins with an estimate of the cost of the least-cost patch from itself to every other node and broadcasts this vector of costs. When receiving everybody else's distance vector, it uses this new info to compute a better estimate of its distance vector with the Bellman-Ford equation, broadcasts this new version and so on.
In normal conditions, the $D_x(y)$ estimate converges to the least-cost $d_x(y)$.

Note that in distance vector algorithms **black-holing** is possible: routers can communicate incorrect path costs ("i have low cost paths to every node"), error gets propagated along the network.
#### Poisoned reverse
It could happen that a node $y$ doesn't know its cost to get to $x$, then it receives a distance vector from $z$ where it says he has a path to $x$, but it doesn't know that to do so it passes through $y$. Now $y$ will try to pass through $z$ to get to $x$ even though it has a shorter path, creating a cycle. 

The solution is a technique called **poisoned reverse**:
If $z$ passes through $y$ to get to $x$:
1. $z$ notifies $y$ that the cost $z,y$ is $\infty$ 
	1. $D_z(x)=\infty$ even if $z$ knows this is not the case
	2. until $z$ gets to $x$ passing through $y$, it will continue to announce $(z,x)=\infty$ 
2. $y$ will think that $z$ doesn't have a path to $x$, therefore it will not try to get to $x$ through $z$ and the cycle will not be created, leading to the costs being communicated correctly

Note that this technique works only for solving cycles creating between adjacent nodes.
# 3 Autonomous Systems (5.3)
This view of the network as simply a collection of interconnected routers has a few problems:
- **Scale**: as the number of routers becomes large, the overhead involved in communicating, computing, and storing routing information becomes prohibitive.
- **Administrative autonomy**: the Internet is a network of ISPs, with each ISP consisting of its own network of routers. An ISP generally desires to operate its network as it pleases (for example, to run whatever routing algorithm it chooses within its network) or to hide aspects of its network’s internal organization from the outside. Ideally, an organization should be able to operate and administer its network as it wishes, while still being able to connect its network to other outside networks.

Both of these problems can be solved by organizing routers into **autonomous ­systems (ASs)**, with each AS consisting of a group of routers that are under the same administrative control. Often the routers in an ISP, and the links that interconnect them, constitute a single AS. Some ISPs, however, partition their network into multiple ASs. An autonomous system is identified by its globally unique **autonomous system number** (**ASN**). AS numbers, like IP addresses, are assigned by ICANN regional registries.
Routers within the same AS all run the same routing algorithm and have information about each other. The routing algorithm ­running within an autonomous system is called an **intra-autonomous system routing ­protocol**.
## 3.1 Intra-AS routing: OSPF
**Open Shortest Path First** (**OSPF**) is widely used for intra-AS routing in the internet.
OSPF is a link-state protocol that uses flooding of link-state information and a Dijkstra’s least-cost path algorithm. With OSPF, each router constructs a complete topological map (that is, a graph) of the entire autonomous system. Each router then locally runs Dijkstra’s shortest-path algorithm to determine a shortest-path tree to all _subnets_, with itself as the root node.
A router broadcasts routing information to all other routers in the AS, not just neighbors, whenever link state changes and also periodically, even if nothing changed.
The OSPF protocol also checks that links are operational (via a `HELLO` message that is sent to an attached neighbor) and allows an OSPF router to obtain a neighboring router’s database of network-wide link state.

Individual link costs are configured by the network administrator. The administrator might choose to set all link costs to 1, thus achieving minimum-hop routing, or might choose to set the link weights to be inversely proportional to link capacity in order to discourage traffic from using low-bandwidth links; OSPF itself does not mandate a policy for how link weights are set.

OSPF offers: (1) **security** (exchanges between OSPF routers can be authenticated by using a password, either in plaintext or with MD5), (2) **Multiple same-cost paths** and (3) **integrated support for unicast and multicast routing**.
OSPF also enables **hierarchy within a single AS**: An OSPF autonomous system can be configured hierarchically into areas. Each area runs its own OSPF link-state routing algorithm. Within each area, one or more area border routers are responsible for routing packets outside the area. Lastly, exactly one OSPF area in the AS is configured to be the **backbone area**. The primary role of the backbone area is to route traffic between the other areas in the AS. The backbone always contains all area border routers in the AS and may contain non-border routers as well. Inter-area routing within the AS requires the packet to pass through the backbone to change area.
## 3.2 Inter-ASs routing: BGP (5.4)
When routing a packet between a source and destination within the same AS, the route the packet follows is entirely determined by the intra-AS routing protocol. However, to route a packet across multiple ASs, say from a smartphone in Timbuktu to a server in a datacenter in Silicon Valley, we need an **inter-autonomous system routing protocol**. Since an inter-AS routing protocol involves coordination among multiple ASs, communicating ASs must run the same inter-AS routing protocol. In fact, in the Internet, all ASs run the same inter-AS routing protocol, called the **Border Gateway Protocol** (**BGP**).

In BGP, packets are not routed to a specific destination address, but instead to CIDRized prefixes, with each prefix representing a subnet or a collection of subnets.

BGP provides each router a means to **obtain prefix reachability information from neighboring ASs** and **determine the best routes to the prefixes**.
### 3.2.1 Advertising BGP router information (5.4.2)
For each AS, each router is either a **gateway router** or an **internal router**. In BGP pairs of routers exchange routing information over semi-permanent TCP connections using port 179: a so-called **BGP connection**. A BGP connection that spans two or more ASs is called an **external BGP** (eBGP) connection, as opposed to an **internal BGP** (iBGP) connection.
Gateway routers advertise prefixes inside their AS to other gateway routers in other ASs over eBGP connections. The receiving gateway routers then propagate this info to all routers inside their AS over iBGP connections. At this point gateway routers propagate this AS route for the initially advertised prefix to other ASs (like `intermediateAS initialAS prefix`).
### 3.2.2 Determining the Best Routes (5.4.3)
When a router advertises a prefix across a BGP connection, it includes with the prefix several **BGP attributes**. In BGP jargon, a prefix along with its attributes is called a **route**. Two of the more important attributes are AS-PATH and NEXT-HOP.

The **AS-PATH attribute** contains the list of ASs through which the advertisement has passed. To generate the AS-PATH value, when a prefix is passed to an AS, the AS adds its ASN to the existing list in the AS-PATH.
BGP routers also use the AS-PATH attribute to detect and prevent looping advertisements; specifically, if a router sees that its own AS is contained in the path list, it will reject the advertisement.

The **NEXT-HOP attribute** is the _IP address of the router interface that begins the AS-PATH_.
#### Hot Potato Routing
Hot Potato Routing is one of the simplest routing algorithms for inter-AS routing: the route chosen (from among all possible routes) is that route with the least cost to the NEXT-HOP router beginning that route.
The idea behind hot-potato routing is for the gateway router to get packets out of its AS as quickly as possible (more specifically, with the least cost possible) without worrying about the cost of the remaining portions of the path outside of its AS to the destination: we therefore say that hot potato routing is a selfish algorithm.
#### Route-Selection Algorithm
In practice, BGP uses an algorithm that is more complicated than hot potato routing, but nevertheless incorporates hot potato routing.
For any given destination prefix, the input into BGP’s route-selection algorithm is the set of all routes to that prefix that have been learned and accepted by the router. If there is only one such route, then BGP obviously selects that route. If there are two or more routes to the same prefix, then BGP sequentially invokes the following elimination rules until one route remains:
1. A route is assigned a **local preference** value as one of its attributes. The local preference of a route could have been set by the router or could have been learned from another router in the same AS. The value of the local preference attribute is a policy decision that is left entirely up to the AS’s network administrator. The routes with the highest local preference values are selected.
2. From the remaining routes (all with the same highest local preference value), the route with the shortest AS-PATH is selected.
3. From the remaining routes, hot potato routing is used.
4. If more than one route still remains, the router uses BGP identifiers to select the route.
### 3.2.3 IP-Anycast (5.4.4)
BGP is often used to implement the IP-anycast service, commonly used in DNS. 
During the IP-anycast configuration stage, the CDN company assigns the _same_ IP address to each of its servers, and uses standard BGP to advertise this IP address from each of the servers. When a BGP router receives multiple route advertisements for this IP address, it treats these advertisements as providing different paths to the same physical location (when, in fact, the advertisements are for different paths to different physical locations). When configuring its routing table, each router will locally use the BGP route-selection algorithm to pick the “best” (for example, closest, as determined by AS-hop counts) route to that IP address.

For example when a client requests the video, the CDN returns to the client the common IP address used by the geographically dispersed servers, no matter where the client is located. When the client sends a request to that IP address, Internet routers then forward the request packet to the “closest” server, as defined by the BGP route-selection algorithm.

In practice CDNs generally choose not to use IP-anycast because BGP routing changes can result in different packets of the same TCP connection arriving at different instances of the Web server.
But IP-anycast is extensively used by the DNS system to direct DNS queries to the closest root DNS server.