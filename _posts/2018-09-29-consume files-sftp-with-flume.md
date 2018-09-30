---
title:  "Consume data over SFTP with Flume"
header:
  image: /assets/images/data-engineering.jpg
  caption: "Picture credit: [**braindomain.org**](https://braindomain.org/launch-of-the-big-data-analytics-journal/)"
  teaser: assets/images/2018-09-29-consume files-sftp-with-flume/flume-logo.png
excerpt: "Quickly set up a Flume agent to consume data over SFTP!"
categories:
  - data-engineering
---

{% include toc title="Navigation" %}

When many data scientists join the industry right after graduation, they're often disillusioned because the nature of their new work doesn't live upto their expectations. They quickly realize that data in the real world may be dirty, unreliable and may require an inordinate amount of engineering effort to ingest, clean, transform, store and maintain. This may be in stark contrast to the relatively pristine csv files that they may have taken for granted during their academic utpoia; data that let them focus on algorithms and models instead of housekeeping.

Lisha Li illustrates this nicely:
<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">.<a href="https://twitter.com/karpathy?ref_src=twsrc%5Etfw">@karpathy</a> : sleep-lost pie chart transition from academia to industry. Time spent worry about dataset &gt;&gt; time spent on model optimization <a href="https://t.co/nDmxo4VAC6">pic.twitter.com/nDmxo4VAC6</a></p>&mdash; lisha li (@lishali88) <a href="https://twitter.com/lishali88/status/994723759981453312?ref_src=twsrc%5Etfw">May 10, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


Robert Chang describes his own similar experience in his excellent post series - [A Beginner's Guide to Data Engineering](https://medium.com/@rchang/a-beginners-guide-to-data-engineering-part-i-4227c5c457d7). 

So, what does this have to do with Flume? As a data scientist in a startup or otherwise small team, you may, from time to time, need to do data engineering on your own to obtain data for your analyses. Flume often makes this process less painful, quicker, and even fun:bangbang:

## Why Flume
[Apache Flume][apache-flume] was originally built for collecting and moving large amounts of log data. However, its plugin-based architecture quickly sparked a flurry of plugins that let you ingest data from a variety of sources, including Kafka, JMS, NetCat, HTTP and many others, and load this data into a variety of destinations, including HDFS, HTTP, Solr, local files, etc. 

Flume is one of my favorite tools to ingest data with, because it lets you create simple *configuration-based* ingestion processes, called *agents* in Flume terminology. A simple configuration file may look like the following[^flume_example]:

```properties
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# Describe the sink
a1.sinks.k1.type = file_roll
a1.sinks.k1.channel = c1
a1.sinks.k1.sink.directory = /var/log/flume

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

This simple agent has one *source*, one *channel* and one *sink*, which are essentially plugins that let you read, buffer and store data. The agent loads data from the specified source (`netcat`), buffers the data into the specified channel (`memory`) and then loads it into the specified sink (`file_roll`). That's it! Once the agent is started, it ingests data continuously. 

If a little more flexibility is required in terms of the transformations to be done on the source data before being loaded into the sink, Flume provides the concept of *interceptors*. An interceptor is  pluggable custom code[^interceptor-language] that modifies input data[^flume-events] in-flight. Interceptors are a very interesting topic, one that I would love to discuss in detail in a future post.

Although Flume is very versatile with respect to transformations, some applications may require very complex transformations that may require the flexibility of a code-based approach, such as with [Apache Spark](http://spark.apache.org/).
{: .notice--warning}

Flume also makes it easy to create your own source/sink/channel plugins! We shall look at one such useful community plugin - `flume-ftp-source` - in the next section.

## Consuming data over SFTP

### Motivation
Although, for many applications, getting data as realtime as possible (e.g., using Kafka) is essential, (S)FTP data pipelines are all too common[^ftp-realtime]. FTP servers/accounts are easy to set up, maintain and system administrators everywhere are likely familiar with them. 

As a data engineer, wouldn't it be awesome if you could quickly set up a Flume agent to read data over (S)FTP from the specified server using the specified credentials, and quickly load it into your Hadoop cluster, all using a single configuration file? 

The folks over at Keedio sure agree. They've developed a nifty [FTP source plugin][keedio-flume-ftp-source] that lets you ingest data over FTP, SFTP and FTPS. 

### A simple example
Here's an example agent that transfers data over SFTP:

{% highlight properties linenos %}
agent.sources = sftp1
agent.sinks = logger1
agent.channels = mem1

# Source
# Type - SFTP
agent.sources.sftp1.type = org.keedio.flume.source.ftp.source.Source
agent.sources.sftp.client.source = sftp

# Source connection properties
agent.sources.sftp1.name.server = 142.95.254.34
agent.sources.sftp1.port = 22
agent.sources.sftp1.user = <username>
agent.sources.sftp1.password = <password>

# Source transfer properties
agent.sources.sftp1.working.directory = /home/meee
agent.sources.sftp1.filter.pattern = <filter_regex>
agent.sources.sftp1.run.discover.delay = 5000
agent.sources.sftp1.file.name = file_tracker.ser

# Sink
agent.sinks.logger1.type = logger

# Channel
agent.channels.mem1.type = memory
agent.channels.mem1.capacity = 1000
agent.channels.mem1.transactionCapacity = 100

# Bind source and sink to channel
agent.sources.sftp1.channels = mem1
agent.sinks.logger1.channel = mem1

{% endhighlight %}

For brevity, this example does not show all configuration properties supported by the `ftp-source-plugin`. For a comprehensive example configuration, see [here][sftp-example].
{: .notice--info}

In the above example, most properties are self-explanatory, but I'd like to draw your attention to lines 18 and 19. On line 18, the `<agent_name>.sources.sftp1.filter.pattern` property lets us specify a regular expression to filter the files discovered by the agent; only files whose names match the expression will be read. On line 19, the `<agent_name>.sources.sftp1.run.discover.delay` property specifies the time interval used by the agent to poll the server. A value of 5000 indicates that the agent will search the server for new files every 5 seconds. 

How does the agent know which files have already been read, so that it doesn't read them again? The agent keeps tracks of the files it has read in a special file, which can be specified by the property `<agent_name>.sources.sftp1.file.name` (line 20). 

### Decompressing files on-the-fly
In practice, I've often come across source files that are compressed using the GZIP compression codec. Inside these GZIP files are often `.csv` files. To address this especially interesting case, I've contributed a feature to the source repo that enables the Flume agent to decompress these GZIP files on-the-fly, and make the individual records inside the contained `.csv` files available as individual events in the channel. [My fork][hgadgil-flume-ftp-source] has some additional features that haven't yet been merged into the source repo, including automatically deleting source files once fully consumed. 

With this special case, our example agent may now look like this:

{% highlight properties linenos %}
agent.sources = sftp1
agent.sinks = logger1
agent.channels = mem1

# Source
# Type - SFTP
agent.sources.sftp1.type = org.keedio.flume.source.ftp.source.Source
agent.sources.sftp.client.source = sftp

# Source connection properties
agent.sources.sftp1.name.server = 142.95.254.34
agent.sources.sftp1.port = 22
agent.sources.sftp1.user = <username>
agent.sources.sftp1.password = <password>

# Source transfer properties
agent.sources.sftp1.working.directory = /home/meee
agent.sources.sftp1.filter.pattern = <filter_regex>
agent.sources.sftp1.run.discover.delay = 5000
agent.sources.sftp1.file.name = file_tracker.ser

# Source format properties
agent.sources.sftp1.compressed = gzip
agent.sources.sftp1.flushlines = true # Read line by line
agent.sources.sftp1.deleteOnCompletion = true

# Sink
agent.sinks.logger1.type = logger

# Channel
agent.channels.mem1.type = memory
agent.channels.mem1.capacity = 1000
agent.channels.mem1.transactionCapacity = 100

# Bind source and sink to channel
agent.sources.sftp1.channels = mem1
agent.sinks.logger1.channel = mem1

{% endhighlight %}

On line 23, we're now specifying that our source files are GZIP compressed. This will cause the agent to decompress the files *on-the-fly*. Additionally, on line 24, we're specifying that we want to read the decompressed data line-by-line, so that each event in the channel will be one individual record in the `.csv` files within the `.gz` files. Finally, on line 25, we're requesting that the agent delete the files once they're fully consumed by setting the `<agent_name>.sources.sftp1.deleteOnCompletion` property to `true`.

Setting up your ingestion process over SFTP using Flume is that simple! Contributions to the `flume-ftp-source` plugin, or my fork of it, are welcomed! :relaxed:

[Get SFTP source plugin on GitHub][hgadgil-flume-ftp-source]{: .btn .btn--success .btn--large}

[^flume_example]: Modified version of the simple example in the [official Flume documentation](https://flume.apache.org/FlumeUserGuide.html#a-simple-example)
[^interceptor-language]: Interceptors are usually written in Java, but could be written with any JVM based language
[^ftp-realtime]: That's not to say that it isn't possible to get data in realtime over FTP (indeed, the definition of "realtime" is heavily influenced by the application). However, it is reasonable to suggest that FTP isn't the most efficient medium for transferring data as realtime as possible.
[^flume-events]: Input data are called *events* in Flume terminology
[apache-flume]: https://flume.apache.org/
[keedio-flume-ftp-source]: https://github.com/keedio/flume-ftp-source
[hgadgil-flume-ftp-source]: https://github.com/opensorceror/flume-ftp-source
[sftp-example]: https://github.com/keedio/flume-ftp-source/blob/master/src/main/resources/example-configs/flume-ng-ftp-source-SFTP.conf