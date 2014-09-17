# Storm Metrics Statsd

storm-metrics-statsd is a module for [Storm](http://storm-project.net/) that enables metrics collection and reporting to [statsd](https://github.com/etsy/statsd/).

## Building/Installation

    git clone https://github.com/endgameinc/storm-metrics-statsd.git
    cd storm-metrics-statsd
    mvn compile package install

## Usage

This module can be used in two ways:

1. Configure it for each topology by calling `Conf.registerMetricsConsumer()` prior to launching the topology.
2. Deploy and configure system wide so usage of this is transparent across all topologies.

### Configure each topology separately

Add this as a dependency to your `pom.xml`

    <dependency>
      <groupId>com.endgame</groupId>
      <artifactId>storm-metrics-statsd</artifactId>
      <version>1.0.0-SNAPSHOT</version>
    </dependency>

Configure the `StatsdMetricConsumer` when building your topology.  The example below is
based on the [storm-starter](https://github.com/nathanmarz/storm-starter) [ExclamationTopology](https://github.com/nathanmarz/storm-starter/blob/master/src/jvm/storm/starter/ExclamationTopology.java).

    import com.endgame.storm.metrics.statsd.StatsdMetricConsumer;

    ...

    TopologyBuilder builder = new TopologyBuilder();
    builder.setSpout("word", new TestWordSpout(), 10);
    builder.setBolt("exclaim1", new ExclamationBolt(), 3).shuffleGrouping("word");
    builder.setBolt("exclaim2", new ExclamationBolt(), 2).shuffleGrouping("exclaim1");

    #
    #  Configure the StatsdMetricConsumer
    #
    Map statsdConfig = new HashMap();
    statsdConfig.put(StatsdMetricConsumer.STATSD_HOST, "statsd.server.mydomain.com");
    statsdConfig.put(StatsdMetricConsumer.STATSD_PORT, 8125);
    statsdConfig.put(StatsdMetricConsumer.STATSD_PREFIX, "storm.metrics.");

    Config conf = new Config();
    conf.registerMetricsConsumer(StatsdMetricConsumer.class, statsdConfig, 2);

    if (args != null && args.length > 0) {
      conf.setNumWorkers(3);
      StormSubmitter.submitTopology(args[0], conf, builder.createTopology());
    }
    else {
      LocalCluster cluster = new LocalCluster();
      cluster.submitTopology("test", conf, builder.createTopology());
      Utils.sleep(5*60*1000L);
      cluster.killTopology("test");
      cluster.shutdown();
    }

### System Wide Deployment

System wide deployment requires three steps:

#### 1. Add this section to your `$STORM_HOME/conf/storm.yaml`.  

    topology.metrics.consumer.register:
      - class: "com.endgame.storm.metrics.statsd.StatsdMetricConsumer"
         parallelism.hint: 2
         argument:
           metrics.statsd.host: "statsd.server.mydomain.com"
           metrics.statsd.port: 8125
           metrics.statsd.prefix: "storm.metrics."

#### 2. Install the `storm-metrics-statsd` and `java-statsd-client` JARs into `$STORM_HOME/lib/` ON EACH STORM NODE.

    $ mvn package
    $ mvn org.apache.maven.plugins:maven-dependency-plugin:2.7:copy-dependencies -DincludeArtifactIds=java-statsd-client
    $ cp target/dependency/java-statsd-client-1.0.1.jar $STORM_HOME/lib/
    $ cp target/storm-metrics-statsd-*.jar $STORM_HOME/lib/

#### 3. Restart storm and you will likely need to restart any topologies running prior to changing your `$STORM_HOME/conf/storm.yaml`.

### Notes

You can override the topology name used when reporting to statsd by calling:

    statsdConfig.put(Config.TOPOLOGY_NAME, "myTopologyName");
    // OR
    statsdConfig.put("topology.name", "myTopologyName");

This will be useful if you use versioned topology names (.e.g. appending a timestamp or a version string), but only care to track them as one in statsd.

## Tech details:

This is to get a feel of the data:
```
2014-09-08 17:25:37,925 302817   1410191857     storm-12.mytest.org:6705         -1:__system    memory/nonHeap          {unusedBytes=496928, maxBytes=136314880, usedBytes=34106080, initBytes=24576000, committedBytes=34603008, virtualFreeBytes=102208800}

2014-09-08 17:57:37,925 302817   1410191857     storm-12.mytest.org:6705         -1:__system    memory/nonHeap          {unusedBytes=496928, maxBytes=136314880, usedBytes=34106080, initBytes=24576000, committedBytes=34603008, virtualFreeBytes=102208800}
```

The metrics are received in data points for task.
Next values are in task id:
- timestamp
- srcWorkerHost
- srcWorkerPort
- srcTaskId
- srcComponentId
The data point has name and value.

the value is an object, and basing from examples below - it is a map.
The key in these maps are often composed with `/`


https://github.com/apache/incubator-storm/blob/master/external/storm-kafka/src/jvm/storm/kafka/PartitionManager.java#L111



Several examples below:

```

  GC/PSMarkSweep
  {count=0, timeMs=0}

  memory/nonHeap
  {unusedBytes=385408, maxBytes=136314880, usedBytes=33103488, initBytes=24576000, committedBytes=33488896, virtualFreeBytes=103211392}

  __emit-count
  {}

  __sendqueue
  {write_pos=90057, read_pos=90057, capacity=2048, population=0}

# received from kafka spout
kafkaPartition:
  {
   Partition{host=kafka-05.mytest.org:9092, partition=3}/fetchAPILatencyMean=309.0,
   Partition{host=kafka-05.mytest.org:9092, partition=3}/fetchAPICallCount=1,
   Partition{host=kafka-05.mytest.org:9092, partition=3}/fetchAPILatencyMax=309,
   Partition{host=kafka-05.mytest.org:9092, partition=3}/fetchAPIMessageCount=8350,

   Partition{host=kafka-06.mytest.org:9092, partition=0}/fetchAPIMessageCount=0,
   Partition{host=kafka-06.mytest.org:9092, partition=0}/fetchAPILatencyMean=null,
   Partition{host=kafka-06.mytest.org:9092, partition=0}/fetchAPILatencyMax=null,
   Partition{host=kafka-06.mytest.org:9092, partition=0}/fetchAPICallCount=0,

   Partition{host=kafka-07.mytest.org:9092, partition=1}/fetchAPIMessageCount=8082
   Partition{host=kafka-07.mytest.org:9092, partition=1}/fetchAPILatencyMean=99.0,
   Partition{host=kafka-07.mytest.org:9092, partition=1}/fetchAPICallCount=1,
   Partition{host=kafka-07.mytest.org:9092, partition=1}/fetchAPILatencyMax=99,

   Partition{host=kafka-08.mytest.org:9092, partition=2}/fetchAPIMessageCount=0,
   Partition{host=kafka-08.mytest.org:9092, partition=2}/fetchAPILatencyMax=null,
   Partition{host=kafka-08.mytest.org:9092, partition=2}/fetchAPICallCount=0,
   Partition{host=kafka-08.mytest.org:9092, partition=2}/fetchAPILatencyMean=null,
  }

kafkaOffset
  {partition_3/latestTimeOffset=30109296760,
   totalSpoutLag=610889380,
   totalLatestTimeOffset=30109296760,
   totalLatestEmittedOffset=29498407380,
   partition_3/latestEmittedOffset=29498407380,
   totalEarliestTimeOffset=29498407377,
   partition_3/earliestTimeOffset=29498407377,
   partition_3/spoutLag=610889380}

```

As we may see, these metrics are too custom to be correctly moved to statsd format.

So far I have excluded partition (because it is hardcoded in kafka spout).
Also changed '/' to '.' so the metrics would be nested.

Looking forward to come up with some standard naming approach in kafka spout 
so it can be processed easily by machines then.

## License

storm-metrics-statsd

Copyright 2014 [Endgame, Inc.](http://www.endgame.com/)

![Endgame, Inc.](http://www.endgame.com/images/logo.svg)


        Licensed under the Apache License, Version 2.0 (the "License"); you may
        not use this file except in compliance with the License. You may obtain
        a copy of the License at

             http://www.apache.org/licenses/LICENSE-2.0

        Unless required by applicable law or agreed to in writing,
        software distributed under the License is distributed on an
        "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
        KIND, either express or implied.  See the License for the
        specific language governing permissions and limitations
        under the License.

## Author

[Jason Trost](https://github.com/jt6211/) ([@jason_trost](https://twitter.com/jason_trost))
