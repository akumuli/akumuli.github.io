---
layout:     post
title:      Akumuli Markedly Outperforms InfluxDB in Time-Series Data & Metrics Benchmark
date:       2017-01-24 12:00:00
summary: Usually, I'm trying to avoid comparisons with other databases so nobody can blame me biased. Folks at InfluxData don't share this judgment so they created the comprehensive test suite and published several articles comparing their product with the competition. This test suite is available on GitHub. For me, this looks like an invitation so I decided to use it to compare Akumuli with InfluxDB.
categories: akumuli
---

Usually, I'm trying to avoid comparisons with other databases so nobody can blame me biased. Folks at InfluxData don't share this judgment so they created the comprehensive test suite and published several articles comparing their product with the competition. This test suite is available on GitHub. For me, this looks like an invitation so I decided to use it to compare Akumuli with InfluxDB (v1.1.1). I forked this repo and added [Akumuli](https://github.com/akumuli/Akumuli) support. It's available [here](https://github.com/Lazin/influxdb-comparisons). Now I want to share the results and some thoughts about it with you.

There is a bunch of applications in this test suite (in /cmd dir). There is an app that generates input (one for all databases, you should tell it what database are you working with), the app that generates queries, the bunch of apps that used to load data (one per supported database), and another set of apps that can be used to benchmark each database (one per database as well) using generated queries. 

An app that generates input is called `bulk_data_gen`. It can produce `DevOps` or `IoT` workload (the later haven't been implemented yet, I think). The DevOps workload generator can produce input that simulates real-life DevOps workload. It contains metrics describing CPU, memory, Redis, Postgres and stuff like that. Each data point has a lot of tags and fields (up to 30+ fields). This is the first data point in InfluxDB format:

```
cpu,hostname=host_0,region=eu-west-1,datacenter=eu-west-1b,rack=67,os=Ubuntu16.10,arch=x86,team=NYC,service=7,service_version=0,service_environment=production usage_user=58.1317132304976170,usage_system=2.6224297271376256,usage_idle=24.9969495069947882,usage_nice=61.5854484633778867,usage_iowait=22.9481393231639395,usage_irq=63.6499207106198313,usage_softirq=6.4098777048301052,usage_steal=44.8799140503027445,usage_guest=80.5028770761136201,usage_guest_nice=38.2431182911542820 1451606400000000000
```

The table name is `cpu`. There is 10 tags - `hostname`, `region`, and so on, and also, there are 10 fields - `usage_user`, `usage_system`, etc.

What's the problem with this? Many Time-Series databases don't support fields (e.g. Graphite, OpenTSDB). With this databases, one should simulate fields using some series naming convention. E.g. for OpenTSDB `bulk_data_gen` app generates series names that contain table name and field name (cpu.usage_user, cpu.usage_nice, etc). This results in terrible protocol bloat, because each data point now should have entire set of tags and timestamp. The same data point in OpenTSDB protocol will look like this:

```json
{"metric":"cpu.usage_user","timestamp":1451606400000,"tags":{"arch":"x86","datacenter":"eu-west-1b","hostname":"host_0","os":"Ubuntu16.10","rack":"67","region":"eu-west-1","service":"7","service_environment":"production","service_version":"0","team":"NYC"},"value":58.13171323049762}
{"metric":"cpu.usage_system","timestamp":1451606400000,"tags":{"arch":"x86","datacenter":"eu-west-1b","hostname":"host_0","os":"Ubuntu16.10","rack":"67","region":"eu-west-1","service":"7","service_environment":"production","service_version":"0","team":"NYC"},"value":2.6224297271376256}
{"metric":"cpu.usage_idle","timestamp":1451606400000,"tags":{"arch":"x86","datacenter":"eu-west-1b","hostname":"host_0","os":"Ubuntu16.10","rack":"67","region":"eu-west-1","service":"7","service_environment":"production","service_version":"0","team":"NYC"},"value":24.99694950699479}
{"metric":"cpu.usage_nice","timestamp":1451606400000,"tags":{"arch":"x86","datacenter":"eu-west-1b","hostname":"host_0","os":"Ubuntu16.10","rack":"67","region":"eu-west-1","service":"7","service_environment":"production","service_version":"0","team":"NYC"},"value":61.58544846337789}
{"metric":"cpu.usage_iowait","timestamp":1451606400000,"tags":{"arch":"x86","datacenter":"eu-west-1b","hostname":"host_0","os":"Ubuntu16.10","rack":"67","region":"eu-west-1","service":"7","service_environment":"production","service_version":"0","team":"NYC"},"value":22.94813932316394}
{"metric":"cpu.usage_irq","timestamp":1451606400000,"tags":{"arch":"x86","datacenter":"eu-west-1b","hostname":"host_0","os":"Ubuntu16.10","rack":"67","region":"eu-west-1","service":"7","service_environment":"production","service_version":"0","team":"NYC"},"value":63.64992071061983}
{"metric":"cpu.usage_softirq","timestamp":1451606400000,"tags":{"arch":"x86","datacenter":"eu-west-1b","hostname":"host_0","os":"Ubuntu16.10","rack":"67","region":"eu-west-1","service":"7","service_environment":"production","service_version":"0","team":"NYC"},"value":6.409877704830105}
{"metric":"cpu.usage_steal","timestamp":1451606400000,"tags":{"arch":"x86","datacenter":"eu-west-1b","hostname":"host_0","os":"Ubuntu16.10","rack":"67","region":"eu-west-1","service":"7","service_environment":"production","service_version":"0","team":"NYC"},"value":44.879914050302744}
{"metric":"cpu.usage_guest","timestamp":1451606400000,"tags":{"arch":"x86","datacenter":"eu-west-1b","hostname":"host_0","os":"Ubuntu16.10","rack":"67","region":"eu-west-1","service":"7","service_environment":"production","service_version":"0","team":"NYC"},"value":80.50287707611362}
{"metric":"cpu.usage_guest_nice","timestamp":1451606400000,"tags":{"arch":"x86","datacenter":"eu-west-1b","hostname":"host_0","os":"Ubuntu16.10","rack":"67","region":"eu-west-1","service":"7","service_environment":"production","service_version":"0","team":"NYC"},"value":38.24311829115428}
```

That's a lot of text! And the `cpu` table is not the largest one! There is a table with 30+ fields and a huge set of tags. That's why ingestion in OpenTSDB is slow (5x slower then InfluxDB) because database needs to parse input that large, no surprises here. We can see this in [the article](https://www.influxdata.com/influxdb-markedly-outperforms-opentsdb-in-time-series-data-metrics-benchmark/). But if your data don't have all this fields figures will be different. I think that OpenTSDB wouldn't look that bad.

But let's look how Akumuli performs here. Akumuli doesn't support fields and tables (there are only series names) but it has bulk load protocol that can encode many data points that share timestamp and tags using only one message. Because of that input size for Akumuli is about the same size as InfluxDB input (~1GB). I have run the tests on my machine and on two m4.xlarge AWS instances.

### Write Performance

I've run the tests locally at first, here is [the script](https://github.com/Lazin/influxdb-comparisons/blob/master/cmd/load_data.sh) output:

    $ ./load_data.sh 
    Loading data into Akumuli
    daemon URLs: [127.0.0.1:8282]
    loaded 19284480 items in 151.588062sec with 1 workers (mean rate 127216.350267/sec)
    Loading data into InfluxDB
    daemon URLs: [http://localhost:8086]
    [worker 0] backoffs took a total of 0.000000sec of runtime
    loaded 19284480 items in 423.174608sec with 1 workers (mean rate 45570.976222/sec, 19.99MB/sec from stdin)


|                |     Run time|  Write throughput |
|---|---|---|
|Akumuli  |  151.6sec|     1427650 points/sec|
|InfluxDB|    423sec  |     511407 points/sec|

In the case of Akumuli ingestion speed was limited by the gzip decompression speed because the CPU core that has been running gzip was 100% busy. Actual disk write throughput was steady (iotop showed around 6M/s). InfluxDB has been writing to disk around 3-6 M/s with occasional bumps up to 13M/s and it was 33M/s one time. I guess this is due to compaction process.

I've used only one worker thread with both databases because I wasn't able to replicate the real workload the other way. In real world, each connection will write to the unique subset of series. Akumuli is optimized for this case, it scales linearly with CPU cores and it actually uses many write threads to write data to disk. If this property doesn't hold, this writer threads will need to synchronize access and this will limit throughput. Loaders from this test suite split messages amongst workers in round-robin manner but they should split them by hostname. That's why I decided to use a single worker.

### AWS

I tried to run this test on two m4.xlarge instances, one for database and another one to generate load. Results is here:


    ubuntu@ip-172-31-41-230:~/work/src/influxdb-comparisons/cmd$ ./load_data.sh 
    Loading data into Akumuli
    daemon URLs: [172.31.36.43:8282]
    loaded 19284480 items in 249.247543sec with 1 workers (mean rate 77370.792655/sec)
    Loading data into InfluxDB
    daemon URLs: [http://172.31.36.43:8086]
    [worker 0] backoffs took a total of 0.000000sec of runtime
    loaded 19284480 items in 614.613496sec with 1 workers (mean rate 31376.597036/sec, 13.76MB/sec from stdin)

As you can see, Akumuli is still faster. The throughput of both databases is smaller due to EBS volume being used instead of SSD.

### On Disk Compression

InfluxDB did a great job here. As you can see it uses twice less disk space then Akumuli. It compresses this data almost as good as gzip does. This difference is due to compression algorithms being used, Akumuli uses a faster algorithm that avoids bit packing and InfluxDB uses a slower algorithm that uses bit packing and has a better compression ratio as result.

    705M    /home/elazin/.akumuli/
    323M    /var/lib/influxdb/

BTW, I don't really believe in x16.5 less disk space claim. I've been using HBase for time-series for some time and I can say that with proper data block encoding and enabled compression it gives decent results.

### Query Performance

I have run query benchmarks only locally. Before running each benchmark I [cleaned](https://github.com/Lazin/influxdb-comparisons/blob/master/cmd/clear_caches.sh) file caches and then started the databases. The results are in the table below.

#### Akumuli Results

|     | 1 host, 12h by 1min | 1 host, 1h by 1min | 8 hosts, 1h by 1min | 1 host, 12h by 1h |
|-----|---------------------|--------------------|---------------------|-------------------|
| min | 1.59ms              | 0.70ms             | 1.93ms              | 0.77ms            |
| avg | 1.93ms | 0.82ms | 2.22ms | 0.88ms |
| max | 3.41ms | 1.67ms | 2.82ms | 3.59ms |

Here `1 host, 1h by 1min` means that query reads 1 hour interval with 1 minute step for 1 host.

#### InfluxDB Results

|     | 1 host, 12h by 1min | 1 host, 1h by 1min | 8 hosts, 1h by 1min | 1 host, 12h by 1h |
|-----|---------------------|--------------------|---------------------|-------------------|
| min | 2.96ms | 0.62ms | 1.50ms | 0.78ms |
| avg | 4.30ms | 0.87ms | 2.51ms | 1.24ms |
| max | 11.02ms | 5.62ms | 9.89ms | 6.26ms |

As you can see, average performance is about the same (but Akumuli is just slightly faster). Both databases are column oriented, no surprises here. But max latency is quite different. I think that this is due to GC pauses in InfluxDB. 

### Final Thoughts

I tried to be objective but I don't know InfluxDB very well so maybe I'm just using it incorrectly. But anyway, I'd like to encourage you to evaluate all products on your own. Probably, your use case is quite different from mine. If this is the case, you will see different results.
