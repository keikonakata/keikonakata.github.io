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
  - Master periodically communicate with all chunkservers via heartbeat messages

GFS clients interact with master for metadata-related operations.
They interacts directly with chunkservers for file data. (All reads and writes go directly to chunkservers.)

Large chunk size, default 64MB, in comparison to Linux ext4 block sizes, typically 4KB and up to 1MB.
Large chunk makes it feasible to keep a TCP connection open to a chunkserver for an extended time. (Why?)

Reading files: contact the master, get the metadata, including the locations of chunks, contact any available chunkserver.

Writing
- Master grants a chunk lease to one of the replicas. This replica will be the primary chunk server, which can request lease extensions if needed. Master increases the chunk version number and informs replicas.

Write in two phases.
- Phase 1: A client is given a list of replicas and it identifies the primary and secondaries. The client writes to the closest replica chunkserver, which further forwards the data to anothe replica chunkserver,.... Maximize bandwidth via pipelining.
Chunkservers store the received data in a cache.
Order does not matter at the phase 1.

- Phase 2: The client waits for replicas to acknowledge receiving the data, then sends a write request to the primary, identifying the data that was sent.
The primary is responsible for serialisation of writes. It assigns consecutive serial numbers to all write it has received. Applies write in the serial number order and forwards write requests in order to secondaries.
Once all acknowledgements have been received, the primary acknowledges the client.
At the phase 2, locking is used and order is maintained.

Chunk version numbers are used to detect if any replica has stale data (was not updated because it was down.)

GFS has no per-directory data structure like most file systems.
Namespace is a single lookup table, mapping pathnames to metadata.


###### References

https://www.cs.rutgers.edu/~pxk/417/notes/pdf/12-dfs-slides.pdf
(accessed on 6.04.2021)

