---
title: "Fault Tolerance"
is_beta: false

sub-nav-group: streaming
sub-nav-id: fault_tolerance
sub-nav-pos: 4
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


Flink容错机制在程序出现异常状态时恢复程序并继续运行。故障包括机器硬件故障，网络故障和程序异常中断等。

* This will be replaced by the TOC
{:toc}


Streaming 容错机制
-------------------------

Flink checkpoint 机制在程序失败之后恢复job。checkpointing 机制需要*persistent* (或 *durable*) 可以再次读取之前的消息 (Apache Kafka 是一个很好的source)

checkpointing 机制存储进度在data sources 和 data sinks, windows状态和用户定义状态(见[Working with State](state.html)) 用于提供仅仅执行一次语义。checkpoint 存储根据[state backend](state_backends.html)配置(e.g., JobManager 内存, 文件系统, 数据库)。

[streaming fault tolerance文档]({{ site.baseurl }}/internals/stream_checkpointing.html) 描述了 Flink streaming 容错机制技术细节.

`StreamExecutionEnvironment` 上调用 `enableCheckpointing(n)` 开启 checkpointing 机制 , *n* 是 checkpoint 毫秒时间间隔。

checkpointing 其他参数包括:

- *Number of retries*: `setNumberOfExecutionRerties()`方法定义了job失败后的重试次数。当checkpointing 机制开启的时候,且这个值没有显式指定时，job会无限重试。

- *exactly-once vs. at-least-once*: 可以通过调用 `enableCheckpointing(n)` 方法在两个一致性级别之间选择其中之一。Exactly-once 更适合大多数程序。 At-least-once 适用于极低延迟 (几毫秒) 的应用.

- *number of concurrent checkpoints*: 默认情况下, 当有一个 checkpoint 在运行时，系统不会触发另一个程序。这可以确保 topology 不会在 checkpoint 上花费过多时间和处理数据流。可以允许多个 checkpoint 相互重叠，这对于有确定延迟的处理流程是有意义的。(举个例子，调用外部服务需要一定响应时间) 但频繁的建立 checkpoint (100ms) 会导致少量的 checkpoint 失败

- *checkpoint 超时*: 如果 checkpoint 尚未完成且时间已到，则checkpoint进程会被中断。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000);

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig().setCheckpointTimeout(60000);

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000)

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig.setCheckpointTimeout(60000)

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig.setMaxConcurrentCheckpoints(1)
{% endhighlight %}
</div>
</div>

{% top %}

### Data Sources 和 Sinks 容错保证

当 source 执行快照时，Flink 可以确保执行一次状态更新到用户状态。目前Kafka source (和内部序列生成器) 可以保证，但其他source不能保证。下表列出了Flink source 状态更新保证：

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Source</th>
      <th class="text-left" style="width: 25%">Guarantees</th>
      <th class="text-left">Notes</th>
    </tr>
   </thead>
   <tbody>
        <tr>
            <td>Apache Kafka</td>
            <td>exactly once</td>
            <td>使用与你Apache Kafka 版本相匹配的连接器</td>
        </tr>
        <tr>
            <td>RabbitMQ</td>
            <td>at most once (v 0.10) / exactly once (v 1.0) </td>
            <td></td>
        </tr>
        <tr>
            <td>Twitter Streaming API</td>
            <td>at most once</td>
            <td></td>
        </tr>
        <tr>
            <td>Collections</td>
            <td>exactly once</td>
            <td></td>
        </tr>
        <tr>
            <td>Files</td>
            <td>at least once</td>
            <td>失败时，会从文件开始处继续读取</td>
        </tr>
        <tr>
            <td>Sockets</td>
            <td>at most once</td>
            <td></td>
        </tr>
  </tbody>
</table>

为了确保端对端确保执行一次记录(一次执行状态语义),data sink需要检查点机制。下列表格表明了 Flink 绑定的sink状态更新保证(假设确保一次状态更新)

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Sink</th>
      <th class="text-left" style="width: 25%">Guarantees</th>
      <th class="text-left">Notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>HDFS rolling sink</td>
        <td>exactly once</td>
        <td>Implementation depends on Hadoop version</td>
    </tr>
    <tr>
        <td>Elasticsearch</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Kafka producer</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>File sinks</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Socket sinks</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Standard output</td>
        <td>at least once</td>
        <td></td>
    </tr>
  </tbody>
</table>

{% top %}

## 重启策略

Flink提供多种不同重启策略用于控制job在失败时如何重启。集群可以使用默认重启策略启动，经常作为job没有指定重启策略时的默认方案。如果job提交时指定了重启策略，会覆盖集群默认重启策略。
 
默认重启策略由 Flink 配置文件 `flink-conf.yaml` 所指定。配置参数 *restart-strategy* 定义了使用策略。默认情况下，没有启动重启策略。下列指定了可用的重启策略。

每个重启策略都有一套自己的参数用来控制他的行为。这些参数也在配置文件中指定。每一个重启策略的描述含有更多关于配置值的信息。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 50%">Restart Strategy</th>
      <th class="text-left">Value for restart-strategy</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>Fixed delay</td>
        <td>固定的延迟时间</td>
    </tr>
    <tr>
        <td>No restart</td>
        <td>none</td>
    </tr>
  </tbody>
</table>


除了默认重启策略，每个Flink job也可以指定各自重启策略。重启策略由 `ExecutionEnvironment` 调用`setRestartStrategy` 方法指定。`StreamExecutionEnvironment`也可以生效。
下面的例子展示了如何在job中设置重启延迟。在失败的情况下，系统会重试3次，每次之间间隔10秒。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelay(
  3, // number of restart attempts 
  10000 // delay in milliseconds
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelay(
  3, // number of restart attempts 
  10000 // delay in milliseconds
))
{% endhighlight %}
</div>
</div>

{% top %}

### 固定延迟重启策略

固定重启延迟策略指定次数之内重启重启job。如果超过最大重启次数，job被认定为最终失败。两次连续重启尝试之间，停顿一个固定时间间隔。

这个策略由 `flink-conf.yaml` 中设置下列参数设置为默认：

~~~
restart-strategy: fixed-delay
~~~

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Configuration Parameter</th>
      <th class="text-left" style="width: 40%">Description</th>
      <th class="text-left">Default Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td><it>restart-strategy.fixed-delay.attempts</it></td>
        <td>重试尝试次数</td>
        <td>1</td>
    </tr>
    <tr>
        <td><it>restart-strategy.fixed-delay.delay</it></td>
        <td>两次重启尝试之间的延迟时间</td>
        <td><it>akka.ask.timeout</it></td>
    </tr>
  </tbody>
</table>

~~~
restart-strategy.fixed-delay.attempts: 3
restart-strategy.fixed-delay.delay: 10 s
~~~

重启延迟也可以通过程序指定:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelay(
  3, // number of restart attempts 
  10000 // delay in milliseconds
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelay(
  3, // number of restart attempts 
  10000 // delay in milliseconds
))
{% endhighlight %}
</div>
</div>

#### 重试尝试次数

Flink 重试执行次数在job运行失败之前 由 *restart-strategy.fixed-delay.attempts* 参数指定。

默认为 **1**.

#### 重试间隔

执行重试可配置间隔时间。 重试间隔意味着在一次执行失败之后，并不是立即重新执行，而是延迟一段时间之后。

延迟重试当程序与外部系统交互时会很有用，例如连接或等待中的事务达到了超时时间。

参数 *akka.ask.timeout* 指定默认值。

{% top %}

### 不重启策略

job运行失败且不尝试重启。

~~~
restart-strategy: none
~~~

不重启策略可以使用程序声明:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.noRestart())
{% endhighlight %}
</div>
</div>

{% top %}
