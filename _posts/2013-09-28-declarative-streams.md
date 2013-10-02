---
layout: body
title: "Scraping Twitter Using Declarative Streams"
comments: true
---
##{{ page.title }}

_I'm often asked for long form examples of a complete 
[Spark](http://spark.incubator.apache.org/) application. 
Today, I wanted to collect a large amount of data from Twitter's streaming 
API and put it into S3 to have as an experimental dataset. The resulting
Spark program seemed like a perfect one to share with others._

Before I started writing code I spent a few minutes sketching out 
what the program I was going to write needed to do:

<ol style="margin-left: 20px;">
<li> Connect to the Twitter API and continously ingest new Tweets.</li>
<li> Produce a delimited text record for each incoming Tweet.</li>
<li> Batch together all the tweets from each hour (to avoid having tiny files).</li>
<li> Compress the batched tweets within each hour using GZip (to save storage space).</li>
<li> Save the file in Amazon S3 in a new directory based on its date.</li>
<li> Repeat steps 1-5 indefinitely.</li>
</ol>

Writing an application from scratch do to this would a major effort, because there 
are a bunch of tricky things in here. Managing the connection to the Twitter API, 
parsing and cleaning text fields, slicing together tweets from within certain time 
windows, dealing with compressing the data, and connecting to and writing against 
S3 are all nontrivial undertakings. 

<p style="text-align: center; ">[See the code on GitHub]</p>

In Spark, this entire pipeline can be written in about 100 lines of succinct
Scala code. Most of the code is devoted parametrizations of the script itself
and defining the schema of the output data we want. The actual stream
declaration almost exactly matches the pipeline we sketched out earlier:

{% highlight scala %}
// New Twitter stream
val statuses = ssc.twitterStream()

// Format each tweet
val formattedStatuses = statuses.map(s => formatStatus(s))

// Group into larger batches
val batchedStatuses = formattedStatuses
  .window(Minutes(60), Minutes(60))

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

The `window` operator here may be less familiar to some users. It provides a
[sliding window](XXX) over each micro-batch created by the underlying stream.
In our case we set the width of the window to be equal to the amount it "slides"
each time, which has the effect of bucketing the tweets into larger batches
of one hour. Windows can be used to create more sophisticated collections, such
as a rolling view of set of tweets within the last hour.

The largest "win" in writing this using Spark is that we're able to leverage
the [Streaming API](XXX)'s notion of continuous recomputation. We just
describe the dataflow once and under the covers things happen at periodic intervals.
The built in support for Twitter specifically and for writing to Hadoop files
using compression also solves our other requirements.

See the README file for instructions on how to run the project yourself.
You'll need to setup a Twitter API key as described [here](). It is written 
in Scala, but even those not familiar with Scala should be able to follow 
it fairly easily.

