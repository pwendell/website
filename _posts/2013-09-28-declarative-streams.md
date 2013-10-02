---
layout: body
title: "Sampling Twitter Using Declarative Streams"
comments: true
---
##{{ page.title }}

_I'm often asked for long form examples of a complete 
[Spark](http://spark.incubator.apache.org/) application. 
Last week, I wanted to collect a large amount of data from Twitter's streaming 
API and put it into S3 to have as an experimental dataset. The resulting
Spark program seemed like a perfect one to share._

Before I started writing code I spent a few minutes sketching out 
what the program I was going to write needed to do:

<ol style="margin-left: 20px;">
<li> Connect to the Twitter API and continously ingest new Tweets.</li>
<li> Produce a delimited text record for each incoming Tweet.</li>
<li> Batch together all the tweets from each hour to avoid having tiny files.</li>
<li> Compress the batched tweets within each hour using GZip to save storage space.</li>
<li> Save the file in Amazon S3 in a new directory based on its date.</li>
<li> Repeat steps 1-5 indefinitely.</li>
</ol>

Writing an application from scratch do to this would a major effort because there 
are a bunch of tricky things in here. Managing the connection to the Twitter API, 
parsing and cleaning text fields, slicing together tweets from within certain time 
windows, dealing with compressing the data, and connecting to and writing against 
S3 are all nontrivial undertakings. 


In Spark, this entire pipeline can be written in about 100 lines of code.
The full project is [available on GitHub](https://github.com/pwendell/spark-twitter-collection/blob/master/TwitterCollector.scala)
including instructions on how to run it from a blank EC2 image. I'll walk through the interesting
parts here.

The first thing we do is initialize a `StreamingContext` using Spark's
[Streaming API](http://spark.incubator.apache.org/docs/latest/streaming-programming-guide.html).
Rather than connecting to a Spark cluster, this example runs a local Spark instance with
12 worker threads. Twitter only allows sampling a few megabytes per second, so this
doesn't need to run on several nodes.

{% highlight scala %}
// Create a local StreamingContext
val ssc = new StreamingContext("local[12]", "Twitter Downloader", Seconds(30))
{% endhighlight %} 

<p style="margin-top:15px;">
Spark Streaming represents streams as a sequence of micro-batches. When you initialize a 
StreamingContext you provide an internal batch interval. Spark streaming can support very 
small intervals (such as 1 second). In this case we use a 30 second interval, 
because this is not a latency sensitive service. In fact, we later aggregate tweets 
into much larger batches.
</p>

Spark's streaming API allows you to directly sample Twitter via a 
[`twitterStream`](http://spark.incubator.apache.org/docs/latest/api/streaming/index.html#org.apache.spark.streaming.StreamingContext). 
Twitter streams are composed of <a href="http://twitter4j.org/oldjavadocs/3.0.0/index.html">
Status</a> objects which represent indiviual tweets. To help us parse these tweets later, 
we create a list summarizing the fields we want to extract from each Status. This list 
also includes a column name and column type of the Hive output table 
we'd like to create:

{% highlight scala %}
// A list of fields we want along with Hive column names and data types
val fields: Seq[(Status => Any, String, String)] = Seq(
  (s => s.getId, "id", "BIGINT"),
  (s => s.getInReplyToStatusId, "reply_status_id", "BIGINT"),
  (s => s.getInReplyToUserId, "reply_user_id", "BIGINT"),
  (s => s.getRetweetCount, "retweet_count", "INT"),
  (s => s.getText, "text", "STRING"),
  (s => Option(s.getGeoLocation).map(_.getLatitude()).getOrElse(""), "latitude", "FLOAT"),
  (s => Option(s.getGeoLocation).map(_.getLongitude()).getOrElse(""), "longitude", "FLOAT")
) 
{% endhighlight %}

We use this `fields` list to create a function called `formatStatus` (not shown)
which extracts a list of fields form each tweet. Once we have a format function,
declaring the actual stream is straightforward:

{% highlight scala %}
// New Twitter stream
val statuses = ssc.twitterStream()

// Format each tweet
val formattedStatuses = statuses.map(s => formatStatus(s))

// Group into larger batches
val batchedStatuses = formattedStatuses.window(Minutes(60), Minutes(60))

// Coalesce each batch into fixed number of files
val coalesced = batchedStatuses.transform(rdd => rdd.coalesce(1))

// Save as output in correct directory
coalesced.foreach((rdd, time) =>  {
  val outPartitionFolder = outDateFormat.format(new Date(time.milliseconds))
  rdd.saveAsTextFile("%s/%s".format("s3n://my_bucket/", outPartitionFolder), 
    classOf[DefaultCodec])
})

/** Start Spark streaming */
ssc.checkpoint(checkpointDir)
ssc.start()
{% endhighlight %}

<br/>

Notice how closely this resembles the checklist we started with. In fact, there is
a 1:1 correspondence between steps 1-5 listed above and the Spark transformations
listed here. In general, using a declarative runtime like Spark offers several advantages:

* **_Simplicity_**: This code is concise and has very little boilerplate. It closely reflects the underlying logic and is easy to understand. This makes the code more maintainable and less prone to bugs.
* **_Testability_**: Each transformation on a Spark stream (`map`, `window`, etc) can be 
independently tested using mocked collections. Spark streaming also supports injecting synthetic
streams, which allows you to operators which cover multiple time periods such as `window`. 
* **_Scalability_**: Want to run this code on 5 nodes, 10 nodes, on 100 nodes? Because every
Spark operator is inherently parallel, you don't need to change any code when scale increases.
Task scheduling and data movement are all handled for you automatically as part of the runtime.

The `window` operator here may be less familiar to some users. It provides a
_sliding window_ over each micro-batch created by the underlying stream. This
can be used, for instance, to return a stream which represents a rolling view
of the last 5 minutes worth of data. That windowed view could then be used to
create rolling averages and other windowed statistics. In our case, we set the width of 
the window to be equal to the amount it "slides" each time, which has the 
effect of bucketing the tweets into larger batches of one hour.

See the [README file](https://github.com/pwendell/spark-twitter-collection/blob/master/README.txt) 
for instructions on how to run the project yourself. You'll need to setup a free Twitter API key as 
described [here](http://ampcamp.berkeley.edu/3/exercises/realtime-processing-with-spark-streaming.html#twitter-credential-setup). This program is written 
in Scala, but even those not familiar with Scala should be able to follow 
it fairly easily. Streaming programs never terminate, so you can run this for as long as 
you'd like. I've had it sampling Twitter for the last few days and in that time have 
collected several million tweets.
