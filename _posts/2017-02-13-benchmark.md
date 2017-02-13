---
layout:     post
title:      Understanding Akumuli Performance
date:       2017-02-13 18:00:00
summary: Akumuli is designed to be a blazingly fast database with a small footprint on the system. I'm taking performance really seriously and will consider any performance drop between versions as a bug. But how fast is it actually?
categories: akumuli
---

Akumuli is designed to be a blazingly fast database with a small footprint on the system. I'm taking performance really seriously and will consider any performance drop between versions as a bug. But how fast is it actually?

To answer this question I've benchmarked Akumuli on AWS and it managed to sustain 4.5 million write operations per second on a dedicated m3.2xlarge node. This isn't a peak but steady write throughput. And my tests showed that it scales almost linearly on a multicore machine.

![Akumuli benchmark results on AWS m3.2xlarge instance](/images/benchmark_bar_plot.png)

### Methodology

Akumuli has a configuration option that called `TCP.pool_size`. It controls the server side parallelism. By default, this option is equal to one. But it can be increased to take an advantage when running on the multicore machine. I ran the benchmark with different numbers on the eight core machine and here are the results:

| Number of threads | Throughput datapoints/sec |
|-------------------|---------------------------|
|1                      | 693 984|
|4                      | 2 433 831|
|8                      | 4 547 421|

The input is constructed using [this script](https://github.com/akumuli/test_input_generator). All test data was generated beforehand and resides in eight archives. Each archive has 86401000 data elements. All eight archives had around 691M data elements (3.5G compressed).
I started Akumuli with different `pool_size` values and measured the time it took to write all the data. After each run, the resulting database size was 2.7GB (4.2 bytes per element).

Single `m3.xlarge` instance has been used to generate load. I started 8 writer processes, each with its own set of series.

### What makes Akumuli so fast?

Storage in Akumuli is based on append-only B+tree. Each series is represented using a separate B+tree instance and can be modified independently, without any synchronization. The only moment when synchronization is needed is when individual pages get committed to disk because all trees reside in the same file.

![B+tree mapping](/images/btree_schema.png)

The write-path code is based on the observation that different TCP-sessions usually write data to the different time-series. When session writes something to the series first time the corresponding B+tree that stores series data get assigned to that session. After that, the session will be able to write data to this series without any synchronization. If B+tree is already owned by another thread, additional synchronization will be needed. 

# Final notes

Using a single-node instance of Akumuli running on the m3.2xlarge instance I managed to write 4.5M data points / second. This is not a high-end machine by any means. I'm looking forward to running this test on more performant hardware and see the results. That would be very interesting to try.
