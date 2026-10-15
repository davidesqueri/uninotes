#uni 
# Introduction
**Protocol Layering** offers many advantages: it provides a structured way to discuss system components and enables modularity, making it easier to update system components.

Some researchers are opposed to layering for many reasons, for example one layer may duplicate lower-layer functionality, or one layer may need information held only in another layer, which violates the goal of separation of layers.

**Protocol Stack**: the set of protocols of the various layers.
# The Internet protocol Stack
>the Internet Protocol Stack: ![[internetProtocolStack.svg|580]]

The **Internet Protocol Stack** consists of 5 layers:
- **application layer**: where the [[Network Applications]] (http, imap, smtp) operate. Packets of information at this layer are called **messages**.
- **transport layer**: transports application-layer messages between application endpoints: process to process data transfer (TCP: reliable, ordered delivery and connection oriented; UDP: connectionless). This layer's packets are called **segments**.
- **network layer**: this layer routes **datagrams** through a series of routers between the source and the destination using the IP address (IP protocol: the only network layer protocol, routing protocols).
- **link layer** (or data layer, or data link): this layer transfers **frames** between neighboring network elements using the MAC address (ethernet, 802.11 (wifi), PPP).
- **physical layer**: this layer moves the individual **bits** within the frame across the physical medium of a single link.
# ISO/OSI reference model
This model adds two layers that are not found in the internet protocol stack, where if they are needed they must be implemented in the application layer.
> iso/osi reference model:
>  ![[isoosireferencemodel.svg|180]]

