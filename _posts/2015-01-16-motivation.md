---
layout:     post
title:      Motivation
date:       2015-01-16 19:03:29
summary: I don't like most opensource time-series databases that have been created recently, some of them lack compression, some of them is slow or focus on the wrong problems. In my opinion good time-series database should satisfy several requirements.
categories: akumuli
---
I don't like most opensource time-series databases that have been created recently. Some of them lack compression, some of them are slow or focus on the wrong problems.
In my opinion good time-series database should satisfy several requirements:

* Good compression. It shouldn't be possible to disable compression. It should be used by both, storage engine and network protocol.
* Good ingestion rate. Time-series data is often created at high pace and writing it to a database can be a real challenge. I guess that TSDB should be able to achieve sustained write rate above one million data points per second. If TSDB can't do this it can be replaced with convenient relational database with little or no effort.
* It should use a fixed amount of disk space like [RRD-tool](http://oss.oetiker.ch/rrdtool/). This is a very good property that allows TSDB to operate without danger of data loss.
* Simplicity. No need for complex features like continuous queries in influxdb or complex query languages. Just simple retrieval by time range and series.
* No complex dependencies. One should be able to run TSDB on a single machine like Redis. Just run and start to collect data from different sources. One executable, one configuration point.

You can argue that a system like this is not very useful and a complex query language or other features should be implemented. Well, this is a very basic requirements to start from. 
I'm planning one advanced feature that absent from all time-series databases that I know — indexing. 
Not the same indexing as used in normal databases but specific for numeric time-series that can speed up similarity search on numeric time-series data. 
Such indexing opens paths for things like time-series clustering, motif discovery, summarization and anomaly detection.
I think that having the ability to quickly perform similarity search is infinitely better than the ability to execute continuous queries or draw graphs. 
