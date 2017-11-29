# TOR (The Onion Router)

## Motivation
High-latency designs for anonymous communication date to Chaum's MixNets.

## Mix Networks
Imagine participants A and B, as well as a middleperson M. 

$$
A -> K_m(R1, K_b(R0, msg, K_m(S1, A), K_x), B) -> M \\
$$
$$$
M -> K_b(R0, msg, K_m(S1, A), K_x) -> B \\
$$
$$
B -> K_m(S1, A), K_x(S0, response) -> M \\
$$
$$
M -> A, S1(K_x(S0, response)) -> A
$$

This is the simplest case - mixnets randomly distribute traffic making it difficult to correlate sending and receiving.

## Terms and Definitions
* **Perfect forward secrecy** - when the compromise of keys or secrets does not allow you to compromise past conversations. For example, if during a TLS handshake the client and server negotiate a key via Diffie-Hellman, even if the attacker compromises the private keys, the ephemeral DH keys used in the exchange are gone, and thus previous sessions cannot be decrypted. In the realm of Tor: if the attacker logs the encrypted traffic between two onion routers AND the attacker learns the RSA keypairs of the logged encrypted traffic, the attacker cannot decrypt any of the logged encrypted messages.
* Protocol Normalization - trying to de-anonymize a specific protocol using the semantics of that protcol
* Cells - TOR "packets", usually fixed-size at 512 bytes. A few types are variable-sized.
* Circuit - a virtual connection between two end-hosts. Multiple circuits may be multiplexed over the same TLS connection.
* Stream - TCP flow that goes through a particular circuit. A single circuit may contain multiple streams.

**Diffie-Hellman Key-exchange**

`g` and `p` are publicly known.
$$$
X -> g^x mod p -> Y
Y -> g^y mod p -> X
$$$
Both calculate ephemeral $g^(xy)$, the key.

## Goals
**Tor seeks to frustrate attackers from linking communication partners, or from linking multiple communications to or from a single user.**

**To allow a client to exchange TCP data with a server without revealing the client's IP address to the server.**

**Tor can also provide anonymity to a webservice using "hidden services".**

Hiding your IP address is necessary but insufficient to hiding your *identity*. For example, it's possible to fingerprint you based on your browser features.

* It must not be expensive to run
* It must not place a high liability burden on operators
* It must not be difficult to implement or run
* It cannot require non-anonymous parties such as websites to modify their software
* It should not be difficult to use
* The protocol must be flexible
* The protocol's design and security parameters must be well understood
* It should be resistant to censorship

Non-goals

* It should not be fully peer-to-peer (there will be fewer, longer-lived servers)
* **It should not be secure against end-to-end attacks**
* It should not perform protocol normalization

## Threat Model
**Not a global passive adversary.** Tor protects against an attacker who

* Can generate, modify, delete, or delay traffic. 
* Can observe some fraction of network traffic.
* Can operate onion routers of his own.
* Can compromise some fraction of onion routers.

The attackers typical goal is **to observe both the initiator and the responder**, or **to correlate a request into Tor with a request made out of Tor.**

## Design
Onion routers are user-level processes. Each OR maintains TLS connections to other ORs. *Users* run onion proxies, which establish circuits across the network. They accept TCP streams from the user and multiplex them across the circuits.

Each onion router maintains
* A long-term identity key used to sign TLS certificates, router descriptors
* Short-term onion key which is used to decrypt requests from users to set up a circuit and negotiate ephemeral keys

The TLS protocol also establishes short-term link-keys when communicating between ORs. These short-term keys are rotated.

There are two types of cells: control cells and relay cells. Control cells are interpreted by the node that recieves them, and the relay cells carry end-to-end stream data. All cells have circuit IDs (circID) which specifies which circuit the cell refers to. **Many circuits may be multiplexed over a single TLS connection**.

**Cells**
* Circuit ID (2 bytes)
* Control/Relay (1 byte)
* Data

If the cell is a relay cell and the relay message is intended for the local OR, then when the OR decrypts the message, the OR will get this either a relay message or nonsense. If you've successfully stripped off the last layer of encryption, you can calculate a hash value of the rest of the stuff to check the digest.

**Control cell commands**
* Padding : keepalive, and link padding
* Create : set up new circuit
* netinfo : discover time and their own address
* Destroy : tear down circuit
* vpadding : (variable length) padding
* certs/auth_challenge/authenticate/authorize : used for authentication
 
**Relay cells**
* StreamID - the streamID, since many streams can be multiplexed over a circuit
* Length
* Digest - for integrity
* Command

**Relay cell commands**
* Relay data
* Relay begin
* Relay begin dir - open local stream for directory information
* Relay connected
* Relay extend - to extend the relay by one hop
* Relay sendme - congestion control
* Relay resolve - anonymous DNS

Tor nodes have generic TLS certificates as well as Tor onion keys. When a node connects to an OR, the OR uses a *generic* TLS connection using its certificate, in order to evade detection. Once the TLS connection is established, the Tor nodes use their onion key to authenticate *as a Tor node*.

Multiple circuits may exist over a given TCP connection, and multiple streams may exist over a given circuit. It's possible to specify that different streams should occur in different circuits for anonymization purposes.

### Constructing a Circuit

The client builds a circuit atop the onion routers.

1. Alice contacts Bob, performs TLS and onion key authentication, then they establish a Diffie-Hellman session key. 
1. Alice tells Bob to relay-extend to Carol. Alice sends Bob her half of a second DH session key (encrypted with Carol's public key), and Bob forwards this to Carol (after performing the TLS and key authentication). 
1. Carol sends her half of the DH key back to Bob, who sends it back to Alice. **Carol does not know it was Alice who sent the key**.
1. Alice can now send a relay cell. This cell is encrypted with Alice-Bob's session key. Bob receives it (and knows the state based on its circID), decrypts it, then sees that the next message is destined for Carol (based on the fact that the incoming circuit is associated with the outgoing circuit). He sends the message on to Carol (over TLS). Carol decrypts this message using Alice-Carol's session key (which Carol has saved based on the circID of Bob-Carol). 
1. If Carol makes an HTTP request and wishes to send it back to Alice, she encrypts the message with Alice-Carol's shared key and sends this to Bob (using the circID is the identifying state). Bob encrypts this encrypted message with Alice-Bob's shared key and sends this to Alice. Alice decrypts this one-after-another to find the original message.

Some properties:

1. Only the last OR sees the raw data the client exchanges with the service.
1. Only the last OR knows the IP of the service.
1. The last OR does not know the client's IP address or any of the hops before its prior hop.
1. Only first OR knows the client's IP address.
1. First OR doesn't know IP address of the service.

Clients use measured bandwidth measurements to choose nodes for circuits. They use the **TOR Directory Service** to learn which ORs exist, their details, and their bandwidths (shows LIVE ORs). 

Once an hour, all directory servers exchange their lists of Live ORs and see which of these ORs are on all of the lists. Each server signs a consensus list. When a client downloads an OR list from a directory server, the client checks to ensure that a majority of directory servers signed the list.

### Bridges / Relays
ORs that are not publicly advertised. You learn about them via out-of-band mechanisms.

## Attacks

## Hidden Services
Suppose Bob wants to run a Hidden Service. 

He picks some onion routers to act as "rendevouz points". He signs a statement with his private/public keypair that states that these ORs are his rendevouz points. He distributes this statement out-of-band.

He has his webserver build a Tor circuits to the rendevouz points. In order to connect, Alice builds a circuit to a RP. Then builds a stream over that circuit and encrypts her half of a DH exchange using Bob's public key. Send the HELLO message to the RP. Ask the RP to forward it to Bob's service. Bob sends his half to the RP, who then connects Alice's circuit and Bob's circuit and forwards Bob's message to Alice.


