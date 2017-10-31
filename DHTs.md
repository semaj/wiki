# Distributed Hash Tables

## Terms and Definitions

# Pastry
Each node in the network has a unique numeric identifier called a nodeId. When presented with a message and a numeric key, a Pastry node efficiently routes the message to the node with a nodeID that is numerically closest to the key among all currently live Pastry nodes.

It takes *network locality* into account: it seeks to minimize the distance messages travel according to a metric like IP routing hops.

## Use Cases
PAST is a file system which uses the hash of a file as the fileId and the file is stored on the *k* pastry nodes with nodeIDs numerically closest to this ID. A lookup is guaranteed to reach a node that stores the file as long as one of the *k* nodes is live.

## Overview
Each Pastry node keeps track of:
* Its immediate neighbors in the ID space
* Notifies applications of new node arrivals, failures, and recoveries

NodeIds are generated in a 128-bit space from the hash of the IP address or public key.

Pastry has the following guarantees:
* Route to the numerically closest node in less than $\lceil{log_{2^b}N}$ steps (*b* is a configuration parameter, typical value 4)
* Eventual delivery is guaranteed unless $\floor{|L|/2}$ nodes with adjacent nodeIDs fail simultaneously. *|L|* is usually 16 or 32. 
