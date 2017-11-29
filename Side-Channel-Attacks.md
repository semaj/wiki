# Side Channels & Attacks

## Terms and Definitions
* **Side-channel** - an unintentional communication vector which leaks information about a computation
* **Covert channels** - methods of sending messages through non-information-transfer systems. For example, covert channels may allow a user to send information within established network protocols, without actually using the protocol itself. **The channel itself must be hidden**. The sender and receiver are both on the same "side".
* **Storage channels** - the sender modulates the value at a storage location
* **Timing channels** - the either intentional or unintentional leakage of information via the time given operations take
* **Side-Channel Attack** - when a system inadvertantly leaks information via covert channel, and an attacker can determine something about the execution of the system
* **Network covert channels** - send hidden messages over legitimate packets by modifiying the delay in between messages or headers in the messages. The sender has secret information that she tries to send to a receiver over the network, including a network interface (L1-2), kernel network stack (L3-4), and application (L5). 
* **Active adversary** - employs network applications like network jammers to reduce the possibility of covert channels
* **Passive adversary** - monitors packet information to *detect* covert channels
* **Supply chain attack**  - switches and routers between the sender and receiver are compromised and cabaple of forwarding hidden messages.
* **PHY** - [[the physical layer|Network-Stack]]
* **inter-packet delay (IPD)** - the time difference between the first bits of two successive packets
* **interpacket gap (IPG)** - the time difference between the last bit of the first packet and the first bit of the next packet (IPD = IPG + packet transmission time)
* **homogeneous packet stream** - packets that have the same destination, size, and IPG. variance of IPGs and IPDs is 0
* **Bit Error Rate (BER)** - percentage of bits that were interpreted incorrectly

## Examples

* Sound: that a printer makes can be analyzed to recover the words that were printed
* Time: measure the time required for computation to finish. If the time is correlated to the internal state of cmoputation, you can infer that internal state

### Web Pages
For example, given a website that includes a "forgot my password" link: when the backend server receives an HTTP request from the "forgot my password" page, let's assume it:

1. Looks up the email given
1. If the address is valid, the server generates a password recovery email and sends it to the address
1. Regardless, it sends a responds to the "forgot my password" page/client.

Since step #2 only occurs if the email exists in the database, an attacker can try various emails and depending on the time it takes for the response to occur, it can infer whether a given email exists in the system. This is a problem since many sites use email addresses as usernames.

Defenses:

1. Introduce a random delay: this only increases the number of samples an attacker must retrieve, since they can filter out the noise
1. Make all operations take constant time: this increases your load time
1. Add step 2 to a queue, rather than executing it serially: probably the best

### TENEX OS
The TENEX operating system required a process to provide a password string to system calls involving file access. It checked the password like so (assume password and password attempt are the same length for simplicity)

```
for (int i = 0; i <= strlen(pwGuess); i++) {
  if (pwGuess[i] != pwReal[i]) {
    return 0;
  }
}
return 1;
```

The attacker, in a sophisticated attack, can use page faults to determine if the guessed password characters are correct. In order to guess the *first* character of a password, the user (somehow?) allocates a 2-page long buffer and lays the password across the two pages such that *the first character of the password is on the first page, and the rest of the characters lay on the second page*. The attacker can force both of these pages into memory by reading the first character (the OS will read surrounding pages into memory). 

The check above will read the password one character at a time via these two pages. Before the loop is run, the attacker uses the system call `posix_fadvise(int fd, offset_t offset, offset_t len, POSIX_FADV_DONTNEED)`. This tells the operating system: "I don't think I'll need data so please evict it from memory, back to disk". 

If the first character is incorrect, the program will immediately return false since the first page is in memory. But if the attacker *evicts the second page* and the first character is **correct**, the loop will attempt to read from a page not in memory. This causes a page fault and forces the program to read the page from disk. Since this takes time, the attacker knows when the first character is correct and continue this process until the entire password is cracked.

# PHY Covert Channels: Can you see the Idles?

## Motivation
Network covert timing channels modulate the delay between packets to send messages. But these channels are implemented at layer 3 or above, so these delays are easily detected or prevented. However, if you have access to the physical layer of the network stack, it's possible to create virtually invisible network covert timing channels.

## Network Covert Timing Channels
The simplest form is sending sending or not sending packets in a pre-arranged interval. But synchronizing the sender and reciever is hard due to network delays and noise. 

But what if a long delay means 0 and a short delay is 1? 

### [[The Physical Layer|Network-Stack]]
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
G_i = G - e, if, b_i = 0
$$
$$
G_i = G + e, if, b_i = 1
$$

Where $G_i$ is the *i*th interpacket gap between packet *i* and packet *i + 1*. When *Gi* is less than the min IPG (or 12 `/I/`s) it is set to 12.

If `e` is large enough, encoded messages will be preserved because most switches do not significantly perturb interpacket gaps.

To decode:

$$
b_i^p = 1, if G_i >= G
$$
$$
b_i^p = 0, if G_i < G
$$

$b_i^p$ prime may not be equal to $b_i$ due to noise.

**The receiver considers an IPG larger than W as a pause, and uses the next IPG to decode the next signal.**

## Evaluation
The larger `e` is, the lower the BER is. As packets travel through routers, the routers will perturb the IPGs. We must pick a large `e`, which makes the channel robust against perturbation but easier for outsiders to detect.

64 byte ethernet frames are too small to support robust timing chains. As clusters of frames travel through the network, they get *compressed* as they wait in router queues. Let's say you're trying to send a message. You may attempt to use small (64 byte) packets to send more gaps (hidden messages). Unfortunately routers may end up queuing these small messages and upon translation, higher layers will compress these gaps. By sending larger real packets, you decrease the likelihood that routers will end up queueing too many of the real packets.

The paper decides that the ideal packet size is 1518 byte frames.

Chupja achieves 81 kbit/second over 9 routing hops and thousands of miles over the internet (assuming 1 Gbps), with a BER of less than 10%.
