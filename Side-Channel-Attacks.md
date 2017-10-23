# PHY Covert Channels: Can you see the Idles?

## Motivation
Network covert timing channels modulate the delay between packets to send messages. But these channels are implemented at layer 3 or above, so these delays are easily detected or prevented. However, if you have access to the physical layer of the network stack, it's possible to create virtually invisible network covert timing channels.

## Terms and Definitions
* **Covert channels** - methods of sending messages through non-information-transfer systems. For example, covert channels may allow a user to send information within established network protocols, without actually using the protocol itself. **The channel itself must be hidden**.
* **Storage channels** - the sender modulates the value of a storage location
* **Timing channels** - sender modulates system resources over time to send a message
* **Side-Channel Attack** - when a system inadvertantly leaks information via covert channel, and an attacker can determine something about the execution of the system
* **Network covert channels** - send hidden messages over legitimate packets by modifiying the delay in between messages or headers in the messages. The sender has secret information that she tries to send to a receiver over the network, nicluding a network interface (L1-2, kernel network stack (L3-4) and application (L5). 
* **Active adversary** - employs network applications like network jammers to reduce the possibility of covert channels
* **Passive adversary** - monitors packet information to *detect* covert channels
* **Supply chain attack**  - switches and routers between the sender and receiver are compromised and cabaple of forwarding hidden messages.
* **PHY** - [[the physical layer|Physical-Layer]]
* **inter-packet delay (IPD)** - the time difference between the first bits of two successive packets
* **interpacket gap (IPG)** - the time difference between the last bit of the first packet and the first bit of the next packet (IPD = IPG + packet transmission time)
* **homogeneous packet stream** - packets that have the same destination, size, and IPG. variance of IPGs and IPDs is 0
* **Bit Error Rate (BER)** - percentage of bits that were interpreted incorrectly

## Network Covert Timing Channels
The simplest form is sending sending or not sending packets in a pre-arranged interval. But synchronizing the sender and reciever is hard due to network delays and noise. 

But what if a long delay means 0 and a short delay is 1? 

### [[The Physical Layer|Physical-Layer]]
The IEEE 802,3 standard requires that at least twelve `/I/` characters must be inserted after every packet. A sender could change the number of `/I/` characters in between messages in order to convey information, and higher layers would have no way of knowing, since these characters are discarded before translation to higher levels. 

Unfortunately this would only work between 1 hop, since the first hope would discard these characters after translation. PHY would work in conjunction with a supply chain attack. This would constitute a *physical storage attack*.

## Chupja
*Chupja* is a physical layer network timing channel which acheives:

* high bandwidth - covert rate of many 10s/100s of bits/second
* robustness - how to deliver messages with minimal errors
* undetectability - hiding the existence of the channel

Sender and receiver share two parameters:

* `G` - the number of `/I/`s in the IPG that is used to encode and decode hidden messages
* `W` - how the sender and receiver synchronize
* `e` - the modulation parameter for `G`

If `e` is some agreed upon time, then 

$$
G_i = G - e if b_i = 0
$$
$$
G_i = G + e if b_i = 1
$$

Where $G_i$ is the *i*th interpacket gap between packet *i* and packet *i + 1*. When G+i is less than the min IPG (or 12 `/I/`s) it is set to 12.

If `e` is large enough, encoded messages will be preserved because most switches do not significantly perturb interpacket gaps.

To decode:

$$
b_i^p = 1 if G_i >= G
$$
$$
b_i^p = 0 if G_i < G
$$

$b_i^p$ prime may not be equal to $b_i$ due to noise.

**The receiver considers an IPG larger than W as a pause, and uses the next IPG to decode the next signal.**

## Evaluation
The larger `e` is, the lower the BER is.

Chupja achieves 81 kbit/second over 9 routing hops and thousands of miles over the internet, with a BER of less than 10%.
