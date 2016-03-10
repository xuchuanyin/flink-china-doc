---
title: "Apache Kafka Connector"

# Sub-level navigation
sub-nav-group: streaming
sub-nav-parent: connectors
sub-nav-pos: 1
sub-nav-title: Kafka
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

该连接器可以访问由 [Apache Kafka](https://kafka.apache.org/) 提供的事件流。

Flink 提供了特殊的 Kafka 连接器，用来从 Kafka 话题（topic）中读取数据，和写入数据到 Kafka 话题中。Flink Kafka Consumer 整合了 Flink 的 checkpoint 机制来提供 exactly-once 处理语义。要实现这个，Flink 并不是单纯依赖 Kafka 的消费组偏移跟踪，而是在内部也会跟踪并 checkpoint 这些偏移（offset）。

请根据你的案例和环境来选择对应的包（maven artifact id）和类名。对于大多数用户来说，`FlinkKafkaConsumer08`（在 `flink-connector-kafka` 中）是很合适的。


<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left">Maven 依赖</th>
      <th class="text-left">支持自</th>
      <th class="text-left">Consumer 和 <br>
      Producer 类名</th>
      <th class="text-left">Kafka 版本</th>
      <th class="text-left">注意</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>flink-connector-kafka</td>
        <td>0.9.1, 0.10</td>
        <td>FlinkKafkaConsumer082<br>
        FlinkKafkaProducer</td>
        <td>0.8.x</td>
        <td>内部使用 Kafka 的 <a href="https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example">SimpleConsumer</a> API 。Flink 会将 offset 提交到 ZK。</td>
    </tr>
     <tr>
        <td>flink-connector-kafka-0.8{{ site.scala_version_suffix }}</td>
        <td>1.0.0</td>
        <td>FlinkKafkaConsumer08<br>
        FlinkKafkaProducer08</td>
        <td>0.8.x</td>
        <td>内部使用 Kafka 的 <a href="https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example">SimpleConsumer</a> API 。Flink 会将 offset 提交到 ZK。</td>
    </tr>
     <tr>
        <td>flink-connector-kafka-0.9{{ site.scala_version_suffix }}</td>
        <td>1.0.0</td>
        <td>FlinkKafkaConsumer09<br>
        FlinkKafkaProducer09</td>
        <td>0.9.x</td>
        <td>使用新的 Kafka <a href="http://kafka.apache.org/documentation.html#newconsumerapi">Consumer API</a></td>
    </tr>
  </tbody>
</table>

然后，将连接器导入到你的 maven 工程中：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka-0.8{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

注意流连接器目前还不是二进制发布包中的一部分。请查阅 [这里]({{ site.baseurl}}/apis/cluster_execution.html#linking-with-modules-not-contained-in-the-binary-distribution) 了解如何在集群环境中关联它们。

#### 安装 Apache Kafka

* 按照 [Kafka's quickstart](https://kafka.apache.org/documentation.html#quickstart) 的指引下载并启动服务（在启动应用程序之前，必须先启动一个 Zookeeper 和 一个 Kafka 服务）。
* 在 32 位的机器上，[这个问题](http://stackoverflow.com/questions/22325364/unrecognized-vm-option-usecompressedoops-when-running-kafka-from-my-ubuntu-in) 可能会发生。
* 如果 Kafka 和 Zookeeper 服务运行在远程机器上，那么 `config/server.properties` 中的 `advertised.host.name` 配置必须设置成机器的 IP 地址。

#### Kafka 消费者

Flink 的 Kafka 消费者称作 `FlinkKafkaConsumer08` (或 `09`)。它提供了访问一个或多个 Kafka 话题的功能。

该类的构造器需要接受如下的参数：

1. 话题名 / 话题列表
2. 一个 `DeserializationSchema` / `KeyedDeserializationSchema` 用来反序列化从 Kafka 获得的数据。
3. Kafka 消费者的 `Properties`（属性）：
  下面的属性是必须的：
  - "bootstrap.servers" (逗号分隔的 Kafka brokers 列表)
  - "zookeeper.connect" (逗号分隔的 Zookeeper servers 列表) (**只有 Kafka 0.8 才需要**)
  - "group.id" 消费组的 id

示例：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
DataStream<String> stream = env
	.addSource(new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties))
	.print();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
stream = env
    .addSource(new FlinkKafkaConsumer08[String]("topic", new SimpleStringSchema(), properties))
    .print
{% endhighlight %}
</div>
</div>


##### `DeserializationSchema`

`FlinkKafkaConsumer08` 需要知道如何将 Kafka 中的数据转成 Java 对象。`DeserializationSchema` 允许用户指定这样的一个 schema。Kafka 的每条消息都会调用 `T deserialize(byte[] message)` 方法，传入来自 Kafka 的值作为参数。要同时获得 Kafka 消息中的键和值，可以使用 `KeyedDeserializationSchema` 的反序列化方法 `T deserialize(byte[] messageKey, byte[] message, String topic, int partition, long offset)`。

为了方便，Flink 提供了 `TypeInformationSerializationSchema` (和 `TypeInformationKeyValueSerializationSchema`) ，用来创建基于 Flink `TypeInformation` 的 schema。

#### Kafka 消费者和容错

当 Flink 的 checkpoint 开启了，Flink Kafka Consumer 会从话题中消费记录，并以一致性的方式，周期性地 checkpoint 它的 Kafka offsets，以及其他操作中的状态。当作业失败了，Flink 会恢复流程序到最后一个 checkpoint 的状态，然后重新从 Kafka 的保存在 checkpoint 中的 offset 处开始消费记录，

因此在故障情况下，写 checkpoint 的间隔决定了程序最多可能需要回退多少。

要使用 Kafka Consumers 的容错，需要在 execution environment 上打开拓扑的 checkpoint：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(5000); // checkpoint every 5000 msecs
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.enableCheckpointing(5000) // checkpoint every 5000 msecs
{% endhighlight %}
</div>
</div>

另外注意 Flink 只有在还有足够的 slots 数的时候，才能够重启拓扑。所以如果是由于 TaskManager 挂了导致拓扑失败，那集群中必须仍有足够的 slots 才行。Flink on YARN 支持了对 YARN 容器的自动重启。

如果没有开启 checkpoint，那么 Kafka consumer 会周期性地将 offset 提交到 Zookeeper。

#### Kafka 生产者

`FlinkKafkaProducer08` 能将数据写入到 Kafka 话题中。生产者可以指定一个自定义的分区来写入数据。


示例:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
stream.addSink(new FlinkKafkaProducer08<String>("localhost:9092", "my-topic", new SimpleStringSchema()));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
stream.addSink(new FlinkKafkaProducer08[String]("localhost:9092", "my-topic", new SimpleStringSchema()))
{% endhighlight %}
</div>
</div>

你可以使用这个构造器为 KafkaSink 定义一个自定义的 Kafka 生产者配置。请查阅 [Apache Kafka 文档](https://kafka.apache.org/documentation.html) 更详细地了解关于如何配置 Kafka 生产者。

**注意**：默认情况下，重试次数被设为 “0”。也就是说生产者在发生错误后会立即失败，包括 leader 改变。默认设置为“0”，是为了避免目标话题中出现重复消息。对于大多数生产环境来说（一般都有频繁的 broker 切换），我们建议将重试次数的值调大。

目前还没有事务性的 Kafka 生产者，所以 Flink 还不能保证发送到 Kafka 话题的 exactly-once 传递。

