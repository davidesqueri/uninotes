#uni 
([[Configurazione DHCP]])
The **Dynamic Host Configuration Protocol** dynamically gets an IP address for a Host from a server, when it "joins" a network.
This assigned address is only held while connected to the network (IP addresses have a due date, so the hosts need to renew their lease), this leads to the possibility of reuse of IP addresses, so it is better for mobile users who join and leave often a network.
Typically routers also include a DHCP server, serving every subnet to which the router is attached
# Overview
1. the host broadcasts (255.255.255.255:67) a *DHCP discover* `msg[optional]`, with a source IP address of `0.0.0.0`
2. the DHCP server responds (broadcast 255.255.255.255:68) with a *DHCP offer* `msg[optional]`, sends a possible IP address and a "lifetime"
3. the host requests an IP address with a *DHCP request* `msg`, sends an IP address and a "lifetime"
4. the DHCP server responds to the DHCP request with a *DHCP ack* `msg` 
5. the client can now begin using its newly found IP address and ask for a renewal for it when necessary.

- in every DHCP message, a transaction ID is included, so that even with multiple hosts, they don't get mixed up
	- steps 1 and 2 use one transaction ID, steps 3,4 use another one
- at step 3, now the host knows the address of the DHCP server that responded, but still uses broadcast, because there may be different DHCP servers on the network, and they also could listen and respond
- RFC 2131: the first two steps can be skipped if the client host wants to renew a lease, so already knows what IP address to request.

> the DHCP server can also return, together with the IP address, various info:
> - the address of the first-hop router for the client
> - name and IP address of the DNS server
> - the network (subnet) mask

# Allocation Modes
DHCP has three different allocation modes:
- **manual**: the administrator assigns the address to the host and the server provides the rest of the informations needed
- **automatic**: the server permanently assign an IP to the host
- **dynamic**: the server assign the IP address to a host for a limited amount of time (*leasing*)
# DHCP State Diagram
![[lab12.1 - DHCP (dragged).pdf]]
# Address Allocation
1. DHCPDISCOVER: host ask for available DHCP servers
2. DHCPOFFER: server respond with available addresses
3. DHCPREQUEST: host ask for one ip address
4. DHCPACK: server acknowledges the clients use of said address
	1. 50% of lease time passes -- client tries to renew the lease for its ip address
	2. DHCPREQUEST: host asks to renew lease
		1. DHCPACK: renew given
		2. DHCPNACK: address cannot be renewd
			1. DHCPREQUEST: for a different ip address
			2. ecc
5. DHCPRELEASE: the host gives up the ip address
# Configuring DHCP on a Cisco Router
