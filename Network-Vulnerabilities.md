
Let's examine some protocol-level (some implementation-level) vulnerabilites in traditional network protocols such as IP and TCP. 

# Ping of Death

* IP spec: max packet size is 65,535 ($2^16 - 1$) bytes (len field has 16 bits)
* IP packets cross multiple links, each of which has an MTU (Max Transmission Unit): the largest payload that the link frame can handle
* Ethernet's MTU is 1500 bytes, so IP packets will be fragmented into packets of size 1500 bytes.

## IP Packet

* Fragment offset (13 bits wide), in units of 8 bytes.
* Max offset is 65,528 (($2^13 - 1) * 8$)
* This means the max possible lenth of the last fragment is 7 (bytes). Because $max length - max offset = 7$.

The ping of death involved sending a last fragment whose size was > 7, this caused a buffer overflow on the receiving machine. Windows `ping` allowed you to specify arbitrary packet sizes.

# A Look Back at "Security Problems in the TCP/IP Protocol Suite"

## TCP Sequence Number Prediction (Robert Morris)

A TCP connection handshake looks like so:

```
C -> S : SYN(C)
S -> C : SYN(S), ACK(C + 1)
C -> S : ACK(S + 1)
C -> S : C + 1, data
S -> C : ACK(C + len(data) + 1), S + 1
```

For a conversation to take place, the client must first hear `S`, which is essentially a random number. But if these sequence numbers are *wrongfully* used as a security mechanism, and an attacker could predict `S`, it could impersonate a another host.

```
X -> S : SYN(X), SRC = T
S -> T : SYN(S), ACK(X) [This message can be blocked by DoSing T]
X -> S : ACK(S), SRC = T 
X -> S : ACK(S), SRC = T, bad-data
```

The Berkeley implementation of TCP incremented the local sequence number by a constant amount once per second and by half that amount every time a connection is initiated. The attacker can create a legit connection to the server and guess the `S` for the next connection.

The spec says that TCP should increment this 250k times per second. Even if you did that, the attacker could create a legit connection, calculate the RTT, and guess the next number. If you could bound the RTT estimate within 10 milliseconds, there's an uncertainty of 2500 in the possible values for `S`. If each trial takes 5 seconds to allow time to re-measure the round trip time, the attacker could definitely find the next sequence within a day.

The client will send TCP RST packets to the server when it receives the response to the attacker's packet. The attacker can flood the client so it drops packets in its queue.

Really,
1. This number should not be treated as a security property
1. You could use random numbers (and I'm pretty sure that's what TCP does)

**Routers should drop incoming external packets with internal sources and outgoing internal packets with external source addresses.**

# The Kaminsky Attack: DNS

## Terms and Definitions

* **zone** - domain: a collection of hostnames and IP pairs managed together. `google.com` is a zone: `www.google.com`, `test.google.com` are all managed under that zone.
* **Nameserver** - answers the question: What is the IP address for `test.google.com`? Sometimes the NS knows this directly if it's "authoritative" for the zone. Else it has to ask around.
* **Authoritative Nameserver** - the NS that holds the actual information for a zone
* **Resolver** - it's the *client* in this setup: it asks the questions about hostnames. On unix, the file is `/etc/resolv.conf`.
* **Resource record** - key/value pairs stored at nameservers
* **delegation** - when the NS doesn't have the contents of the zone but knows where to find the owner, it delegates service of that zone to another nameserver. This means telling the client where to ask next, as opposed to asking a nameserver itself.

## A simple recursive DNS query

1. Client asks for an A record (IP mapping) for `www.unixwiz.net`. The ISP's nameserver knows it's not authoritative and it hasn't cached this data.
1. All NSes have a set of 13 preconfigured root servers. The NS pick one at random (an actual IP) and forwards this A record query.
1. The root server doesn't know about `unixwiz.net`, the zone, so it sends **us** the address of the `.net` Global Top Level Domain servers. **The root server includes for us "glue", the actual IPs of these GTLDs** so we don't have to go looking for them.
1. The ISP NS chooses a GTLD at random and queries it for the original A level. The GTLD sends us back a referral for `unixwiz.net` including the glue (actual IPs) for these NSes.
1. The ISP NS chooses one and sends the original request. One of these provides us with the authoritative A record and a flag indicating the authoritative response.

* Note that nothing is stopping someone from setting up a NS for any domain to any IP, but a GTLD would never resolve with it.

## A DNS Packet

* Source IP
* Destination IP
* Source Port Number (UDP, same for all queries in a given uptime session)
* Destination Port Number
* Query ID **(This increments by 1 each time we send a new query)**
* QR (0 for query by client, 1 for response from server)
* RD (Recursion Required: the NS our local recursive NS asks may be willing to do the asking for us)
* RA (Recursion Available, 0 or 1)

Note that the final authoritative answer **includes all information such as NS records and glue records.**

## Common Record Types

* **A** - IP Address record
* **NS** - Nameserver record 
* **MX** - Mail server
* **SOA** - Start of Authorities, describes key data about the zone
* **CNAME** - Canonical Name, or alias
* **TXT** - descriptive data about a domain

```
www.example.com. IN A 192.168.1.3
www.example.com. IN A 192.168.7.149
```

## Caching
In reality, recursive NSes cache responses so they don't have to perform these checks. Authoritative answers have the power to set the TTL for a given record. 

But it's not just the A record that's cached. All other authority data, the NS data plus the associated glue A records 

DNS servers only accept responses to *pending queries* that:

* Are on the same port
* Have the same Question section
* Have the same Query ID
* The authority & additional sections contain names that are within the same domain as the question, so `ns.unixwiz.net` can't respond with the IP of `www.unixwiz.net` and also `BankofSteve.com`.

## Attack
The attacker wants to include a mapping from `www.goodsite.com` to his own malicious IP address. In order to do this, he must guess the port, query ID, and eavesdrop on a question. Guessing the port is easy since most NSes use the same port for a given session, and guessing the query ID (when it was simply incremented) is easy as well:

Roles: attacker, victim NS

1. Attacker asks victim NS for IP of `www.bankofsteve.com`. 
1. Victim eventually asks the last GTLD server, which provides a referral to `ns1.bankofsteve.com`.
1. Attacker eavesdrops (or increments a sequence number it got from a previous query to the victim) and finds/guesses the last sequence number the victim used to make this request.
1. Attacker floods victim with responses containing a mapping from `www.bankofsteve.com` to `6.6.6.6` before the authoritative NS can respond.
1. If the QID is correct, win.

Random Query IDs can help here, but they don't protect against eavesdropping.

Kaminsky's attack:
1. Attacker requests `random.bankofsteve.com` via the victim.
1. Victim makes requests to root/GTLD servers.
1. Attacker floods victim with packets guessing the QID, but rather than returning an authoritative answer it returns an NS record with glue pointing to *her own nameserver*, which she can claim is authoritative.
1. Victim will now cache this record and use it for all `bankofsteve.com` requests.

Query IDs are 16 bits (64k values). 

If the attacker can send 50 requests before `random#{random}.bankofsteve.com` returns, the attacker can achieve success within 10 minutes. Attacker can also hijack GTLDs...

## The Fix

Randomize the source port! 

134 million...

# TCP SYN Flooding

## Terms and Definitions
* **Transmission Control Block** - small in-memory buffer server's construct upon receiving a TCP SYN

## The Attack
Attacker can flood the server with SYN packets and force an OOM. The defense: server cookies. We make the server-side stateless until the third packet in the handshake is received. 

```
S = hash(secret, srcIP, srcPORT, destIP, destPORT, C, coarse timestmap)
```
Server only allocates a TCP after receiving the third message and validating that the S ack is correct.

# BGP

## Terms and Definitions

* **Autonomous Systems** - (ASes) set of IP addresses managed by a single admin domain. Each AS can advertise whatever routes they'd like. An AS can advertise 19.1.0.0/16, for example.

## Examples

* In 2008 all YouTube traffic was routed to Pakistan. Pakistan tried to block YouTube by keeping a local route to a non-site, but instead they advertised a route for 208.65.153.0/24 to an AS in Hong Kong, who forwarded it on. YouTube's prefix is 208.65.152.0/22
