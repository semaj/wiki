# Physical Network Layer (L1)

The IEEE 802.3 standard describes the physical network layer, which is the physical medium at which packets are sent over the network.

On the transmit path, PHY encodes every 64 bits of an Ethernet frame into a 66-bit *block*.

Block (66 bits):
* **synchronization header** (2 bits)
* **payload** (64 bits)

If someone says their link is 10Gb/s, it's actually $10.3125 = 10G * \frac{66}{64}$, because we do not include the syncheader in the calculation.

The entire block is sent as a continuous stream of symbols which a 10Gb network transmits over a physical medium. Each bit is approximately 97 picoseconds wide (it takes approximately 97 picoseconds to send one bit.

When there is no message to send, PHY inserts `/I/` continuously until the next frame is available. The standard requires at least *twelve*. 

`/I/` consists of 7-8 bits, and takes about 700-800 picoseconds to transmit. They are discarded by hardware.
