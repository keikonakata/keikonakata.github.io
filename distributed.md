# Fischer-Lynch-Paterson Impossibility result

The paper proves that any consensus protocol that tolarates one process failure
under the reliable asyhchronous message system, in which all messages are eventually delivered with arbitrary delay, fails to reach consensus under some circumstances.

The paper proves the statement (quite elegantly) by showing, for any given protocol, circumstances in which the protocol remains forever indecisive.

## Details

Assume that the message system is reliable -- it delivers all messages correctly and exactly once.

An atomic broadcast capability is assumed, so a process can send the same message in one step to all other processes.

Crucial to the proof: Processing is completely asynchronous: no assumption about the relative speeds of processes or the delay time in delivering messages.
Processes do not have access to synchronized clocks, hence algorithms based on time-out cannot be used.
Processes cannot detect the death of a process, they cannot tell if another has died or is running very slowly.

Every process starts with an initial value in {0, 1}.
A process decides on a value in {0, 1} by entering an appropriate decision state.
All processes that makes a decision are required to choose the same value.
Some process must eventually make a decision.
Both 0 and 1 must be possible decision values (perhaps for different initial configurations.)
The last requirement is to eliminate the trivial solution, in which, say, 0 is always chosen.

A configuration reachable from some initial state is said to be accessible.

A consensus protocol is partially correct if

1) No accessible configuration has more than one decision value.
2) For each v in {0, 1}, some accessible configuration has decision value v.

A run is admissible if at most one process is faulty and all messages sent to nonfaulty processes are eventually received.

A run is deciding if some process reaches a decision state.

Main result: 

Proof:

[https://dl.acm.org/doi/10.1145/3149.214121](https://dl.acm.org/doi/10.1145/3149.214121)

# Concurrency Control

Optimistic concurrency control with timestamps``
Each object has two timestamps
Read timestamp is updated when the object is read
Write timestamp is updated when the object is written

Each transaction has a timestamp = start of transaction

If a transaction wants to write, the object’s read and write timestamps must be older
If a transaction wants to read, the object’s write timestamp must be older

# Distributed commit protocols

Properties of transactions: Atomic, Consistent, Isolated, Durable

distributed transaction need unanimous agreement to commit
- all participants agree => all commit, otherwise all abort

## Two phase commit protocol
one coordinator
everyone has write-ahead log in stable stroge
no system dies forever (i.e., fail-recover model)
messages always arrive
Phase 1
1. coordinator write "prepare to commit" to log
2. coodinator send "can commit?" message to participants
3. participants receive "can commit?" message, when ready write "agree to commit" or "abort" to the log. Send either to the coodinator
Phase 2
1. after receiving messages from all the participants, write "commit" or "abort" to the log and send message to participants
2. Participants receive "commit" or  "abort". Write to the log, and commit or abort. Then send "done" to the coodinator.
3. Cordinator receives "done" from all participants and done.

Two phase commit is a blocking protocol. (Failure modes that require all systems to recover eventually.)

## Three phase commit protocol
Add timeouts to each phase that result in abort
Split the 2nd phase of 2PC into two steps:
1a. coordinator send "prepare" to all participants when it receives yes fom all participants in phase 1
1b. participants reply with "ack" but don't commit yet
2a. If coordinator gets ACK from all, it send "commit" to all, otherwise it abort and send "abort" to all

3PC is not resilient against network partions or fail-recover environment.

Consensus-based commit
Run a consensus algorithm on the commit/abort decision of each participant

Eventual consistency
If no updates are made to a data item, eventually all accesses to that item will return the last updated value.

# Distributed Hash Tables

Earlier ideas on distributed look-ups

- central coordinator
- query flooding
  - requests contain Time-To-Live
- DNS uses hierarchical lookups

## distributed hash tables

Algorithmic requirement: every node can find the answer

Trade-off between state, maintenance traffic and #-lookups

Chord

consistent hashing based on a logical circular space

#-keys/#-buckets need to be remapped when a node arrives/leaves

To create N replicas, store each key-value at N-1 successor nodes

Have all nodes know about each other O(1) look up vs finger tables O(log N) look up
  finger tables - i-th entry in finger table identify the first node that succeeds or equals to n+2^i

To have good load balance, represent each bucket by log(N) virtual buckets. (Why log(N)?)

Data replicated on N hosts. Key is assigned to a coordinator node (via hashing) who is in charge of replication.

a node is assigned random values in the hash space and to multiple points in the ring (virtual node)

virtual nodes help balanced load distribution

Routing table size: Log N, Routing time: O(log N) hops, because each hop expects to half the distance

Other topologies than finger tables:

- tree-like structures (Pastry, Tapestry, Kademlia)

- Hypercubes (CAN)
  Maintains pointers to a neighbour who differs in one bit position. Only one possible neighbour in each direction. Route to reciever by changing any bit.

The ring geometry allows the greatest flexibility, and hence archives the best resilience and proximity performance. (Cf. [The impact of DHT routing geometry on resilience and proximity](https://dl.acm.org/doi/10.1145/863955.863998))

#-neighbours: Hypercube 1, Chord&tree: 2^i (what are neighbours about?)
#-forwarding: tree 1, hypercube N/2, ring Log N

Many services (like Google) are scaling to huge #s without DHT like log(N) techniques, but with direct routing where everyone knows full routing tables.
One-hop routing with sqrt(N) state instead of log(N) state.

### Amazon Dynamo

- incremental scalability, decentralization, heterogenecity (mix of slow and fast systems)

- always writable data store (even during network partitions)

- use weaker consistency

- may have conflicting changes that have to be resolved during reads by applications

- Not all updates may arrive at all replicas. Data is versioned, using vector clocks.

  If a node was unreachable, the replica is sent to another node in the ring.
  Metadata about the original desired destination is sent with the data.
  Periodically, a node checks if the original targeted node is alive.

- use a logical ring

# References

https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-852j-distributed-algorithms-fall-2009/

http://www.cs.cmu.edu/~dga/15-744/S07/lectures/16-dht.pdf

https://www.cs.rutgers.edu/~pxk/417/index.html


https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-033-computer-system-engineering-spring-2018/