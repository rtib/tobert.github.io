---
id: "2014-11-07-docker-entrypoints"
title: "Docker Entrypoints and the UX of Containers"
abstract: "Docker entrypoints provide an opportunity to give your users a better UX than they would have without Docker"
tags: ["docker", "cassandra", "entrypoint", "sprok"]
pubdate: 2014-11-07T20:30:00Z
---

### Entrypoints

When building containers using Dockerfiles, you can specify an entrypoint for the container
that Docker will use as process 1 inside the container. This is the default command that
is executed on `docker run`.

It seems that most people are going with a process supervisor these days. That's a fine
approach, but it's not what I'm here to talk about.

### Precision

One of my goals for the [cassandra-docker](https://github.com/tobert/cassandra-docker) images
is to have very precise control over how Cassandra and its tools start. It is common to have to edit
shell scripts to tune projects that use the JVM and this has always bothered me. What started out as a
clever way to build a classpath has grown over time to be a sprawling set of shell scripts with lots
of undefined behavior. I also don't like the way '../' shows up in so many paths. It could be cleaned up
but it works, so there it is. It makes me itchy every time I see it.

> cassandra 9359 9358 19 19:34 pts/0 00:00:06 java -ea -javaagent:./../lib/jamm-0.2.6.jar -XX:+CMSClassUnloadingEnabled -XX:+UseThreadPriorities -XX:ThreadPriorityPolicy=42 -Xms3941M -Xmx3941M -Xmn800M -XX:+HeapDumpOnOutOfMemoryError -Xss256k -XX:StringTableSize=1000003 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=1 -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+UseTLAB -XX:CompileCommandFile=./../conf/hotspot_compiler -XX:CMSWaitDuration=10000 -XX:+UseCondCardMark -Djava.net.preferIPv4Stack=true -Dcom.sun.management.jmxremote.port=7199 -Dcom.sun.management.jmxremote.rmi.port=7199 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dlogback.configurationFile=logback.xml -Dcassandra.logdir=./../logs -Dcassandra.storagedir=./../data -Dcassandra-foreground=yes -cp ./../conf:./../build/classes/main:./../build/classes/thrift:./../lib/airline-0.6.jar:./../lib/antlr-runtime-3.5.2.jar:./../lib/apache-cassandra-2.1.1.jar:./../lib/apache-cassandra-clientutil-2.1.1.jar:./../lib/apache-cassandra-thrift-2.1.1.jar:./../lib/commons-cli-1.1.jar:./../lib/commons-codec-1.2.jar:./../lib/commons-lang3-3.1.jar:./../lib/commons-math3-3.2.jar:./../lib/compress-lzf-0.8.4.jar:./../lib/concurrentlinkedhashmap-lru-1.4.jar:./../lib/disruptor-3.0.1.jar:./../lib/guava-16.0.jar:./../lib/high-scale-lib-1.0.6.jar:./../lib/jackson-core-asl-1.9.2.jar:./../lib/jackson-mapper-asl-1.9.2.jar:./../lib/jamm-0.2.6.jar:./../lib/javax.inject.jar:./../lib/jbcrypt-0.3m.jar:./../lib/jline-1.0.jar:./../lib/jna-4.0.0.jar:./../lib/json-simple-1.1.jar:./../lib/libthrift-0.9.1.jar:./../lib/logback-classic-1.1.2.jar:./../lib/logback-core-1.1.2.jar:./../lib/lz4-1.2.0.jar:./../lib/metrics-core-2.2.0.jar:./../lib/netty-all-4.0.23.Final.jar:./../lib/reporter-config-2.1.0.jar:./../lib/slf4j-api-1.7.2.jar:./../lib/snakeyaml-1.11.jar:./../lib/snappy-java-1.0.5.2.jar:./../lib/stream-2.5.2.jar:./../lib/stringtemplate-4.0.2.jar:./../lib/super-csv-2.1.0.jar:./../lib/thrift-server-0.3.7.jar org.apache.cassandra.service.CassandraDaemon

So what about precision?

While the most common options - heap sizes - can be overridden in `/etc/{default,sysconfig}/cassandra`, there are plenty
of other settings I'm interested in benchmarking that require editing a shell script. This could be automated using
templates, but the number of input variables grows to include most of them quickly if you want to avoid using conflicting
settings (such as different GC strategies).

```
# GC tuning options
JVM_OPTS="$JVM_OPTS -XX:+UseParNewGC"
JVM_OPTS="$JVM_OPTS -XX:+UseConcMarkSweepGC"
JVM_OPTS="$JVM_OPTS -XX:+CMSParallelRemarkEnabled"
JVM_OPTS="$JVM_OPTS -XX:SurvivorRatio=8"
JVM_OPTS="$JVM_OPTS -XX:MaxTenuringThreshold=1"
JVM_OPTS="$JVM_OPTS -XX:CMSInitiatingOccupancyFraction=75"
JVM_OPTS="$JVM_OPTS -XX:+UseCMSInitiatingOccupancyOnly"
JVM_OPTS="$JVM_OPTS -XX:+UseTLAB"
JVM_OPTS="$JVM_OPTS -XX:CompileCommandFile=$CASSANDRA_CONF/hotspot_compiler"
JVM_OPTS="$JVM_OPTS -XX:CMSWaitDuration=10000"

```

By the time you get all of the options into a data file, it starts to look an awwful lot like
an argv array ...

```yaml
argv:
  - /usr/bin/java
  - -ea
  - -javaagent:{{ glob "" "/opt/cassandra/lib/jamm-*.jar" }}
  - -XX:+CMSClassUnloadingEnabled
  - -XX:+UseThreadPriorities
  - -XX:ThreadPriorityPolicy=42
  - -Xms3941M
  - -Xmx3941M
  - -Xmn800M
  ...
```

In the entrypoint for cassandra-docker, [this is exactly how it's done](https://github.com/tobert/cassandra-docker/blob/master/conf/sproks/cassandra.yaml).