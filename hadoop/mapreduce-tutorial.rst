################################
MapReduce开发教程
################################

本文翻译自 Apache Hadoop 官方文档 `MapReduce Tutorial <http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html>`_ 。

* :ref:`purpose`
* :ref:`prerequisites`
* :ref:`overview`
* :ref:`inputs_and_outputs`
* :ref:`example_wordcount_v1`
    * :ref:`example_wordcount_v1_source_code`
    * :ref:`example_wordcount_v1_usage`
    * :ref:`example_wordcount_v1_walk-through`
* :ref:`mapreduce_user_interfaces`
    * :ref:`payload`
        * :ref:`payload_mapper`
        * :ref:`payload_reducer`
        * :ref:`payload_partitioner`
        * :ref:`payload_counter`
    * :ref:`job_configuration`
    * :ref:`task_execution_and_environment`
        * :ref:`memory_management`
        * :ref:`map_parameters`
        * :ref:`shuffle_reduce_parameters`
        * :ref:`configured_parameters`
        * :ref:`task_logs`
        * :ref:`distributing_libraries`
    * :ref:`job_submission_and_monitoring`
        * :ref:`job_control`
    * :ref:`job_input`
        * :ref:`input_split`
        * :ref:`record_reader`
    * :ref:`job_output`
        * :ref:`output_committer`
        * :ref:`task_side_effect_files`
        * :ref:`record_writer`
    * :ref:`other_useful_features`
        * :ref:`submitting_jobs_to_queues`
        * :ref:`counters`
        * :ref:`distributed_cache`
        * :ref:`profiling`
        * :ref:`debugging`
        * :ref:`data_compression`
        * :ref:`skipping_bad_records`
    * :ref:`example_wordcount_v2`
        * :ref:`example_wordcount_v2_source_code`
        * :ref:`example_wordcount_v2_sample_runs`
        * :ref:`example_wordcount_v2_highlights`


.. _purpose:

********************************
目标
********************************

这篇教程从用户的角度出发，全面地介绍了 Hadoop MapReduce 框架的各个方面。

.. _prerequisites:

********************************
前提条件
********************************

请先确保已经正确地安装和配置 Hadoop 并且已经成功地运行 Hadoop。更多细节参见:

* `单节点集群搭建 <http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html>`_ - 针对首次使用 Hadoop 的用户。
* `分布式集群搭建 <http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/ClusterSetup.html>`_ - 针对大规模分布式集群。

.. _overview:

********************************
概述
********************************

Hadoop MapReduce 是一个软件框架，用于很容易地编写出以可靠的、可容错的方式在由上千个商用机器组成的大型集群上并行处理大量数据(上T级别的数据集)的应用程序。

MapReduce 作业通常将输入数据集切分成若干独立的块，这些数据块由 map 任务以完全并行的方式进行处理。MapReduce 框架对 map 的输出先进行排序，然后将排序结果输入到 reduce 任务。通常，作业的输入和输出都存储在文件系统中。MapReduce 框架负责调度任务，监控它们并重新执行失败的任务。

计算节点和存储节点通常都是相同的, 也就是说, MapReduce 框架和 Hadoop 分布式文件系统(参见 `HDFS架构指南 <http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html>`_)是在同一组节点上运行的。该配置允许框架在已经存在数据的节点上有效地调度任务, 从而在整个集群中产生非常高的聚合带宽。

MapReduce 框架由单个充当 master 节点的 ResourceManager，每个集群节点对应一个充当 worker 节点的 NodeManager 以及每个应用程序对应的一个 MRAppMaster 组成(参见 `YARN 架构指南 <http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html>`_ )。

最低限度，应用程序会通过实现适当的接口和/或抽象类来指定输入/输出位置以及提供 map 和 reduce函数。这些和其他作业参数构成了作业配置。

Hadoop作业客户端将作业(jar或可执行文件等)和配置提交给 ResourceManager，然后 ResourceManager 负责将软件/配置分发到 worker 节点，调度任务并监控它们，并向作业客户端提供状态和诊断信息。

尽管 Hadoop 框架是用 Java™ 实现的, 但是 MapReduce 应用程序不一定要用 Java 编写。

* `Hadoop Streaming <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/streaming/package-summary.html>`_ 是一个实用程序，它允许用户使用任何可执行文件(比如 shell 实用程序) 作为 mapper 和/或 reducer 来创建和运行作业。
* `Hadoop Pipes <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/pipes/package-summary.html>`_ 是一个兼容 `SWIG <http://www.swig.org>`_ 的 C++ API，用于实现 MapReduce 应用程序(不是基于 JNI™)。


.. _inputs_and_outputs:

********************************
输入和输出
********************************

MapReduce 框架专门处理键值对, 也就是说框架将作业的输入视作一组键值对并产生一组键值对作为作业的输出, 这两组键值对的类型可能不同。

key 和 value 类需要被框架序列化，因此需要实现 `Writable <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/io/Writable.html>`_ 接口。另外, 为了便于框架进行排序, key 类必须实现 `WritableComparable <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/io/WritableComparable.html>`_ 接口。

下面是一个 MapReduce 作业的输入和输出类型:

.. code-block:: text

  (input) <k1, v1> -> map -> <k2, v2> -> combine -> <k2, v2> -> reduce -> <k3, v3> (output)


.. _example_wordcount_v1:

********************************
示例: WordCount v1.0
********************************

在深入学习 MapReduce 细节之前, 我们先通过一个 MapReduce 示例程序了解下他们是如何工作的。

WordCount 是一个简单的应用程序，它统计给定输入数据集中每个单词出现的次数。

它可以运行于单机模式、伪分布式模式或完全分布式模式下安装的 Hadoop(`单节点集群搭建 <http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html>`_)。


.. _example_wordcount_v1_source_code:

源代码
================================

.. code-block:: java

  import java.io.IOException;
  import java.util.StringTokenizer;

  import org.apache.hadoop.conf.Configuration;
  import org.apache.hadoop.fs.Path;
  import org.apache.hadoop.io.IntWritable;
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Job;
  import org.apache.hadoop.mapreduce.Mapper;
  import org.apache.hadoop.mapreduce.Reducer;
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

  public class WordCount {

    public static class TokenizerMapper
         extends Mapper<Object, Text, Text, IntWritable>{

      private final static IntWritable one = new IntWritable(1);
      private Text word = new Text();

      public void map(Object key, Text value, Context context
                      ) throws IOException, InterruptedException {
        StringTokenizer itr = new StringTokenizer(value.toString());
        while (itr.hasMoreTokens()) {
          word.set(itr.nextToken());
          context.write(word, one);
        }
      }
    }

    public static class IntSumReducer
         extends Reducer<Text,IntWritable,Text,IntWritable> {
      private IntWritable result = new IntWritable();

      public void reduce(Text key, Iterable<IntWritable> values,
                         Context context
                         ) throws IOException, InterruptedException {
        int sum = 0;
        for (IntWritable val : values) {
          sum += val.get();
        }
        result.set(sum);
        context.write(key, result);
      }
    }

    public static void main(String[] args) throws Exception {
      Configuration conf = new Configuration();
      Job job = Job.getInstance(conf, "word count");
      job.setJarByClass(WordCount.class);
      job.setMapperClass(TokenizerMapper.class);
      job.setCombinerClass(IntSumReducer.class);
      job.setReducerClass(IntSumReducer.class);
      job.setOutputKeyClass(Text.class);
      job.setOutputValueClass(IntWritable.class);
      FileInputFormat.addInputPath(job, new Path(args[0]));
      FileOutputFormat.setOutputPath(job, new Path(args[1]));
      System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
  }

.. _example_wordcount_v1_usage:

用法
================================

假设环境变量已经按照下面这样设置好:

.. code-block:: bash

  export JAVA_HOME=/usr/java/default
  export PATH=${JAVA_HOME}/bin:${PATH}
  export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar

编译 WordCount.java 并创建一个 jar 包:

.. code-block:: bash

  $ bin/hadoop com.sun.tools.javac.Main WordCount.java
  $ jar cf wc.jar WordCount*.class

假设:

  * /user/joe/wordcount/input - HDFS中的输入目录
  * /user/joe/wordcount/output - HDFS中的输出目录

将文本文件作为示例的输入:

.. code-block:: bash

  $ bin/hadoop fs -ls /user/joe/wordcount/input/
  /user/joe/wordcount/input/file01
  /user/joe/wordcount/input/file02

  $ bin/hadoop fs -cat /user/joe/wordcount/input/file01
  Hello World Bye World

  $ bin/hadoop fs -cat /user/joe/wordcount/input/file02
  Hello Hadoop Goodbye Hadoop

运行应用程序:

.. code-block:: bash

  $ bin/hadoop jar wc.jar WordCount /user/joe/wordcount/input /user/joe/wordcount/output

输出:

.. code-block:: bash

  $ bin/hadoop fs -cat /user/joe/wordcount/output/part-r-00000
  Bye 1
  Goodbye 1
  Hadoop 2
  Hello 2
  World 2

应用程序可以使用 -files 选项指定一个逗号分隔的路径列表，这些路径会出现在任务的当前工作目录中。-libjars 选项允许应用程序向 map 和 reduce 的 classpath 中添加 jar 包。-archives 选项允许传递逗号分隔的归档文件列表作为参数，这些归档文件会被解压并且在任务的当前工作目录下会创建一个符号链接(名称是归档文件名)。有关命令行选项的更多细节请参考 `命令指南 <http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/CommandsManual.html>`_ 。

下面使用 -libjars, -files 和 -archives 选项运行 wordcount 示例程序:

.. code-block:: bash

  bin/hadoop jar hadoop-mapreduce-examples-<ver>.jar wordcount -files cachefile.txt -libjars mylib.jar -archives myarchive.zip input output

现在, myarchive.zip 文件被解压到名为 "myarchive.zip" 的目录中。

用户可以使用 # 号为传递给 -files 和 -archives 选项的文件和归档文件指定一个不同的符号名称。

例如,

.. code-block:: bash

  bin/hadoop jar hadoop-mapreduce-examples-<ver>.jar wordcount -files dir1/dict.txt#dict1,dir2/dict.txt#dict2 -archives mytar.tgz#tgzdir input output

现在，任务可以分别使用符号名称 dict1 和 dict2 访问 dir1/dict.txt 和 dir2/dict.txt 文件。归档文件 mytar.tgz 将被解压到名为 "tgzdir" 的目录中。

.. _example_wordcount_v1_walk-through:

代码走读
================================

WordCount 应用程序非常简单。

.. code-block:: java

  public void map(Object key, Text value, Context context
                ) throws IOException, InterruptedException {
    StringTokenizer itr = new StringTokenizer(value.toString());
    while (itr.hasMoreTokens()) {
      word.set(itr.nextToken());
      context.write(word, one);
    }
  }

Mapper 实现中的 map 方法一次处理由指定 TextInputFormat 所提供的一行数据。然后，它通过 StringTokenizer 将该行分割成由空格分隔的 tokens，最后输出 < <word>, 1> 形式的一个键值对。

对于给定的示例输入，第一个 map 输出是：

.. code-block:: text

  < Hello, 1>
  < World, 1>
  < Bye, 1>
  < World, 1>

第二个 map 输出是：

.. code-block:: text

  < Hello, 1>
  < Hadoop, 1>
  < Goodbye, 1>
  < Hadoop, 1>

我们将在本教程的后续部分学习到更多关于给定作业产生的 map 数量以及如何以细粒度的方式去控制他们。

.. code-block:: java

  job.setCombinerClass(IntSumReducer.class);

WordCount 还指定了一个 combiner。因此，在对键进行排序后，每个 map 的输出传递给本地 combiner (与每个作业配置中的Reducer相同) 进行本地聚合。

第一个 map 的输出如下:

.. code-block:: text

  < Bye, 1>
  < Hello, 1>
  < World, 2>

第二个 map 的输出如下:

.. code-block:: text

  < Goodbye, 1>
  < Hadoop, 2>
  < Hello, 1>

.. code-block:: java

  public void reduce(Text key, Iterable<IntWritable> values,
                   Context context
                   ) throws IOException, InterruptedException {
    int sum = 0;
    for (IntWritable val : values) {
      sum += val.get();
    }
    result.set(sum);
    context.write(key, result);
  }


Reducer 实现中的 reduce 方法只是将每个 key (即本例中的单词) 出现的次数进行累加。

因此作业的输出是：

.. code-block:: text

  < Bye, 1>
  < Goodbye, 1>
  < Hadoop, 2>
  < Hello, 2>
  < World, 2>

main 方法在 Job 中指定了作业的各个方面，例如：输入/输出路径(通过命令行传递)、key/value 的类型、输入/输出的格式等等。然后调用 ``job.waitForCompletion`` 方法提交作业并监控作业进度。

我们将在本教程的后续部分学习更多关于 Job，InputFormat, OutputFormat 以及其他接口和类的相关知识。

.. _mapreduce_user_interfaces:

********************************
MapReduce - 用户接口
********************************

本节在 MapReduce 框架面向用户的每个方面提供了适量的细节。这应该可以帮助用户以细粒度的方式实现，配置以及调优他们的作业。但是需要注意的是，每个类/接口的 javadoc 才是最全面的文档，而这仅仅是一个教程。

让我们先来看看 Mapper 和 Reducer 接口。应用程序通常会实现这两个接口以提供 map 和 reduce 方法。

然后，我们会讨论其他核心接口，包括 Job, Partitioner, InputFormat, OutputFormat 等等。

最后，我们将讨论框架的一些实用功能，比如 DistributedCache，IsolationRunner 等等。

.. _payload:

Payload
================================

应用程序通常会实现 Mapper 和 Reducer 接口以提供 map 和 reduce 方法，它们组成了作业的核心。


.. _payload_mapper:

Mapper
--------------------------------

`Mapper <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Mapper.html>`_ 将输入的键值对映射成一组中间键值对。

map 是将输入记录转换为中间记录的单个任务。转换的中间记录不需要与输入记录具有相同的类型。一个给定的输入键值对可能映射到零个或多个输出键值对。

Hadoop MapReduce 框架为作业的 InputFormat 生成的每个 InputSplit 产生一个 map 任务。

总的来说，mapper 实现是通过 `Job.setMapperClass(Class) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 方法传递给作业的。然后，框架为任务的 InputSplit 中的每个键值对调用 `map(WritableComparable, Writable, Context) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Mapper.html>`_ 。最后，应用程序可以重写 ``cleanup(Context)`` 方法来执行任何所需的清理工作。

输出键值对不需要与输入键值对的类型相同。一个给定的输入键值对可能映射到零个或多个输出键值对。 输出键值对通过调用 ``context.write(WritableComparable, Writable)`` 方法进行收集。

应用程序可以使用 Counter 报告其统计数据。

所有与给定输出键相关的中间值随后由框架进行分组并传递给 Reducer 以确定最终输出。用户可以通过 `Job.setGroupingComparatorClass(Class) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 指定一个比较器来控制分组。

Mapper 输出会被排序，然后按 Reducer 进行分区。分区总数与作业的 reduce 任务数相同。用户可以实现一个自定义的分区控制器来控制哪些键(以及记录)分发到哪个Reducer。

用户可以选择通过 `Job.setCombinerClass(Class) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 指定一个 combiner 来执行中间输出的本地聚合, 这样可以减少从 Mapper 到 Reducer 传输的数据量。

中间的排序输出总是以简单的(key-len，key，value-len，value)格式存储。应用程序可以通过 Configuration 控制是否以及如何对中间输出进行压缩以及如何使用 `CompressionCodec <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/io/compress/CompressionCodec.html>`_。


Map数量
^^^^^^^^^^^^^^^^^^^^^^^^^^

map 的数量通常是由输入数据的总大小决定的，也就是输入文件的总块数。

map 正确的并行度大概是每个节点 10-100 个 map，尽管有些 CPU 轻量型 map 任务已经将其设置为 300个 map。任务设置需要一段时间，所以最好是 map 任务至少需要一分钟才能执行。

因此，如果您希望输入 10TB 数据，并且块大小为 128MB，那么最终将有 82,000 个 map，除非用 ``Configuration.set(MRJobConfig.NUM_MAPS, int)`` (仅向框架提供提示)进行设置, 否则 map 数量会更高。

.. _payload_reducer:

Reducer
--------------------------------

`Reducer <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Reducer.html>`_ 将共享一个key的一组中间值归并为一个小的数值集。

用户可以通过 `Job.setNumReduceTasks(int) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 来设置作业的 reduce 数量。

总的来说，Reducer 实现通过 `Job.setReducerClass(Class) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 方法传递给作业，并可以重写它以初始化它们自己。然后，框架为分组输入中的每个 ``<key, (list of values)>`` 对调用 `reduce(WritableComparable, Iterable, Context) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Reducer.html>`_ 方法。最后，应用程序可以重写 ``cleanup(Context)`` 方法来执行任何所需的清理工作。

Reducer 有3个主要阶段: shuffle, sort 和 reduce。

Shuffle
^^^^^^^^^^^^^^^^^^^^^^^^^^

Reducer 的输入就是 mapper 的排序输出。在这个阶段，框架通过 HTTP 获取所有 mapper 输出的相关分区。

Sort
^^^^^^^^^^^^^^^^^^^^^^^^^^

在这个阶段中，框架将按照 key (因为不同 mapper 的输出中可能会有相同的 key) 对 Reducer 的输入进行分组。

shuffle 和 sort 两个阶段是同时发生的；map 的输出一遍取出一边被合并。

Secondary Sort
^^^^^^^^^^^^^^^^^^^^^^^^^^

如果用于分组中间键的等价规则需要区别于reduce之前用于分组键的等价规则, 那么可以通过 `Job.setSortComparatorClass(Class) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 指定一个比较器。由于可以使用 `Job.setGroupingComparatorClass(Class) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 来控制中间键的分组方式，因此可以结合使用它们来模拟次要排序值。


Reduce
^^^^^^^^^^^^^^^^^^^^^^^^^^

在这个阶段中, 会为分组输入中的每个 ``<key, (list of values)>`` 对调用 ``reduce(WritableComparable, Iterable<Writable>, Context)`` 方法。

reduce 任务的输出通常是通过 ``Context.write(WritableComparable, Writable)`` 写入 `文件系统 <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/fs/FileSystem.html>`_ 的。

应用程序可以使用 Counter 来报告其统计数据。

Reducer 的输出是没有排过序的。

Reduce数量
^^^^^^^^^^^^^^^^^^^^^^^^^^

reduce 数量建议是 0.95 或 1.75 乘以 (节点数量 * 每个节点最大容器数量)。

使用 0.95, 所有的 reduce 任务可以在 map 任务一完成时就立即启动并开始传输 map输出。如果使用 1.75，更快的节点将完成第一轮 reduce 任务并启动第二轮 reduce 任务，这样可以得到比较好的负载均衡的效果。

增加 reduce 任务数量会增加框架的开销，但可以提升负载均衡并降低失败成本。

上述比例因子比整体数目稍小一些，主要是为了给框架中的推测性任务或失败的任务预留一些 reduce 资源。

Reducer NONE
^^^^^^^^^^^^^^^^^^^^^^^^^^

如果没有必要进行 reduce 操作，则可以将reduce 任务数设置为0。

在这种情况下，map 任务的输出结果直接写入文件系统上由 `FileOutputFormat.setOutputPath(Job, Path) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/output/FileOutputFormat.html>`_ 指定的输出路径中。在将它们写出到文件系统之前，框架不会对 map 输出进行排序。

.. _payload_partitioner:

Partitioner
--------------------------------

`Partitioner <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Partitioner.html>`_ 对键空间进行分区。

Partitioner 负责控制 map 中间输出结果的键的分区。键(或者键的子集)用于产生分区，通常通过一个散列函数。分区总数与作业的 reduce 任务数是一样的。因此，它控制中间输出结果(也就是这条记录)的键发送给 m 个 reduce 任务中的哪一个来进行 reduce 操作。

`HashPartitioner <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/partition/HashPartitioner.html>`_ 是默认的 ``Partitioner``。

.. _payload_counter:

Counter
--------------------------------

`Counter <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Counter.html>`_ 是 MapReduce 应用程序报告其统计数据的一个工具。

Mapper 和 Reducer 实现可以使用 Counter 报告统计数据。

Hadoop MapReduce 附带一个包含通用的 mapper, reducers 以及 partitioners 的 `类库 <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/package-summary.html>`_ 。

.. _job_configuration:

作业配置
================================

`Job <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 代表一个 MapReduce 作业的配置。

Job 是用户向 Hadoop 框架描述一个 MapReduce 作业如何执行的主要接口。框架会按照 Job 的描述执行作业, 然而:

* 一些配置参数可能被管理员标记成了 final (参见 `Final 参数 <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/conf/Configuration.html#FinalParams>`_ )，因此它们不能被修改。
* 虽然有些作业参数可以直接进行设置 (比如 `Job.setNumReduceTasks(int) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ ), 但是另外一些微妙地影响着框架和/或作业配置的其余部分的参数设置起来就比较复杂(比如 `Configuration.set(JobContext.NUM_MAPS, int) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/conf/Configuration.html>`_)。

作业通常用于指定 Mapper、组合器(如果有的话)，Partitioner、Reducer、InputFormat 以及 OutputFormat 的实现。`FileInputFormat <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/input/FileInputFormat.html>`_ 表示一组输入文件 (`FileInputFormat.setInputPaths(Job, Path…) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/input/FileInputFormat.html>`_ / `FileInputFormat.addInputPath(Job, Path) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/input/FileInputFormat.html>`_ ) 和 (`FileInputFormat.setInputPaths(Job, String…) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/input/FileInputFormat.html>`_ ) / `FileInputFormat.addInputPaths(Job, String) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/input/FileInputFormat.html>`_) 以及输出文件应该写入到哪儿 (`FileOutputFormat.setOutputPath(Path) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/input/FileOutputFormat.html>`_ )。

可以选择使用Job来指定作业的其他高级方面，例如要使用的比较器，要放入 DistributedCache 的文件，是否(以及如何)压缩中间和/或作业输出，作业任务是否可以以推测的方式执行(`setMapSpeculativeExecution(boolean) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ / `setReduceSpeculativeExecution(boolean) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_)，每个任务的最大尝试次数 (`setMaxMapAttempts(int) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ / `setMaxReduceAttempts(int) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_) 等。

当然, 用户可以使用 `Configuration.set(String, String) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/conf/Configuration.html>`_ / `Configuration.get(String) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/conf/Configuration.html>`_ 来设置/获取应用程序所需的任意参数。但是对于大规模只读数据，请使用 ``DistributedCache``。


.. _task_execution_and_environment:

任务执行和环境
================================

MRAppMaster 在一个独立的 jvm 中作为子进程执行 Mapper/Reducer 任务。

子任务继承父 MRAppMaster 的执行环境。用户可以通过 ``mapreduce.{map|reduce}.java.opts`` 和 Job 配置参数来为子jvm指定额外的选项，比如通过 -Djava.library.path=<> 选项指定运行时链接程序搜索共享类库的非标准路径等。如果 ``mapreduce.{map|reduce}.java.opts`` 参数包含符号 @taskid@，它将插入 MapReduce 任务的任务ID的值。

下面是一个包含多个参数和替换的例子，显示了jvm GC日志记录，并启动了无密码JVM JMX代理，以便它可以与 jconsole 连接，并且可以观察子内存，线程和线程转储。 它还设置地图的最大堆大小，并分别将子jvm减少到512MB和1024MB。 它还为 child-jvm 的 java.library.path 添加了一个额外的路径。

.. code-block:: xml

  <property>
    <name>mapreduce.map.java.opts</name>
    <value>
    -Xmx512M -Djava.library.path=/home/mycompany/lib -verbose:gc -Xloggc:/tmp/@taskid@.gc
    -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false
    </value>
  </property>

  <property>
    <name>mapreduce.reduce.java.opts</name>
    <value>
    -Xmx1024M -Djava.library.path=/home/mycompany/lib -verbose:gc -Xloggc:/tmp/@taskid@.gc
    -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false
    </value>
  </property>


.. _memory_management:

内存管理
--------------------------------

用户/管理员还可以递归地使用 ``mapreduce.{map|reduce}.memory.mb`` 参数指定启动的子任务以及它启动的任何子进程的最大虚拟内存。请注意，此处设置的值是针对每个进程的限制。``mapreduce.{map|reduce}.memory.mb`` 参数指定的值应以兆字节(MB)为单位, 而且该值必须大于或等于传递给 JavaVM 的 -Xmx 选项的值，否则虚拟机可能无法启动。

.. note:: ``mapreduce.{map|reduce}.java.opts`` 仅用于配置从 MRAppMaster 中启动的子任务。配置守护进程的内存选项参见文档：`配置Hadoop守护进程的环境变量 <http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/ClusterSetup.html#Configuring_Environment_of_Hadoop_Daemons>`_。

框架某些部分的可用内存也是可配置的。在 map 和 reduce 任务中, 性能可能会受到影响操作并发性和数据写到磁盘频率的参数调整的影响。监控作业的文件系统计数器，特别是有关 map 端到 reduce 端的字节计数，对于这些参数的调优是非常有用的。


.. _map_parameters:

Map 参数
--------------------------------

从 map 端输出的记录将被序列化到一个缓冲区中，并且记录的元数据将被存储到记帐缓冲区中。如以下选项所述，当序列化缓冲区或元数据超过阈值时，缓冲区的内容将在后台进行排序并写入到磁盘上，而 map 端继续输出记录。如果任何一个缓冲区在分隔过程中被完全填满，则 map 线程被阻塞。 当 map 完成后，剩余的任何记录都将写入磁盘并且所有磁盘上的段会合并到一个文件中。 尽量减少写到磁盘的分片数量可以缩短 map 时间，但是更大的缓冲区也会减少 mapper 的可用内存。

====================================      ================      ================
参数名称                                   参数类型               参数说明
====================================      ================      ================
mapreduce.task.io.sort.mb                 int                   存储从 map 端输出记录的序列化和记帐缓冲区的总大小, 以兆字节(MB)为单位。
mapreduce.map.sort.spill.percent          float                 序列化缓冲区的软限制。一旦到达该阈值，一个后台线程开始将内容写到磁盘中。
====================================      ================      ================

其它说明

* 如果溢出过程中超过溢出阈值，则收集将继续直到溢出完成。例如，如果 ``mapreduce.map.sort.spill.percent`` 设置为 0.33，并且在溢出运行时填充其余的缓冲区，则下一次溢出将包括所有收集的记录或缓冲区的0.66，并且不会产生额外的溢出。 换句话说，阈值是定义触发器的，而不是阻塞。
* 如果记录大小超过了序列化缓冲区的大小, 首先会触发一个分割操作, 然后写到一个单独的文件中。没有定义记录是否需要先通过 combiner。


.. _shuffle_reduce_parameters:

Shuffle/Reduce 参数
--------------------------------

如前面所述，每个 reduce 会将分区程序分配给它的输出通过 HTTP 提取到内存中，并且定期将这些输出合并到磁盘。如果 map 输出的中间压缩打开，则每个输出都会解压缩到内存中。 以下选项影响在 reduce 期间这些合并到磁盘的频率和分配给映射输出的内存的频率。

==================================================      ======================      ======================
参数名称                                                 参数类型                     参数说明
==================================================      ======================      ======================
mapreduce.task.io.soft.factor                           int                         Specifies the number of segments on disk to be merged at the same time. It limits the number of open files and compression codecs during merge. If the number of files exceeds this limit, the merge will proceed in several passes. Though this limit also applies to the map, most jobs should be configured so that hitting this limit is unlikely there.
mapreduce.reduce.merge.inmem.thresholds                 int                         The number of sorted map outputs fetched into memory before being merged to disk. Like the spill thresholds in the preceding note, this is not defining a unit of partition, but a trigger. In practice, this is usually set very high (1000) or disabled (0), since merging in-memory segments is often less expensive than merging from disk (see notes following this table). This threshold influences only the frequency of in-memory merges during the shuffle.
mapreduce.reduce.shuffle.merge.percent                  float                       The memory threshold for fetched map outputs before an in-memory merge is started, expressed as a percentage of memory allocated to storing map outputs in memory. Since map outputs that can’t fit in memory can be stalled, setting this high may decrease parallelism between the fetch and merge. Conversely, values as high as 1.0 have been effective for reduces whose input can fit entirely in memory. This parameter influences only the frequency of in-memory merges during the shuffle.
mapreduce.reduce.shuffle.input.buffer.percent           float                       The percentage of memory- relative to the maximum heapsize as typically specified in mapreduce.reduce.java.opts- that can be allocated to storing map outputs during the shuffle. Though some memory should be set aside for the framework, in general it is advantageous to set this high enough to store large and numerous map outputs.
mapreduce.reduce.input.buffer.percent                   float                       The percentage of memory relative to the maximum heapsize in which map outputs may be retained during the reduce. When the reduce begins, map outputs will be merged to disk until those that remain are under the resource limit this defines. By default, all map outputs are merged to disk before the reduce begins to maximize the memory available to the reduce. For less memory-intensive reduces, this should be increased to avoid trips to disk.
==================================================      ======================      ======================


其它说明

* If a map output is larger than 25 percent of the memory allocated to copying map outputs, it will be written directly to disk without first staging through memory.
* When running with a combiner, the reasoning about high merge thresholds and large buffers may not hold. For merges started before all map outputs have been fetched, the combiner is run while spilling to disk. In some cases, one can obtain better reduce times by spending resources combining map outputs- making disk spills small and parallelizing spilling and fetching- rather than aggressively increasing buffer sizes.
* When merging in-memory map outputs to disk to begin the reduce, if an intermediate merge is necessary because there are segments to spill and at least mapreduce.task.io.sort.factor segments already on disk, the in-memory map outputs will be part of the intermediate merge.


.. _configured_parameters:

配置参数
--------------------------------

以下属性在每个任务执行的作业配置中进行了本地化:

================================      ========================      ========================
参数名称                               参数类型                       参数说明
================================      ========================      ========================
mapreduce.job.id                      String                        作业ID
mapreduce.job.jar                     String                        作业目录下 job.jar 的路径
mapreduce.job.local.dir               String                        作业特有的共享临时空间
mapreduce.task.id                     String                        任务ID
mapreduce.task.attempt.id             String                        任务尝试ID
mapreduce.task.is.map                 boolean                       是否是 map 任务
mapreduce.task.partition              int                           作业内的任务ID
mapreduce.map.input.file              String                        map 端读取数据的文件名
mapreduce.map.input.start             long                          map 端输入分片起始偏移量
mapreduce.map.input.length            long                          map 端输入分片的字节大小
mapreduce.task.output.dir             String                        任务临时输出目录
================================      ========================      ========================

.. attention:: 在 streaming 作业执行期间, 以 mapreduce 开头的参数名称会被转换。点号( . )变成了下划线( _ )。例如, mapreduce.job.id 变成 mapreduce_job_id，mapreduce.job.jar 变成 mapreduce_job_jar。要想在 streaming 作业的 mapper/reducer 过程中获取参数值，请使用带有下划线的参数名称。


.. _task_logs:

任务日志
--------------------------------

标准输出(stdout)和错误(stderr)流以及任务的系统日志由 NodeManager 读取并记录到 ``${HADOOP_LOG_DIR}/userlogs`` 中。


.. _distributing_libraries:

分发类库
--------------------------------

`DistributedCache <http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#DistributedCache>`_ 还可以用来分发 map 和/或 reduce 任务中要用的 jar 包以及本地类库。child-jvm 总是将其当前工作目录添加到 java.library.path 和 LD_LIBRARY_PATH 中。因此缓存的类库可以通过 `System.loadLibrary <http://docs.oracle.com/javase/7/docs/api/java/lang/System.html>`_ 或 `System.load <http://docs.oracle.com/javase/7/docs/api/java/lang/System.html>`_ 加载。更多关于如何通过分布式缓存加载共享类库的细节参见 `Native Libraries <http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/NativeLibraries.html#Native_Shared_Libraries>`_ 文档。

.. _job_submission_and_monitoring:

作业提交与监控
================================

`Job <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 是用户作业和 ResourceManager 进行交互的主要接口。

Job 提供了用于提交作业、跟踪作业进度、访问组件任务的报告和日志以及获取 MapReduce 集群状态信息等工具。

作业提交过程包括:

1. 检查作业的输入和输出是否规范。
2. 为作业计算 InputSplit 值。
3. 如果需要的话，为作业的 DistributedCache 设置必要的统计信息。
4. 拷贝作业的 jar 包和配置文件到文件系统上的 MapReduce 系统目录下。
5. 提交作业到 ResourceManager 并有选择性地监控作业状态。

作业的历史记录文件也会记录到用户指定的目录，这个目录由 ``mapreduce.jobhistory.intermediate-done-dir`` 和 ``mapreduce.jobhistory.done-dir`` 参数设定，默认值是作业的输出目录。

用户可以使用以下命令查看指定目录中的历史日志摘要:

.. code-block:: bash

  $ mapred job -history output.jhist

该命令将打印作业详细信息，失败及终止的提示信息。有关作业的更多详细信息，比如成功的任务和为每个任务所做的尝试都可以使用以下命令查看:

.. code-block:: bash

  $ mapred job -history all output.jhist

通常，用户使用 Job 创建应用程序，描述作业的各个方面，提交作业并监控其进度。


.. _job_control:

作业控制
--------------------------------

用户可能需要链接多个 MapReduce 作业来完成无法通过单个 MapReduce 作业完成的复杂任务。这样做非常简单，因为作业的输出通常会进入到分布式文件系统，而输出反过来又可以用作下一个作业的输入。

然而，这也意味着确保作业完成(成功/失败)的责任就直接落在了客户身上。在这种情况下，可用的作业控制选项有:

* `Job.submit() <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ : 提交作业到集群并立即返回。
* `Job.waitForCompletion(boolean) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ : 提交作业到集群并等待作业完成。


.. _job_input:

作业输入
================================

`InputFormat <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/InputFormat.html>`_ 描述了一个 MapReduce 作业的输入规范。

MapReduce 框架依赖作业的 InputFormat 来:

#. 确认作业的输入规范。
#. 把输入文件分割成多个逻辑的 InputSplit 实例，然后将每个实例分配给一个单独的 Mapper。
#. 提供 RecordReader 的实现，用于从逻辑 InputSplit 中读取输入记录以供 Mapper 处理。

基于文件的 InputFormat 实现(通常是 `FileInputFormat <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/input/FileInputFormat.html>`_ 的子类) 的默认行为是是根据输入文件的总字节大小将输入分割成逻辑 InputSplit 实例。但是，输入文件的文件系统块大小被视为输入分割的上限。分割大小的下限可以通过 mapreduce.input.fileinputformat.split.minsize 参数来设置。

按照输入文件大小进行逻辑分割对于很多应用程序来说显然是不够的，因为我们必须要考虑记录边界。在这种情况下，应用程序需要实现一个 RecordReader 来负责处理记录边界，以及为单个任务提供逻辑 InputSplit 的一个面向记录的视图。

`TextInputFormat <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/input/TextInputFormat.html>`_ 是默认的 InputFormat。

如果一个给定作业的 Inputformat 是 TextInputFormat，则框架会检测带有 .gz 后缀的输入文件并使用合适的 CompressionCodec 自动解压缩这些文件。但是需要注意的是，带有上述扩展名的压缩文件不会被切分，并且每个压缩文件会被一个 mapper 作为一个整体来处理。


.. _input_split:

InputSplit
--------------------------------

`InputSplit <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/InputSplit.html>`_ 表示单个 Mapper 要处理的数据。

通常 InputSplit 提供一个面向字节的输入视图，RecordReader 负责处理并转换成一个面向记录的视图。

`FileSplit <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/input/FileSplit.html>`_ 是默认的 InputSplit。它将 ``mapreduce.map.input.file`` 参数设置为逻辑分割的输入文件的路径。


.. _record_reader:

RecordReader
--------------------------------

`RecordReader <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/RecordReader.html>`_ 从 InputSlit 读取 <key, value> 对。

通常 RecordReader 会把 InputSplit 提供的面向字节的输入视图转换成一个面向记录的视图并呈现给 Mapper 的实现进行处理。 因此 RecordReader 负责处理记录边界并将使用键和值表示任务。

.. _job_output:

作业输出
================================

`OutputFormat <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/OutputFormat.html>`_ 描述了一个 MapReduce 作业的输出规范。

MapReduce 框架依赖作业的 OutputFormat 来:

1. 确认作业的输出规范; 例如检查输出路径是否已经存在。
2. 提供用于写入作业输出文件的 RecordWriter 实现。输出文件存储在文件系统上。

TextOutputFormat 是默认的 OutputFormat。


.. _output_committer:

OutputCommitter
--------------------------------

`OutputCommitter <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/OutputCommitter.html>`_ 描述了一个 MapReduce 作业提交任务输出。

MapReduce 框架依赖作业的 OutputCommitter 来:

#. 在初始化期间设置作业。例如，在作业初始化期间创建临时输出目录。当作业处于 PREP 状态并且初始化任务后, 作业设置又一个单独的任务完成。一旦设置任务完成, 作业将变成 RUNNING 状态。
#. 作业完成后清理作业。例如在任务完成后删除临时输出目录。作业清理是在作业结束时由单独的任务完成的。在清理任务完成后，作业被声明为 SUCCEDED/FAILED/KILLED。
#. 设置任务临时输出。任务设置是在任务初始化期间作为同一任务的一部分完成的。
#. 检查任务是否需要一个提交。这是为了避免提交过程，如果任务不需要提交的话。
#. 提交任务输出。一旦任务完成, 如有必要任务将会提交它的输出。`
#. 放弃任务提交。如果任务失败或终止, 任务输出将会被清理。如果任务无法清理(在异常块中)，则启动带有相同attempt-id的单独任务来执行清理。

FileOutputCommitter 是默认的 OutputCommitter。作业设置/清理任务占用 map 或 reduce 容器, 无论 NodeManager 上哪个可用。JobCleanup 任务, TaskCleanup 任务 和 JobSetup 任务按顺序具有最高优先级。


.. _task_side_effect_files:

任务Side-Effect文件
--------------------------------

在某些应用程序中，组件任务需要创建 和/或 写入与实际作业输出文件不同的副文件中。

在这种情况下, 如果同一个 Mapper 或者 Reducer 同时运行的两个实例(例如推测任务)试图打开和/或写入文件系统上的同一个文件(路径)就可能会有问题。因此应用程序编写者需要为每个任务尝试取一个独一无二的文件名(使用 attemptid，比如 attempt_200709221812_0001_m_000000_0), 而不仅仅是每个任务。

为了避免这些问题, 当 OutputCommitter 是 FileOutputCommitter 时, MapReduce 框架在文件系统上为每个任务尝试维护了一个特殊的 ``${mapreduce.output.fileoutputformat.outputdir}/_temporary/_${taskid}`` 子目录来存储任务尝试的输出，该目录可以通过 ``${mapreduce.task.output.dir}`` 访问。在任务尝试成功执行完成时, ``${mapreduce.output.fileoutputformat.outputdir}/_temporary/_${taskid}`` 中的文件(仅)会被移动到 ``${mapreduce.output.fileoutputformat.outputdir}`` 目录下。当然，框架会丢弃那些失败任务尝试的子目录。这个过程对于应用程序来说是完全透明的。

应用程序编写者可以在任务执行期间通过 `FileOutputFormat.getWorkOutputPath(Context) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/output/FileOutputFormat.html>`_ 在 ``${mapreduce.task.output.dir}`` 中创建任意需要的副文件来利用该功能，并且同样地对于成功的尝试框架会移动这些文件，因此不需要为每个任务尝试选取唯一路径。

.. note:: 在特定任务尝试执行期间，``${mapreduce.task.output.dir}`` 的值实际上是 ``${mapreduce.output.fileoutputformat.outputdir}/_temporary/_{$taskid}``, 并且这个值是由 MapReduce 框架设置的。因此, 只需要从 MapReduce 任务的 `FileOutputFormat.getWorkOutputPath(Context) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/output/FileOutputFormat.html>`_ 返回路径中创建任意副文件即可充分利用该功能。

整个讨论适用于 reducer=NONE (即 0 个 reduce) 的作业，因为在这种情况下，map 的输出直接写到 HDFS。


.. _record_writer:

RecordWriter
--------------------------------

`RecordWriter <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/RecordWriter.html>`_ 将生成的 <key, value> 对写到输出文件中。

RecordWriter 的实现把作业的输出结果写到文件系统。

.. _other_useful_features:

其他有用的特性
================================

.. _submitting_jobs_to_queues:

提交作业到队列
--------------------------------

用户将作业提交到队列。队列，作为作业集合，允许系统提供特定的功能。 例如，队列使用 ACL 来控制哪些用户可以向他们提交作业。 预计队列将主要由 Hadoop 调度程序使用。

Hadoop 配置了一个名为 'default' 的强制队列。队列名称在 Hadoop 站点配置的 ``mapreduce.job.queuename`` 属性中定义。一些作业调度程序，如 `Capacity Scheduler <http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html>`_ 支持多个队列。

作业通过 ``mapreduce.job.queuename`` 属性或通过 ``Configuration.set(MRJobConfig.QUEUE_NAME，String)`` API 定义其需要提交到的队列。设置队列名是可选的。如果提交的作业没有设置相应的队列名称，则其会提交到 'default' 队列。


.. _counters:

Counters
--------------------------------

Counters 表示由 MapReduce 框架或者应用程序定义的全局计数器。每个计数器可以是任意的枚举类型。同一特定枚举类型的多个计数器可以划分到类型为 Counters.Group 的分组里面。

应用程序可以定义任意数量的计数器(Enum类型)并且可以在 map 和/或 reduce 方法中通过 `Counters.incrCounter(Enum, long) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/Counters.html>`_ 或 ``Counters.incrCounter(String, String, long)`` 方法更新他们。接着框架会对这些计数器进行全局聚合。


.. _distributed_cache:

DistributedCache
--------------------------------

DistributedCache 高效地分发特定于应用程序的大型只读文件。

DistributedCache 是 MapReduce 框架提供的用于缓存应用程序所需文件(包括文本文件，归档文件，jar文件等等)的一个工具。

应用程序在 Job 中通过 url(hdfs://) 指定需要被缓存的文件。DistributedCache 假定由 hdfs:// url 指定的文件已经存在于文件系统中。

作业的任何任务在节点上执行之前，框架会将所有必要文件拷贝到工作节点上。它的高效源自这样一个事实，即每个作业只复制一次文件，并且能够缓存工作节点上解压的归档文件。

DistributedCache 会跟踪缓存文件的修改时间戳。显然在作业执行期间, 缓存文件不应该由应用程序或外部修改。

DistributedCache 可以分发简单的只读数据/文本文件，也可以分发复杂类型的文件，例如归档文件和 jar 文件。归档文件(zip，tar，tgz 和 tar.gz 文件)在工作节点上被解压。文件设置有执行权限。

文件/归档文件可以通过设置属性 ``mapreduce.job.cache.{files | archives}`` 进行分发。如果需要分发多个文件/归档文件，可以使用逗号分隔文件路径。这些属性还可以通过API ``Job.addCacheFile(URI)`` / ``Job.addCacheArchive(URI)`` 和 ``Job.setCacheFiles(URI [])`` / ``Job.setCacheArchives(URI [])`` 来设置，其中 URI 的格式为 hdfs://host:port/absolute-path#link-name。在 Streaming 程序中，可以通过命令行选项 -cacheFile /-cacheArchive 分发文件。

DistributedCache 还可以在 map 和/或 reduce 任务中作为一种基础软件分发机制使用。它可以用来分发 jar 文件和本地库。``Job.addArchiveToClassPath(Path)`` 或 ``Job.addFileToClassPath(Path)`` API 可用于缓存文件/jar文件并且将它们添加到 child-jvm 的类路径中。通过设置配置属性 ``mapreduce.job.classpath.{files | archives}`` 也可以实现同样的效果。同样地，链接到任务工作目录中的缓存文件可以用来分发本地库并加载它们。


Private 和 Public DistributedCache 文件
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

DistributedCache 文件可以是 private 或 public, 这决定了它们如何在工作节点上共享。

* "Private" DistributedCache 文件缓存在一个本地目录中, 对于作业需要这些文件的用户来说是私有的。这些文件仅由特定用户的所有任务和作业共享，并且不能被工作节点上其他用户的作业访问。 DistributedCache 文件由于其上传文件所处文件系统(通常是 HDFS)上的权限而变为 private。如果文件没有全局可读访问权限, 或者文件所在目录路径对于查找没有全局可执行访问权限, 那么文件变成 private。
* "Public" DistributedCache 文件缓存在一个全局目录中，并且设置的文件访问权限可以使它们对所有用户公开可见。这些文件可以被工作节点上所有用户的任务和作业所共享。DistributedCache 文件由于其上传文件所处文件系统(通常是 HDFS)上的权限而变为 public。如果文件有全局可读访问权限, 并且文件所在目录路径对于查找具有全局可执行访问权限, 那么文件变成 public。换句话说, 如果用户打算让文件对所有用户公开可见，则必须将文件权限设置为全局可读的，并且文件所在路径上的目录权限必须是全局可执行的。

用户作业需要这些文件


.. _profiling:

性能分析
--------------------------------

性能分析是一个实用工具，用于获取内置的 Java 分析器关于 map 和 reduce 运行样本中有代表性的(2个或3个)样本。

通过设置配置属性 ``mapreduce.task.profile``，用户可以指定系统是否采集作业中某些任务的性能分析信息。属性值可以使用 ``Configuration.set(MRJobConfig.TASK_PROFILE, boolean)`` API 进行设置。 如果该值设置为 true，则启用任务性能分析, 分析信息存储在用户日志目录中。默认情况下，作业不启用性能分析。

一旦用户配置需要性能分析, 她/他就可以使用配置属性 ``mapreduce.task.profile.{maps|reduces}`` 设置要分析的 MapReduce 任务范围。属性值可以使用 ``Configuration.set(MRJobConfig.NUM_{MAP|REDUCE}_PROFILES, String)`` API 进行设置。指定范围的缺省值是 0-2。

用户也可以通过设置 ``mapreduce.task.profile.params`` 配置属性来指定分析器配置参数。属性值可以使用 ``Configuration.set(MRJobConfig.TASK_PROFILE_PARAMS, String)`` API 指定。如果字符串包含 %s, 在任务运行时它将被替换成性能分析输出文件名。这些参数通过命令行传递给任务子 JVM。性能分析参数的缺省值是 ``-agentlib:hprof=cpu=samples,heap=sites,force=n,thread=y,verbose=n,file=%s``。


.. _debugging:

调试
--------------------------------

MapReduce 框架提供了一个工具来运行用户提供的用于调试的脚本。当 MapReduce 任务失败时, 用户可以运行一个调试脚本，例如处理任务日志。脚本可以访问任务的 stdout 和 stderr 输出, syslog 和 jobconf。调试脚本的 stdout 和stderr 输出显示在控制台诊断中，同时也作为作业 UI 的一部分。

在接下来的章节中，我们讨论如何提交作业的调试脚本。脚本文件需要分发并提交给框架。


如何分发脚本文件
^^^^^^^^^^^^^^^^^^^^^^^^^^^

用户需要使用 `DistributedCache <http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#DistributedCache>`_ 来分发和链接脚本文件。


如何提交脚本
^^^^^^^^^^^^^^^^^^^^^^^^^^^

提交调试脚本的一个快速方法是分别为需要调试的 map 任务和 reduce 任务设置 ``mapreduce.map.debug.script`` 和 ``mapreduce.reduce.debug.script`` 属性值。这些属性也可以通过使用 `Configuration.set(MRJobConfig.MAP_DEBUG_SCRIPT, String) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/conf/Configuration.html>`_ 和 `Configuration.set(MRJobConfig.REDUCE_DEBUG_SCRIPT, String) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/conf/Configuration.html>`_ API 来设置。在 streaming 模式中，可以使用命令行选项 -mapdebug 和 -reducedegug 来提交调试脚本, 分别用于调试 map 任务和 reduce 任务。

脚本的参数是任务的 stdout, stderr, syslog 以及 jobconf 文件。在 MapReduce 任务失败的节点上运行的调试命令是:

.. code-block:: bash

  $script $stdout $stderr $syslog $jobconf

管道程序将 c++ 程序名称作为命令的第五个参数。因此对于管道程序来说, 命令是:

.. code-block:: bash

  $script $stdout $stderr $syslog $jobconf $program


默认行为
^^^^^^^^^^^^^^^^^^^^^^^^^^^

对于管道，运行默认脚本来处理 gdb 下的核心转储，打印堆栈跟踪信息以及提供运行线程的相关信息。

.. _data_compression:

数据压缩
--------------------------------

Hadoop MapReduce 框架为应用程序编写器提供了用于指定 map 中间输出和作业输出(如 reduce 输出) 压缩算法的工具。它还捆绑了 `zlib <http://www.zlib.net/>`_ 压缩算法的 `CompressionCodec <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/io/compress/CompressionCodec.html>`_ 实现。同时也支持 `gzip <http://www.gzip.org/>`_ , `bzip2 <http://www.bzip.org/>`_ , `snappy <http://code.google.com/p/snappy/>`_ 以及 `lz4 <http://code.google.com/p/lz4/>`_ 文件格式。

考虑到性能(zlib)和 Java 类库的不可用性, Hadoop 也为上述压缩编解码器提供了本地实现。有关它们的用法和可用性的更多细节参见`这里 <http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/NativeLibraries.html>`_ 。


中间输出
^^^^^^^^^^^^^^^^^^^^^^^^^^^

应用程序可以通过 ``Configuration.set(MRJobConfig.MAP_OUTPUT_COMPRESS, boolean)`` API 控制 map 的中间输出是否需要压缩并且通过 ``Configuration.set(MRJobConfig.MAP_OUTPUT_COMPRESS_CODEC, Class)`` API 指定需要使用的 CompressionCodec。


作业输出
^^^^^^^^^^^^^^^^^^^^^^^^^^^

应用程序可以通过 `FileOutputFormat.setCompressOutput(Job, boolean) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/output/FileOutputFormat.html>`_  API 控制作业输出是否需要进行压缩并且通过 ``FileOutputFormat.setOutputCompressorClass(Job, Class)`` API 指定需要使用的 CompressionCodec。

如果作业输出要存储在 `SequenceFileOutputFormat <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/output/SequenceFileOutputFormat.html>`_ 中, 需要通过 ``SequenceFileOutputFormat.setOutputCompressionType(Job, SequenceFile.CompressionType)`` API 指定需要的 ``SequenceFile.CompressionType`` (例如 RECORD / BLOCK， 默认值是 RECORD)。

.. _skipping_bad_records:

跳过脏数据
--------------------------------

Hadoop 提供了一个选项，可以在处理 map 端输入时跳过某些脏数据。 应用程序可以通过 `SkipBadRecords <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/SkipBadRecords.html>`_ 类来控制该功能。

当 map 任务在某些输入上确定性地崩溃时，可以使用该功能。这通常是由于 map 函数中的 bug 而发生的。通常用户不得不修复这些 bug。但是，有时候这是不可能的。bug 可能存在于第三方库中，例如源代码不可用。在这种情况下，即使经过很多次尝试，任务也不会成功地完成，并且作业会失败。使用此功能后，只有脏记录周围一小部分数据会丢失，对于某些应用程序(例如对超大型数据执行统计分析的应用程序)来说是可以接受的。

默认情况下，该功能被禁用。要想启用它，请参阅 `SkipBadRecords.setMapperMaxSkipRecords(Configuration, long) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/SkipBadRecords.html>`_ 和 `SkipBadRecords.setReducerMaxSkipGroups(Configuration, long) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/SkipBadRecords.html>`_ 。

启用此功能后，框架会在一定数量的 map 失败后进入"跳过模式"。更多细节，请参阅 `SkipBadRecords.setAttemptsToStartSkipping(Configuration，int) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/SkipBadRecords.html>`_ 。在"跳过模式"中，map 任务维护着正在处理的记录范围。为此，框架依赖于已处理记录计数器。参见 `SkipBadRecords.COUNTER_MAP_PROCESSED_RECORDS <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/SkipBadRecords.html>`_ 和 `SkipBadRecords.COUNTER_REDUCE_PROCESSED_GROUPS <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/SkipBadRecords.html>`_ 。该计数器使框架能够知道已经成功处理了多少记录以及什么记录范围会导致任务崩溃。在进一步的尝试中，这个范围的记录被跳过。

跳过的记录数取决于应用程序增加处理记录计数器的频率。建议在处理完每条记录后，再增加该计数器。在一些通常需要批量处理的应用程序中，可能没办法这么做。在这种情况下，框架可能会跳过脏数据周围的其他数据记录。用户可以通过 `SkipBadRecords.setMapperMaxSkipRecords(Configuration, long) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/SkipBadRecords.html>`_ 和 `SkipBadRecords.setReducerMaxSkipGroups(Configuration, long) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/SkipBadRecords.html>`_ 来控制跳过的记录数。该框架试图使用类似二分查找的方法来缩小跳过记录的范围。跳过的范围被分成两半，只有一半能够被执行。在后续的失败中，框架会查出哪一半包含脏记录。一个任务将被重新执行，直到满足可接受的跳过值或所有任务尝试都用尽。要增加任务尝试次数，可以使用 `Job.setMaxMapAttempts(int) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 和 `Job.setMaxReduceAttempts(int) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html>`_ 。

跳过的记录会以序列文件格式写入 HDFS，以便后面进行分析。文件存储路径可以通过 `SkipBadRecords.setSkipOutputPath(JobConf，Path) <http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/SkipBadRecords.html>`_ 进行更改。

.. _example_wordcount_v2:

示例: WordCount v2.0
================================

这是一个更完整的 WordCount 示例程序，它使用了我们迄今为止已经讨论过的 MapReduce 框架提供的许多功能。

运行这个示例程序需要启动并运行 HDFS，特别是 DistributedCache 相关的功能。因此它只能运行在 `伪分布式 <http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html>`_ 或 `完全分布式 <http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/ClusterSetup.html>`_ Hadoop 上。


.. _example_wordcount_v2_source_code:

源代码
--------------------------------

.. code-block:: Java

  import java.io.BufferedReader;
  import java.io.FileReader;
  import java.io.IOException;
  import java.net.URI;
  import java.util.ArrayList;
  import java.util.HashSet;
  import java.util.List;
  import java.util.Set;
  import java.util.StringTokenizer;

  import org.apache.hadoop.conf.Configuration;
  import org.apache.hadoop.fs.Path;
  import org.apache.hadoop.io.IntWritable;
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Job;
  import org.apache.hadoop.mapreduce.Mapper;
  import org.apache.hadoop.mapreduce.Reducer;
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
  import org.apache.hadoop.mapreduce.Counter;
  import org.apache.hadoop.util.GenericOptionsParser;
  import org.apache.hadoop.util.StringUtils;

  public class WordCount2 {

    public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {

      static enum CountersEnum { INPUT_WORDS }

      private final static IntWritable one = new IntWritable(1);
      private Text word = new Text();

      private boolean caseSensitive;
      private Set<String> patternsToSkip = new HashSet<String>();

      private Configuration conf;
      private BufferedReader fis;

      @Override
      public void setup(Context context) throws IOException,InterruptedException {
        conf = context.getConfiguration();
        caseSensitive = conf.getBoolean("wordcount.case.sensitive", true);
        if (conf.getBoolean("wordcount.skip.patterns", true)) {
          URI[] patternsURIs = Job.getInstance(conf).getCacheFiles();
          for (URI patternsURI : patternsURIs) {
            Path patternsPath = new Path(patternsURI.getPath());
            String patternsFileName = patternsPath.getName().toString();
            parseSkipFile(patternsFileName);
          }
        }
      }

      private void parseSkipFile(String fileName) {
        try {
          fis = new BufferedReader(new FileReader(fileName));
          String pattern = null;
          while ((pattern = fis.readLine()) != null) {
            patternsToSkip.add(pattern);
          }
        } catch (IOException ioe) {
          System.err.println("Caught exception while parsing the cached file '" + StringUtils.stringifyException(ioe));
        }
      }

      @Override
      public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
        String line = (caseSensitive) ? value.toString() : value.toString().toLowerCase();
        for (String pattern : patternsToSkip) {
          line = line.replaceAll(pattern, "");
        }
        StringTokenizer itr = new StringTokenizer(line);
        while (itr.hasMoreTokens()) {
          word.set(itr.nextToken());
          context.write(word, one);
          Counter counter = context.getCounter(CountersEnum.class.getName(),
          CountersEnum.INPUT_WORDS.toString());
          counter.increment(1);
        }
      }
  	}

    public static class IntSumReducer extends Reducer<Text,IntWritable,Text,IntWritable> {
      private IntWritable result = new IntWritable();

      public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        int sum = 0;
        for (IntWritable val : values) {
          sum += val.get();
        }
        result.set(sum);
        context.write(key, result);
      }
    }

    public static void main(String[] args) throws Exception {
      Configuration conf = new Configuration();
      GenericOptionsParser optionParser = new GenericOptionsParser(conf, args);
      String[] remainingArgs = optionParser.getRemainingArgs();
      if (!(remainingArgs.length != 2 | | remainingArgs.length != 4)) {
        System.err.println("Usage: wordcount <in> <out> [-skip skipPatternFile]");
        System.exit(2);
      }

      Job job = Job.getInstance(conf, "word count");
      job.setJarByClass(WordCount2.class);
      job.setMapperClass(TokenizerMapper.class);
      job.setCombinerClass(IntSumReducer.class);
      job.setReducerClass(IntSumReducer.class);
      job.setOutputKeyClass(Text.class);
      job.setOutputValueClass(IntWritable.class);

      List<String> otherArgs = new ArrayList<String>();
      for (int i=0; i < remainingArgs.length; ++i) {
        if ("-skip".equals(remainingArgs[i])) {
          job.addCacheFile(new Path(remainingArgs[++i]).toUri());
          job.getConfiguration().setBoolean("wordcount.skip.patterns", true);
        } else {
          otherArgs.add(remainingArgs[i]);
        }
      }
      FileInputFormat.addInputPath(job, new Path(otherArgs.get(0)));
      FileOutputFormat.setOutputPath(job, new Path(otherArgs.get(1)));

      System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
  }


.. _example_wordcount_v2_sample_runs:

运行示例
--------------------------------

示例文本文件作为输入:

.. code-block:: bash

  $ bin/hadoop fs -ls /user/joe/wordcount/input/
  /user/joe/wordcount/input/file01
  /user/joe/wordcount/input/file02

  $ bin/hadoop fs -cat /user/joe/wordcount/input/file01
  Hello World, Bye World!

  $ bin/hadoop fs -cat /user/joe/wordcount/input/file02
  Hello Hadoop, Goodbye to hadoop.

运行应用程序:

.. code-block:: bash

  $ bin/hadoop jar wc.jar WordCount2 /user/joe/wordcount/input /user/joe/wordcount/output

输出:

.. code-block:: bash

  $ bin/hadoop fs -cat /user/joe/wordcount/output/part-r-00000
  Bye 1
  Goodbye 1
  Hadoop, 1
  Hello 2
  World! 1
  World, 1
  hadoop. 1
  to 1

注意, 这里的输入与我们看到的第一个版本的示例程序的输入不同，因此输出也有不同。

现在我们通过 DistributedCache 插入一个模式文件，文件中列出了要忽略的单词模式。

.. code-block:: bash

  $ bin/hadoop fs -cat /user/joe/wordcount/patterns.txt
  \.
  \,
  \!
  to

再次运行示例程序，这次使用更多的选项：

.. code-block:: bash

  $ bin/hadoop jar wc.jar WordCount2 -Dwordcount.case.sensitive=true /user/joe/wordcount/input /user/joe/wordcount/output -skip /user/joe/wordcount/patterns.txt

和预期的一样，输出结果如下:

.. code-block:: bash

  $ bin/hadoop fs -cat /user/joe/wordcount/output/part-r-00000
  Bye 1
  Goodbye 1
  Hadoop 1
  Hello 2
  World 2
  hadoop 1

再一次运行示例程序，这一次我们关闭大小写敏感:

.. code-block:: bash

  $ bin/hadoop jar wc.jar WordCount2 -Dwordcount.case.sensitive=false /user/joe/wordcount/input /user/joe/wordcount/output -skip /user/joe/wordcount/patterns.txt

果然，输出结果是:

.. code-block:: bash

  $ bin/hadoop fs -cat /user/joe/wordcount/output/part-r-00000
  bye 1
  goodbye 1
  hadoop 2
  hello 2
  horld 2


.. _example_wordcount_v2_highlights:

要点
--------------------------------

通过使用 MapReduce 框架提供的一些特性，第二个版本的 WordCount 示例程序在之前一个版本的基础上做了以下改进:

* 展示了应用程序如何在 Mapper (以及 Reducer) 中的 setup 方法中访问配置参数。
* 展示了如何使用 DistributedCache 来分发作业所需的只读数据。这里允许用户指定单词模式，这样在计数时可以忽略那些符合模式的单词。
* 展示了用于处理 Hadoop 通用命令行选项的实用程序 GenericOptionsParser。
* 展示了应用程序如何使用 Counters, 以及如何设置传递给 map (和 reduce) 方法的特定于应用程序的状态信息。

Java 和 JNI 是 Oracle America, Inc 在美国和其他国家的商标或注册商标。
