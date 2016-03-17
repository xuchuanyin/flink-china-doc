---
title: "容错"
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


Flink容错机制能够在程序出现异常状态时恢复并继续运行。故障包括机器硬件故障，网络故障和程序异常中断等。

* This will be replaced by the TOC
{:toc}


Streaming 容错机制
-------------------------

Flink checkpoint 机制能够在程序失败之后恢复任务。checkpoint 机制需要依赖于如 Apache Kafka 这样的*可持久化* 并且能重放历史数据的数据源。

Checkpoint 机制存储了 data sources 和 data sinks 的处理进度、window 的状态、以及用户自定义的状态(见[Working with State](state.html)) ，以保证只处理一次的语义。根据[state backend](state_backends.html)的配置，checkpoint 可以存储在 JobManager 内存, 文件系统以及数据库中。

[Streaming 容错文档]({{ site.baseurl }}/internals/stream_checkpointing.html) 描述了 Flink streaming 容错机制的技术细节.

在 `StreamExecutionEnvironment` 上调用 `enableCheckpointing(n)` 可以开启 checkpoint 机制 , *n* 是 checkpoint 毫秒时间间隔。

checkpoint 其他参数包括:

- *Number of retries*: `setNumberOfExecutionRerties()`方法定义了job失败后的重试次数。当 checkpoint 机制开启的时候,且这个值没有显式指定时，job会无限重试。

- *exactly-once vs. at-least-once*: 可以通过调用 `setCheckpointingMode()` 方法在两个一致性级别之间选择其中之一。Exactly-once 更适合大多数程序。 At-least-once 适用于极低延迟 (几毫秒) 的应用.

- *number of concurrent checkpoints*: 默认情况下, 当有一个 checkpoint 在运行时，系统不会触发另一个 checkpoint。这可以确保 topology 不会在 checkpoint 上花费过多时间并且不会与数据流的处理竞争资源。然而，有时仍然需要允许多个重叠的 checkpoint 操作，这对于有确定延迟（如调用需要一定响应时间的外部服务）但又想频繁地做 checkpoint（百毫秒级别，为了在失败情况下重新处理尽量少的数据量）的处理流程是有意义的。


- *checkpoint 超时*: 如果 checkpoint 在该时间内还未完成，则正在进行中的 checkpoint 会被中断。

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

### Data Source 和 Sink 容错保证

只有当数据源参与了快照机制，Flink 才能保证只对用户状态更新一次（exactly-once）。目前 Kafka source (和内部序列生成器) 可以保证，但其他 source 不能保证。下表列出了Flink source 状态更新保证：

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
            <td>使用与你Apache Kafka 版本相匹配的 connector</td>
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

为了确保端到端的 exactly-once 消息传递(除了 exactly-once 状态语义)，data sink 也需要参与 checkpoint 机制。下表列出了和 Flink 绑定的 sink 的传递保证（假设状态是 exactly-once 更新了）：

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
        <td>是否实现取决于 Hadoop 版本</td>
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

Flink提供多种不同重启策略用于控制job在失败时如何重启。当没有指定 job 的重启策略时，集群会使用默认的重启策略。如果job提交时指定了重启策略，会覆盖集群默认重启策略。
 
默认重启策略由 Flink 配置文件 `flink-conf.yaml` 所指定。配置参数 *restart-strategy* 定义了使用策略。默认情况下，不重启策略会被使用。下列列出了可用的重启策略。

每个重启策略都有一套自己的参数用来控制他的行为。这些参数也在配置文件中指定。每一个重启策略的描述含有更多关于配置值的信息。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 50%">重启策略</th>
      <th class="text-left">restart-strategy 的值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>固定延迟</td>
        <td>fixed-delay</td>
    </tr>
    <tr>
        <td>不重启</td>
        <td>none</td>
    </tr>
  </tbody>
</table>


除了默认重启策略，每个Flink job也可以指定各自重启策略。重启策略由 `ExecutionEnvironment` 中`setRestartStrategy` 方法指定。注意这对 `StreamExecutionEnvironment` 也是生效的。
下面的例子展示了如何在job中设置固定延迟的重启策略。在失败的情况下，系统会重试3次，每次之间间隔10秒。


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

固定延迟重启策略会在指定次数之内重启重启job。如果超过最大重启次数，job被认定为最终失败。两次连续重启尝试之间，等待一个固定时间间隔。

这个策略是通过在 `flink-conf.yaml` 中设置下列参数来启用的：

~~~
restart-strategy: fixed-delay
~~~

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">配置参数</th>
      <th class="text-left" style="width: 40%">描述</th>
      <th class="text-left">默认值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td><it>restart-strategy.fixed-delay.attempts</it></td>
        <td>重启尝试次数</td>
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

重启延迟也可以通过编程方式指定:

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

#### 重启尝试次数

在 job 被宣告失败前，Flink 重试执行的次数可以通过 *restart-strategy.fixed-delay.attempts* 参数来配置。

默认为 **1**.

#### 重试间隔

执行重试可配置间隔时间。 重试间隔意味着在一次执行失败之后，并不是立即重新执行，而是延迟一段时间之后。

当程序与外部系统交互时，延迟重试会很有用，例如连接或等待中的事务达到了超时时间。

默认值是 *akka.ask.timeout* 的值。

{% top %}

### 不重启策略

job 运行失败且不尝试重启。

~~~
restart-strategy: none
~~~

不重启策略可以用编程方式声明:

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
