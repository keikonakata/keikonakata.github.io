# Google File System

Fault-tolerant scalable distributed file system for large data-intensive applications

Delivers high performance to a large number of clients

Design assumptions

- File access:
  - Most files are appended, not overwritten
    - Random writes within a file are almost never done
    - Once created, files are mostly read, often sequentially
  - Workload is mostly
    - Large streaming reads, small random reads
    - Large appends
    - Hundreds of processes may append to a file concurrently
- GFS stores a few million of files
- Assume apps can handle a relaxed consistency model

1. Use separate servers to store metadata
   - Metadata includes lists of (server, block_number) sets that identify which blocks on which servers hold file data
   - More bandwidth for data access necessary than metadata access
     - Metadata is small; file data can be huge
1. Use large logical blocks
   - In comparison to "normal" file systems, whose typical block size is 4KB, optimized for small files
1. Besides basic operations (such as create, open, write),
   - Snapshot, to create a copy of a file or directory tree at low cost
   - Append, to  allow multiple clients to append atomically without locking

GFS servers are implemented in user space using native linux FS

GFS cluster consists of
- Multiple chunkservers with fixed-size (default, 64MB) chunks.
  32-bit checksome with each chunk to detect data corruption.
  Chunkservers store chunks on local disk as Linux files.
  Chunks replicated on several servers (typically three replicas).
- One master (for 1000s of chunkservers),
  - stores file system metadata, such as access control info, mappings from files to chunks, and current locations of chunks.
    (Master does not store chunk locations persistently, but queries chunk servers.)
  - manages leases, frees unused chunks and copy/move chunks
  - All operations on master are logged and persisted on the disk, with periodic checkpoints stored in a B-tree.
  - Master assigns a globally unique 64-bit number to each chunk, when it is created.

GFS clients interact with master for metadata-related operations.
They interacts directly with chunkservers for file data. (All reads and writes go directly to chunkservers.)

###### References

https://www.cs.rutgers.edu/~pxk/417/notes/pdf/12-dfs-slides.pdf
(accessed on 6.04.2021)

# Google Chubby

Distributed lock service + simple fault-tolerant file system

Interfaces: file access, event notification, file locking

Used for
- coarse-grained long-term locks (hours or days, not < sec)
- store small amount of data associated with a name such as system configuration
- elect masters

Design priority: availability rather than performance

Chubby has at most one master, elected by a consensus protocol (paxos).

Requests from the client go to the master

Master gets a lease time, and re-run master selection after lease time expires of if the master fails.

When a Chubby node receives a proposal for a new master, it only accepts it if the old master's lease has expired

The client access Chubby via an API. Look up Chubby nodes via DNS. Ask any Chubby node for the master. File system interface.

Each file has name, data, access control list, and lock. (No modification/access time, nor partial reads/writes/ nor symbolic links nor moves)

Every file and directory can act as a reader-writer lock.
If a client fails, the lock will be unavailable for a lock-delay period (typically 1 minute.)

Chubby locks for leader election and using it to write to a file server.
- Participant who gets a lock together with a lock sequence count is the master.
- In each RPC to the server, send the sequence count.
- During request processing, the server reject packegs with old sequence count.

Chubby client cashing & master replication

At the client
- Data cached in memory by chubby clients. Cache is maintained by a Chubby lease, which can be invalidated
- All clients write through to the Chubby master

At the master
- Writes are propagated via Paxos to all Chubby replicas
  - Data updated in total order. Replicas remain synchronized
  - The master replies to clients after the consensus is reached
- Cache invalidations
  - Master keeps the list of what each client may be caching
  - Invalidations sent by the master and are acknowledged by clients
  - File is then cacheable again
- Chubby database is backed up to GFS every few hours

https://www.cs.rutgers.edu/~pxk/417/notes/pdf/12-dfs-slides.pdf
(accessed on 6.04.2021)

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
