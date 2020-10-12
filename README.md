# SparkPlugins
![SparkPlugins CI](https://github.com/cerndb/SparkPlugins/workflows/SparkPlugins%20CI/badge.svg)

Code and examples of how to use Apache Spark Plugin extensions to the metrics system with Apache Spark 3.0.

---
Apache Spark 3.0 comes with a new plugin framework. Plugins allow extending Spark monitoring functionality.
- One important use case is extending Spark instrumentation with custom metrics:  
  OS metrics, I/O metrics, external applications monitoring, etc.
- Note: The code in this repo is for Spark 3.x.  
   For Spark 2.x, see instead [Executor Plugins for Spark 2.4](https://github.com/cerndb/SparkExecutorPlugins2.4)

Plugin notes:
- Spark plugins implement the `org.apache.spark.api.Plugin` interface, they can be written in Scala or Java
 and can be used to run custom code at the startup of Spark executors and driver.  
- Plugins configuration: `--conf spark.plugins=<list of plugin classes>`
- Plugin JARs need to be made available to Spark executors
  - YARN: you can distribute the plugin code to the executors using `--jars`.
  - K8S, when using Spark 3.0.1 on K8S, `--jars` distribution will **not work**, you will need to make the JAR available in the Spark container when you build it.
    - [SPARK-32119](https://issues.apache.org/jira/browse/SPARK-32119) fixes this issue and allows to use `--jars` to distribute plugin code. 
- Link to [Spark monitoring documentation](https://spark.apache.org/docs/latest/monitoring.html#advanced-instrumentation)
- See also [SPARK-29397](https://issues.apache.org/jira/browse/SPARK-29397), [SPARK-28091](https://issues.apache.org/jira/browse/SPARK-28091).

Author and contact: Luca.Canali@cern.ch 

## Getting Started  
- Build or download the SparkPlugin `jar`
  - Build from source with:
    ```
    sbt package
    ```
  - Or download from the automatic build in [github actions](https://github.com/cerndb/SparkPlugins/actions)

- Test and debug with a demo plugin `ch.cern.RunOSCommandPlugin`:   
  this runs an OS command on the executors, customizable (see below), by default `"/usr/bin/touch /tmp/plugin.txt"`.
  ```
  bin/spark-shell --master yarn \ 
    --jars <path>/sparkplugins_2.12-0.1.jar \
    --conf spark.plugins=ch.cern.RunOSCommandPlugin 
  ```
  - You can see if the plugin has run by checking that the file `/tmp/plugin.txt` has been
   created on the executor machines. 

## How to use the metrics produced by the plugins

  - You can find the metrics generated by the plugin in the Spark metrics system stream (see details below).
  - Spark Dashboard using the metrics system:    
    Link with details on [how to deploy a dashboard using Spark metrics](https://github.com/cerndb/spark-dashboard)
    This repo contains code and Spark Plugin examples to extend [Spark metrics instrumentation](https://spark.apache.org/docs/3.0.0/monitoring.html#metrics)
    and use an extended grafana dashboard with the Plugin metrics of interest.

## Plugins in this Repository

### Demo and basics
  - [DemoPlugin](src/main/scala/ch/cern/DemoPlugin.scala)
    - `--conf spark.plugins=ch.cern.DemoPlugin`
    - Basic plugin, demonstrates how to write Spark plugins in Scala, for demo and testing.
  - [DemoMetricsPlugin](src/main/scala/ch/cern/DemoMetricsPlugin.scala)
    - `--conf spark.plugins=ch.cern.DemoMetricsPlugin`
    - Example plugin illustrating integration with the Spark metrics system.
    - Metrics implemented:
      - `ch.cern.DemoMetricsPlugin.DriverTest42`: a gauge reporting a constant integer value, for testing.
  - [RunOSCommandPlugin](src/main/scala/ch/cern/RunOSCommandPlugin.scala)
    - `--conf spark.plugins=ch.cern.RunOSCommandPlugin`
    - Example illustrating how to use plugins to run actions on the OS.
    - Action implemented: runs an OS command on the executors, by default it runs: `/usr/bin/touch /tmp/plugin.txt`
    - Configurable action: `--conf spark.cernSparkPlugin.command="command or script you want to run"`

### OS metrics instrumentation with cgroups, for Spark on Kubernetes 
  - [CgroupMetrics](src/main/scala/ch/cern/CgroupMetrics.scala)
    - Configure with: `--conf spark.plugins=ch.cern.CgroupMetrics`
    - Optional configuration: `--conf spark.cernSparkPlugin.registerOnDriver` (default false)
    - Implemented using cgroup instrumentation of key system resource usage, intended mostly for
      Spark on Kubernetes
    - Collects metrics using CGroup stats from `/sys/fs` and from `/proc` filesystem 
      for CPU, Memory and Network usage. See also [kernel documentation](https://www.kernel.org/doc/Documentation/cgroup-v1)
      Note: the metrics are reported for the entire cgroup to which the executor belongs to. This is mostly
      intended for Spark running on Kubernetes. In other cases, the metrics reported
      may not be easily correlated with executor's activity, as the cgroup metrics may include more
      processes, up to the entire system.
    - Metrics implemented (gauges), with prefix `ch.cern.CgroupMetrics`:
      - `CPUTimeNanosec`: reports the CPU time used by the processes in the cgroup.
      - `MemoryRss`: number of bytes of anonymous and swap cache memory.
      - `MemorySwap`: number of bytes of swap usage.
      - `MemoryCache`: number of bytes of page cache memory.
      - `NetworkBytesIn`: network traffic inbound.
      - `NetworkBytesOut`: network traffic outbound.

    - Example:
       - Note this example is intended for Spark build with [SPARK-32119](https://issues.apache.org/jira/browse/SPARK-32119),
         such as Spark 3.1 on K8S, for Spark 3.0.1 on K8S you need to build the container with the plugin jar.
    ```
    bin/spark-shell --master k8s://https://<K8S URL>:6443 --driver-memory 1g \ 
      --num-executors 2 --executor-cores 2 --executor-memory 2g \
      --conf spark.kubernetes.container.image=
      --jars <path>/sparkplugins_2.12-0.1.jar \
      --conf spark.plugins=ch.cern.HDFSMetrics,ch.cern.CgroupMetrics \
      --conf "spark.metrics.conf.driver.sink.graphite.class"="org.apache.spark.metrics.sink.GraphiteSink"   \
      --conf "spark.metrics.conf.executor.sink.graphite.class"="org.apache.spark.metrics.sink.GraphiteSink"         \
      --conf "spark.metrics.conf.driver.sink.graphite.host"=mytestinstance \
      --conf "spark.metrics.conf.executor.sink.graphite.host"=mytestinstance \
      --conf "spark.metrics.conf.*.sink.graphite.port"=2003 \
      --conf "spark.metrics.conf.*.sink.graphite.period"=10 \
      --conf "spark.metrics.conf.*.sink.graphite.unit"=seconds \
      --conf "spark.metrics.conf.*.sink.graphite.prefix"="youridhere"
    ```

   - Visualize the metrics with the Spark dashboard `spark_perf_dashboard_spark3-0_v02_with_sparkplugins`

### Plugins to collect I/O storage statistics for HDFS and Hadoop Compatible Filesystems

#### HDFS extended storage statistics
This Plugin measures HDFS extended statistics.
In particular it provides information on read locality and erasure coding usage (for HDFS 3.x).

  - [HDFSMetrics](src/main/scala/ch/cern/HDFSMetrics.scala)
    - Configure with: `--conf spark.plugins=ch.cern.HDFSMetrics`
    - Optional configuration: `--conf spark.cernSparkPlugin.registerOnDriver` (default true)
    - Collects extended HDFS metrics using Hadoop's [GlobalStorageStatistics](https://hadoop.apache.org/docs/current/api/org/apache/hadoop/fs/GlobalStorageStatistics.html)
      implemented using Hadoop extended statistics metrics introduced in Hadoop 2.8.
    - Use this with Spark built with Hadoop 3.2 or higher
      (it does not work with Spark built with Hadoop 2.7).
    - Metrics (gauges) implemented have the prefix `ch.cern.HDFSMetrics`. List of metrics:
       - `bytesRead`
       - `bytesWritten`
       - `readOps`
       - `writeOps`
       - `largeReadOps`
       - `bytesReadLocalHost`
       - `bytesReadDistanceOfOneOrTwo`
       - `bytesReadDistanceOfThreeOrFour`
       - `bytesReadDistanceOfFiveOrLarger`
       - `bytesReadErasureCoded`

    - Example  
    ```
    bin/spark-shell --master yarn \
      --jars <path>/sparkplugins_2.12-0.1.jar \
      --conf spark.plugins=ch.cern.HDFSMetrics \
      --conf "spark.metrics.conf.driver.sink.graphite.class"="org.apache.spark.metrics.sink.GraphiteSink"   \
      --conf "spark.metrics.conf.executor.sink.graphite.class"="org.apache.spark.metrics.sink.GraphiteSink"         \
      --conf "spark.metrics.conf.driver.sink.graphite.host"=mytestinstance \
      --conf "spark.metrics.conf.executor.sink.graphite.host"=mytestinstance \
      --conf "spark.metrics.conf.*.sink.graphite.port"=2003 \
      --conf "spark.metrics.conf.*.sink.graphite.period"=10 \
      --conf "spark.metrics.conf.*.sink.graphite.unit"=seconds \
      --conf "spark.metrics.conf.*.sink.graphite.prefix"="youridhere"
    ```

   - Visualize the metrics with the Spark dashboard `spark_perf_dashboard_spark3-0_v02_with_sparkplugins`


### Cloud filesystem storage statistics for Hadoop Compatible Filesystems
#### [CloudFSMetrics](src/main/scala/ch/cern/CloudFSMetrics.scala)
This Plugin provides I/O statistics for Cloud Filesystem metrics (for S3A, GS, root, oci, azure, and any other
filesystem exposed as a Hadoop Compatible Filesystem). 

  - Configure with: 
    - `--conf spark.plugins=ch.cern.CloudFSMetrics`
    - `--conf spark.cernSparkPlugin.cloudFsName=<name of the filesystem>` (example: "s3a", "oci", "gs", "root", etc.) 
    - Optional configuration: `--conf spark.cernSparkPlugin.registerOnDriver` (default true)  
    - Collects I/O metrics for Hadoop-compatible filesystems using Hadoop's GlobalStorageStatistics API.   
      Note: use this with Spark built with Hadoop 3.2 (requires Hadoop client version 2.8 or higher).
    - Metrics (gauges) implemented have the prefix `ch.cern.S3AMetricsGSS`. List of metrics:
       - `bytesRead`
       - `bytesWritten`
       - `readOps`
       - `writeOps`
    - Example:
         - Note this example is intended for Spark build with [SPARK-32119](https://issues.apache.org/jira/browse/SPARK-32119),
           such as Spark 3.1 on K8S, for Spark 3.0.1 on K8S you need to build the container with the plugin jar.
         ```
         bin/spark-shell --master k8s://https://<K8S URL>:6443 --driver-memory 1g \ 
          --num-executors 2 --executor-cores 2 --executor-memory 2g \
          --conf spark.kubernetes.container.image=<registry>/spark:v301 \
          --jars <path_to_the_Plugin_jar>/sparkplugins_2.12-0.1.jar \
          --conf spark.plugins=ch.cern.CloudFSMetrics,ch.cern.CgroupMetrics \
          --conf spark.cernSparkPlugin.cloudFsName="s3a" \
          --packages org.apache.hadoop:hadoop-aws:3.2.0 \
          --conf spark.hadoop.fs.s3a.secret.key="<SECRET KEY HERE>" \
          --conf spark.hadoop.fs.s3a.access.key="<ACCESS KEY HERE>" \
          --conf spark.hadoop.fs.s3a.endpoint="https://<S3A URL HERE>" \
          --conf spark.hadoop.fs.s3a.impl="org.apache.hadoop.fs.s3a.S3AFileSystem" \
          --conf "spark.metrics.conf.driver.sink.graphite.class"="org.apache.spark.metrics.sink.GraphiteSink"   \
          --conf "spark.metrics.conf.executor.sink.graphite.class"="org.apache.spark.metrics.sink.GraphiteSink"         \
          --conf "spark.metrics.conf.driver.sink.graphite.host"=mytestinstance \
          --conf "spark.metrics.conf.executor.sink.graphite.host"=mytestinstance \
          --conf "spark.metrics.conf.*.sink.graphite.port"=2003 \
          --conf "spark.metrics.conf.*.sink.graphite.period"=10 \
          --conf "spark.metrics.conf.*.sink.graphite.unit"=seconds \
          --conf "spark.metrics.conf.*.sink.graphite.prefix"="youridhere"
         ```

   - Visualize the metrics with the Spark dashboard `spark_perf_dashboard_spark3-0_v02_with_sparkplugins`

#### [CloudFSMetrics27](src/main/scala/ch/cern/CloudFSMetrics27.scala) 
This Plugin provides I/O statistics for Cloud Filesystem metrics (for S3A, GS, root, oci, azure, and any other
filesystem exposed as a Hadoop Compatible Filesystem). Use this for Spark built using Hadoop 2.7.
  - Configure with:
    - `--conf spark.plugins=ch.cern.CloudFSMetrics27`
    - `--conf spark.cernSparkPlugin.cloudFsName=<name of the filesystem>` (example: "s3a", "oci", "gs", "root", etc.)
    - Optional configuration: `--conf spark.cernSparkPlugin.registerOnDriver` (default true)  
    - Collects I/O metrics for Hadoop-compatible filesystem using Hadoop 2.7 API, use with Spark built with Hadoop 2.7
    - Metrics are the same as for CloudFSMetrics, they the prefix `ch.cern.CloudFSMetrics27`.

## Experimental Plugins for I/O time instrumentation

This section details a few experimental Spark plugins used to expose metrics for I/O-time
instrumentation of Spark workloads using Hadoop-compliant filesystems.  
These plugins use instrumented experimental/custom versions of the Hadoop client API for HDFS and other Hadoop-Compliant File Systems.

  - [S3A Time Instrumentation](src/main/scala/ch/cern/experimental/S3ATimeInstrumentation.scala) 
    - Instruments the Hadoop S3A client.
    - Note: this requires custom S3A client implementation, see experimental code at: [HDFS and S3A custom instrumentation](https://github.com/LucaCanali/hadoop/tree/s3aAndHDFSTimeInstrumentation)  
    - Spark config:
      - `--conf spark.plugins=ch.cern.experimental.S3ATimeInstrumentation`
      - Additional jar: `--jars hadoop-aws-3.2.0.jar` built [from this fork](https://github.com/LucaCanali/hadoop/tree/s3aAndHDFSTimeInstrumentation)
      ```
      --jars <PATH>/hadoop-aws-3.2.0.jar, <PATH>/ sparkplugins_2.12-0.1.jar \ # Note for K8S rather add this to the container
      --packages com.amazonaws:aws-java-sdk-bundle:1.11.804 \
      --conf spark.hadoop.fs.s3a.secret.key="<SECRET KEY HERE>" \
      --conf spark.hadoop.fs.s3a.access.key="<ACCESS KEY HERE>" \
      --conf spark.hadoop.fs.s3a.endpoint="https://<URL HERE>" \
      --conf spark.hadoop.fs.s3a.impl="org.apache.hadoop.fs.s3a.S3AFileSystem"
      ```
    - Metrics implemented (gauges), with prefix `ch.cern.experimental.S3ATimeInstrumentation`:
        - `S3AReadTimeMuSec`
        - `S3ASeekTimeMuSec`
        - `S3ACPUTimeDuringReadMuSec`
        - `S3ACPUTimeDuringSeekMuSec`
        - `S3AReadTimeMinusCPUMuSec`
        - `S3ASeekTimeMinusCPUMuSec`
        - `S3ABytesRead`
        - `S3AGetObjectMetadataMuSec`
        - `S3AGetObjectMetadataCPUMuSec`
        
  - [HDFS Time Instrumentation](src/main/scala/ch/cern/experimental/HDFSTimeInstrumentation.scala) 
    - Instruments the Hadoop HDFS client.
    - Note: this requires custom HDFS client implementation, see experimental code at: [HDFS and S3A custom instrumentation](https://github.com/LucaCanali/hadoop/tree/s3aAndHDFSTimeInstrumentation)
    - Spark config:
      - `--conf spark.plugins=ch.cern.experimental.HDFSTimeInstrumentation`
      - `--jars <PATH>/sparkplugins_2.12-0.1.jar`
      - Hack required for testing: replace `$SPARK_HOME/jars/hadoop-hdfs-client-3.2.0.jar` with the jar built [from this fork](https://github.com/LucaCanali/hadoop/tree/s3aAndHDFSTimeInstrumentation)
    - Metrics implemented (gauges), with prefix `ch.cern.experimental.HDFSTimeInstrumentation`:
        - `HDFSReadTimeMuSec`
        - `HDFSCPUTimeDuringReadMuSec`
        - `HDFSReadTimeMinusCPUMuSec`
        - `HDFSBytesRead`
        - `HDFSReadCalls`
        
  - [Hadoop-XRootD Time Instrumentation](src/main/scala/ch/cern/experimental/ROOTTimeInstrumentation.scala) 
    - Collects metrics for the Hadoop-XRootD connector 
    - Intended use is when using Spark with the CERN EOS (and CERN Box) storage system.
    - Additional details: [Hadoop-XRootD connector instrumentation](https://github.com/cerndb/hadoop-xrootd/blob/master/src/main/java/ch/cern/eos/XRootDInstrumentation.java) 
    - Spark config:
      - `--conf spark.plugins=ch.cern.experimental.ROOTTimeInstrumentation`
      ```
      --jars <PATH>/sparkplugins_2.12-0.1.jar \ # Note for K8S rather add this to the container
      --conf spark.driver.extraClassPath=<path>/hadoop-xrootd-1.0.5.jar \
      --conf spark.executor.extraClassPath=<path>/hadoop-xrootd-1.0.5.jar \
      --files <path_to_krbtgt>#krbcache \
      --conf spark.executorEnv.KRB5CCNAME="FILE:krbcache"
      ```
    - Metrics implemented (gauges), with prefix `ch.cern.experimental.ROOTTimeInstrumentation`:
        - `ROOTBytesRead`
        - `ROOTReadOps`
        - `ROOTTimeElapsedReadMusec`
        
   - [OCITimeInstrumentation](src/main/scala/ch/cern/experimental/OCITimeInstrumentation.scala) 
    - instruments the HDFS connector to Oracle OCI storage.
    - Note: this requires a custom hdfs-oci connector implementation, see experimental code at:
     [OCI-Hadoop connector instrumentation](https://github.com/LucaCanali/oci-hdfs-connector/blob/BMCInstrumentation/hdfs-connector/src/main/java/com/oracle/bmc/hdfs/store)
    - Spark config:
      - `--conf spark.plugins=ch.cern.experimental.OCITimeInstrumentation`
      - Hack required for testing using Spark on Kubernetes: 
        copy `oci-hdfs-connector-2.9.2.6.jar` built [from this fork](https://github.com/LucaCanali/oci-hdfs-connector/)
        into `$SPARK_HOME/jars`
        copy and install the relevant [oracle/oci-java-sdk release jars](oci-java-sdk release jars):
          version 1.17.5: oci-java-sdk-full-1.17.5.jar and related third party jars.
      ```
      --jars <PATH>/sparkplugins_2.12-0.1.jar \ # Note for K8S rather add this to the container
      --conf spark.hadoop.fs.oci.client.auth.pemfilepath="<PATH>/oci_api_key.pem" \
      --conf spark.hadoop.fs.oci.client.auth.tenantId=ocid1.tenancy.oc1..TENNANT_KEY \
      --conf spark.hadoop.fs.oci.client.auth.userId=ocid1.user.oc1.USER_KEY \
      --conf spark.hadoop.fs.oci.client.auth.fingerprint=<fingerprint_here> \
      --conf spark.hadoop.fs.oci.client.hostname="https://objectstorage.REGION_NAME.oraclecloud.com" \
      ```
     - Metrics implemented (gauges), with prefix `ch.cern.experimental.OCITimeInstrumentation`:
         - `OCIReadTimeMuSec`
         - `OCISeekTimeMuSec`
         - `OCICPUTimeDuringReadMuSec`
         - `OCICPUTimeDuringSeekMuSec`
         - `OCIReadTimeMinusCPUMuSec`
         - `OCISeekTimeMinusCPUMuSec`
         - `OCIBytesRead`
