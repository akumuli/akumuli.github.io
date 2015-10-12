---
layout:     post
title:      Why Akumuli is a standalone database?
date:       2015-10-12 19:00:00
summary: 
categories: akumuli
---
There is a lot of distributed time-series databases nowadays but Akumuli takes a different approach. It’s a standalone solution and there is some logic behind this design decision.

Time-series data storage is actually quite simple problem. It was solved many times by many different companies. And the reason why we don’t have a lot of distributed TSDB’s yet is actually very simple - most of us simply don’t need it. I’m not speaking here about monitoring startups - maybe they do need scaleable TSDB’s, but most projects simply don’t generate enough data. 

Several years ago I was heavily involved in development of the SCADA system that handles data from entire russian electric grid. Time-series storage component of this system works on a single machine (with hot reserve). Difficulties were not in time-series storage throughput, actual time-series storage system were backed by relational database (because of good tooling and convenience).

Let’s do some “back of the envelope” calculations. Imagine that we have 1000 servers, each sends 100 data-points per second (CPU time, memory usage, etc). This is only 100K data-points per second. This number is not big, it’s actually quite small. If writes are batched and data is compressed (4 bytes per data-point) only about 390KB/s of write bandwidth will be used. 1M data-points per second is just about 3.8MB/s of write bandwidth. And modern SSD’s has sequential write throughput measured in hundreds of megabytes per second. The write path is not even I/O bound so system that can write hundreds of millions of data-points per second on single machine is feasible (but hard to build, one should parallelize write path to do this).

Modern servers are the number crunching beasts with large number of CPU’s, fast memory, storage and network. There is a good chance that it is possible to handle all your monitoring workload using only one machine with right software.

###Things that some TSDB vendors didn’t get.

It seemed that many TSDB vendors/users didn’t understand this simple things:

1. Modern hardware is capable to handle time-series data at large scale. Time-series data is sequential and hardware don’t need to bother with random writes and reads. Prefetching works best, all hardware resources (disk/RAM write throughput) can be utilized 100%.
2. There is no such thing as one time-series data type! There is many different time-series types, irregular (or raw) time-series, regular (irregular time-series after PAA transform), SAXifyed time-series data, time-series data in frequency domain etc. 
3. You can’t use the same storage mechanisms for all this data types. Best fit for raw time-series data is an append-only log-structured storage. Best fit for SAX-transformed time-series is an inverted index with compression (at least in some cases). Best fit for PAA-transformed time-series is memory-resident key-value store, etc. Why nobody focuses on this?
4. Auto-scaling is not that necessary at all, because actual write load is usually very tiny (see #1).
5. What’s really needed is replication (for fault tolerance).

In many cases you even doesn’t need to store original time-series data. Stream processing is fine for most monitoring tasks (thresholds, anomaly detection, alarms etc). You don’t need raw time-series to draw graph or to see that value is out of range for too long. Raw time-series data usually needed to perform data-analysis, calculate DTW distance measure, SAXify etc. But most of this tasks are fast enough for single machine anyway (except kNN, kNN requires a lot hardware resources)
