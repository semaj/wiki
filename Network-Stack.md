# OSI Network Stack

| Layer Number | Protocol Level | Data Unit | Goal | Examples
| --- | --- | --- | --- | --- |
| 5 | Application | Application-specific | Application-specific | HTTP, FTP 
| | 4 | Transport | Datagram (UDP), Segment (TCP) | Overall, which application is this information for? TCP also provides reliability: it's an ordered data-stream. | TCP, UDP
| 3 | Network | Packet (IP) | How can the sender send information across multiple hops to a specific recipient? | IP (IP addresses)
| 2 | (Data) Link | Frame (Ethernet) | How can a sender send information across *one* hop? | Ethernet (802.3), WiFi (802.11) (MAC addresses)
| 1 | Physical | Bit | How are 0s and 1s encoded across a physical communication medium? | Radio, fiber, copper

Note that the traditional OSI network stack has **7** layers: the Presentation and Session layers have been omitted, since their functionality is usually folded into other layers (Transport and Application).

## Other terms

* **[[router|Network-Devices]]** - Traditional routers (at the ISP level) operate at the Network (L3) layer. They handle routing IP packets across large networks. Their goal is to connect multiple networks, but to treat them as separate entities.
* **[[switch|Network-Devices]]** - Switches are usually devices to which many clients are connected via the same type of Link (L2) connection. They handle routing packets to specific hardware ports at the link layer. A router may route a packet to a switch at which point the switch will route the packet to the specific host.
* **[[bridge|Network-Devices]]** - Bridges are (somewhat obsolete) devices that connect networks to form one larger network. You can think of a bridge as a switch under which multiple hosts may share a port. So if a bridge has two ports, it attempts to allow communication between the hosts on port 1 and the hosts on port 2.  In practice today, you would connect multiple switches which would handle the routing to specific hosts, since each hosts has its own ethernet port.

From [StackOverflow](https://serverfault.com/questions/78184/whats-the-difference-between-a-bridge-and-a-switch):

> An ethernet switch is a multiport ethernet bridge. A bridge is a device that splits collision domains but not broadcast domains. A switch is simply a bridge with lots of ports. Other examples of bridges are wireless access points and dual speed hubs. I don't think implementation (store&forward vs fast forwarding, software vs hardware, 2 ports vs many ports etc) makes it difference in kind, only a difference in degree (ie faster bridge or more ports on a bridge, etc).

> Ethernet was originally an "everyone sees all traffic" protocol. That's how traffic management happened -- if someone else is using the network, you wait until they're not; if two people try to use the network at the same time, both wait a random amount of time before attempting to use the network again. This was a "collision domain" or what people now call a "broadcast domain" because everything is switched and there are no more collisions (two simultaneous initiators of traffic).

> A bridge, in this context, only forwards traffic to stations on the other side of the bridge if it has learned that that station is on the other side of the bridge. If it hasn't seen the target MAC, it will send it over the bridge (flooding) or if it is a broadcast / multicast, it will also send it over the bridge.

> In ethernet, it is useful to remember how the technology was invented and deployed. First came shared media such as 10base5 and 10base2, both of which are coaxial cables that physically carry all traffic to all stations as an RF signal. Because vampire taps on 10base5 connections were expensive, people also used AUI repeaters that acted somewhat like hubs, but weren't. None of this equipment had any memory at all; the traffic went through or it didn't (and if it didn't the sender was expected to retransmit).

> Only much later did people start using twisted pair and deploying ethernet 10baseT hubs. A common topology was to use 10base5 as a building backbone and 10baseT to some locations, and connect different 10base5 backbone networks to each other using bridges or repeaters, depending on the traffic patterns and local budgets.

# Physical Network Layer (L1)

The IEEE 802.3 standard describes the physical network layer, which is the physical medium at which packets are sent over the network.

On the transmit path, PHY encodes every 64 bits of an Ethernet frame into a 66-bit *block*.

Block (66 bits):
* **synchronization header** (2 bits)
* **payload** (64 bits)

**Ethernet hosts are always sending information (the idle character) even when no packets are being ttransmitted**

If someone says their link is 10Gb/s, it's actually $10.3125 = 10G * \frac{66}{64}$, because we do not include the syncheader in the calculation.

The entire block is sent as a continuous stream of symbols which a 10Gb network transmits over a physical medium. Each bit is approximately 97 picoseconds wide (it takes approximately 97 picoseconds to send one bit.

When there is no message to send, PHY inserts `/I/` continuously until the next frame is available. The standard requires at least *12*. 

`/I/` consists of 7-8 bits, and takes about 700-800 picoseconds to transmit. They are discarded by hardware.
