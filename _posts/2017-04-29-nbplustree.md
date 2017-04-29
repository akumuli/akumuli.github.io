---
layout:     post
title:      Storage engine
date:       2017-04-29 18:00:00
summary: In time-series databases the querying pattern is different from write pattern. We usually write data in time order by updating many series every second. But querying is a different story. We usually want to read only a handful of series leaving most of the data ...
categories: akumuli
---

In time-series databases the querying pattern is different from write pattern. We usually write data in time order by updating many series every second. But querying is a different story. We usually want to read only a handful of series leaving most of the data behind. Here lies the biggest problem, we don’t really know how data will be read and can’t put the data that will be read together close on disk. But at least we can put each series to the separate file. This is what Graphite does and it somewhat works (with a fair amount of in batching and in-memory caching).

Some other TSDBs are using LSM-tree or similar structures. In this databases the LSM-tree partitions the key space in time dimension and the data points are stored in chunks in some sort of column oriented format that allows to locate individual series quickly. Each chunk stores data points from many series (not necessary from all of them). This design lead to high read amplification, because it’s impossible to read only the data we need without reading and decompressing all the other data in the chunk. Another problem is that data in the chunk is not aligned. This is fundamental limitation, the data of the individual series in the chunk can’t be properly aligned by block boundary because everything is partitioned by time.

Akumuli’s goal is to maintain separate disk backed data structure for each series in the database to minimize read amplification. This is not feasible with LSM-trees for many reasons:

- LSM tree needs a lot of RAM for memory resident part to be able to batch up many values into one large write operation.
- High write amplification. Each data element gets written to disk several times. When dealing with time-series data this seemed to be unnecessary because most of the data is never changed.
- Each SSTable is stored in its own file. Because of this the database needs to open a lot of files simultaneously. Storing each time-series in it’s own LSM-tree is not feasible in this case. We will need a handful of files for each time-series and we might want to have millions of them.
- Merging SSTables is expensive. We need to read two or more files and write another one. If we want to store each time-series in its own tree, we will need to perform this operation really frequently.
- This design was created for spinning disks, not for modern era SSD and NVMe drives. Random wires is not that slow anymore. 

This is the reason for the LSM-tree based design described above.

### Meet the Numeric B+tree

Akumuli is based on novel data-structure called “Numeric B+tree”. It’s is somewhat close to LSM tree but have some nice properties that the later doesn’t have. BTW, it’s called “numeric” only because it’s supposed to store numeric data (timestamps and values). The data structure itself can be described as B+LSM-tree. Think about LSM-tree but with B+trees instead of SSTables.

Akumuli uses timestamps as keys and keys should be inserted in increasing order (backfill can be implemented but this goes far beyond the subject of this article). This means that timestamps in each B+tree doesn’t overlap and that we can build full trees incrementally, without node splitting (TBD: the link!).

Each NB+tree instance is a series of extents, each extent is a B+tree instance.

- The first extent is a single leaf node of the B+tree stored in memory;
- The second extent is a B+tree of height 2 (inner node that stores up to 32 references to leaf nodes);
- The third extent is a B+tree of height 3 (inner node that stores up to 32 references to inner nodes that stores references to leaf nodes);
- Etc.

Each extent can be seen as a single inner node that stores references to extents of the previous level (e.g. level 3 extent is a single inner node that stores references to level 2 extents, level 2 extent is a single inner node that stores references to level 1 extents/leaf nodes). This property allows us to build extents easily.

![Fig 1](/images/NBtreeExtents_no_shadow.png)

The picture shows B+trees with fan-out ratio of 4 but real B+tree used by Akumuli have fan-out ratio of 32. Note that each B+tree can be incomplete only from the right.

#### Tree construction

The tree construction algorithm is simple:

- Write new values to the level-1 extent (memory resident leaf node);
- If level-1 extent is full - write it to disk and add reference to it to the level-2 extent;
- If level-2 extent is full - write its root node to disk and add reference to it to the level-3 extent;
- Etc.

This process is an equivalent of the SSTable merge in LSM-tree. Number of extents is expected to be small because each leaf node contains many values (it contains variable number of values because of compression) and fan-out ratio is 32. Let’s do some back of the envelope calcuations! (TBD - calculation - 1block - 1000avg, lvl2 update - 32000avg, lvl3 update - 1024000, lvl4 - 32768000, lvl5 - 1048576000, lvl7 - almost 32 years with millisecond step, lvl10(max) - more than 300 years with microsecond (us) precision). 

Here comes the advantages of the NB+tree over the LSM-tree:

- Memory resident part of the tree is small (single leaf node + several inner nodes);
- Many trees can live in one file and blocks of different trees can be interleaved on the storage level;
- Merge is very efficient. It’s enough to just add an address of the root node of the one extent to the other extent; No need to read the data and actually merge anything.
- We can store some information inside inner nodes to speed up aggregation and resampling of the time-series without maintaining some external data-structures (rollups) or indexes.
- Parallel operation. It is possible to write to the database from many writer threads (and Akumuli is doing this).
- Good for modern SSDs. All writes are aligned and performed in parallel.

This data-structure allows Akumuli to maintain separate NB+tree instance for every time-series in the database. The main disadvantage of this design is a complexity of the crash recovery process. It’s impossible to maintain WAL or command log per NB+tree instance. Akumuli uses different approach to crash recovery that’s not discussed in this article.

Ability to build separate NB+tree instance for every time-series is truly important. It enables queries in column oriented fashion and, at the same time, it enables really fast parallel writes with very small write amplification. My recent experiment showed that it can write about 16 million data points per second on c3.8xlarge instance. 
