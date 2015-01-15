---
layout: page
permalink: /about/
---

**Akumuli** is a time-series database. The word "akumuli" can be translated from esperanto as "accumulate".

###Rationale

Most open source projects focus on query language and things useful for web-analytics, but they ignore some key characteristics of time series data:

* High write throughput (millions of data-points per second)
* Very late writes can be dropped
* Numeric time-series can be compressed very efficiently
* Periodic time-series can be compressed very efficiently
* Compression is crucial for time-series storage!

###Features

* Implements specialized storage engine for time-series data
* Memory mapped and x64 only
* Uses constant amount of disk space (like RRD-tool)
* Crash recovery 
* Very high write throughput (about 1M writes per second on single machine)
* Allows unordered writes
* Compressed (specialized compression algorithms for different data elements - timestamps, ids, values)
* Easy to use server software (based on Redis protocol)
 
###Documentation

* [Wiki](https://github.com/akumuli/Akumuli/wiki)
* [Getting started](https://github.com/akumuli/Akumuli/wiki/Getting-started)
* [API reference](https://github.com/akumuli/Akumuli/wiki/API-Reference)

[![Build Status](https://api.shippable.com/projects/5481624dd46935d5fbbf6b58/badge?branchName=master)](https://app.shippable.com/projects/5481624dd46935d5fbbf6b58/builds/latest)
