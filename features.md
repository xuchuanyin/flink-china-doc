---
title: "Features"
---


<!-- --------------------------------------------- -->
<!--                Streaming
<!-- --------------------------------------------- -->

----

<div class="row" style="padding: 0 0 0 0">
  <div class="col-sm-12" style="text-align: center;">
    <h1 id="streaming"><b>Streaming</b></h1>
  </div>
</div>

----

<!-- High Performance -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="performance"><i>高吞吐 & 低延迟</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-12">
    <p class="lead">Flink 的流处理引擎只需要很少配置就能实现高吞吐率和低延迟。下图展示了一个分布式计数的任务的性能，包括了流数据 shuffle 过程。</p>
  </div>
</div>
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12 img-column">
    <img src="{{ site.baseurl }}/fig/features/streaming_performance.png" alt="Performance of data streaming applications" style="width:75%" />
  </div>
</div>

----

<!-- Event Time Streaming -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="event_time"><i>支持 Event Time 和乱序事件</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-6">
    <p class="lead">Flink 支持了流处理和 <b>Event Time</b> 语义的窗口机制。</p>
    <p class="lead">Event time 使得计算乱序到达的事件或可能延迟到达的事件更加简单。</p>
  </div>
  <div class="col-sm-6 img-column">
    <img src="{{ site.baseurl }}/fig/features/out_of_order_stream.png" alt="Event Time and Out-of-Order Streams" style="width:100%" />
  </div>
</div>

----

<!-- Exactly-once Semantics -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="exactly_once"><i>状态计算的 exactly-once 语义</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-6">
    <p class="lead">流程序可以在计算过程中维护自定义状态。</p>
    <p class="lead">Flink 的 checkpointing 机制保证了即时在故障发生下也能保障状态的 <i>exactly once</i> 语义。</p>
  </div>
  <div class="col-sm-6 img-column">
    <img src="{{ site.baseurl }}/fig/features/exactly_once_state.png" alt="Exactly-once Semantics for Stateful Computations" style="width:50%" />
  </div>
</div>

----

<!-- Windowing -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="windows"><i>高度灵活的流式窗口</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-6">
    <p class="lead">Flink 支持在时间窗口，统计窗口，session 窗口，以及数据驱动的窗口</p>
    <p class="lead">窗口可以通过灵活的触发条件来定制，以支持复杂的流计算模式。</p>
  </div>
  <div class="col-sm-6 img-column">
    <img src="{{ site.baseurl }}/fig/features/windows.png" alt="Windows" style="width:100%" />
  </div>
</div>

----

<!-- Continuous streaming -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="streaming_model"><i>带反压的连续流模型</i></h1>
  </div>
</div>

<div class="row">
  <div class="col-sm-6">
    <p class="lead">数据流应用执行的是不间断的（常驻）operators。</p>
    <p class="lead">Flink streaming 在运行时有着天然的流控：慢的数据 sink 节点会反压（backpressure）快的数据源（sources）。</p>
  </div>
  <div class="col-sm-6 img-column">
    <img src="{{ site.baseurl }}/fig/features/continuous_streams.png" alt="Continuous Streaming Model" style="width:60%" />
  </div>
</div>

----

<!-- Lightweight distributed snapshots -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="snapshots"><i>容错性</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-6">
    <p class="lead">Flink 的容错机制是基于 Chandy-Lamport distributed snapshots 来实现的。</p>
    <p class="lead">这种机制是非常轻量级的，允许系统拥有高吞吐率的同时还能提供强一致性的保障。</p>
  </div>
  <div class="col-sm-6 img-column">
    <img src="{{ site.baseurl }}/fig/features/distributed_snapshots.png" alt="Lightweight Distributed Snapshots" style="width:40%" />
  </div>
</div>

----

<!-- --------------------------------------------- -->
<!--                Batch
<!-- --------------------------------------------- -->

<div class="row" style="padding: 0 0 0 0">
  <div class="col-sm-12" style="text-align: center;">
    <h1 id="batch-on-streaming"><b>Batch 和 Streaming 一个系统</b></h1>
  </div>
</div>

----

<!-- One Runtime for Streaming and Batch Processing -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="one_runtime"><i>流处理和批处理共用一个引擎</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-6">
    <p class="lead">Flink 为流处理和批处理应用公用一个通用的引擎。</p>
    <p class="lead">批处理应用可以以一种特殊的流处理应用高效地运行。</p>
  </div>
  <div class="col-sm-6 img-column">
    <img src="{{ site.baseurl }}/fig/features/one_runtime.png" alt="Unified Runtime for Batch and Stream Data Analysis" style="width:50%" />
  </div>
</div>

----


<!-- Memory Management -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="memory_management"><i>内存管理</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-6">
    <p class="lead">Flink 在 JVM 中实现了自己的内存管理。</p>
    <p class="lead">应用可以超出主内存的大小限制，并且承受更少的垃圾收集的开销。</p>
  </div>
  <div class="col-sm-6 img-column">
    <img src="{{ site.baseurl }}/fig/features/memory_heap_division.png" alt="Managed JVM Heap" style="width:50%" />
  </div>
</div>

----

<!-- Iterations -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="iterations"><i>迭代和增量迭代</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-6">
    <p class="lead">Flink 具有迭代计算的专门支持（比如在机器学习和图计算中）。</p>
    <p class="lead">增量迭代可以利用依赖计算来更快地收敛。</p>
  </div>
  <div class="col-sm-6 img-column">
    <img src="{{ site.baseurl }}/fig/features/iterations.png" alt="Performance of iterations and delta iterations" style="width:75%" />
  </div>
</div>

----

<!-- Optimizer -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="optimizer"><i>程序调优</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-6">
    <p class="lead">批处理程序会自动地优化一些场景，比如避免一些昂贵的操作（如 shuffles 和 sorts），还有缓存一些中间数据。</p>
  </div>
  <div class="col-sm-6 img-column">
    <img src="{{ site.baseurl }}/fig/features/optimizer_choice.png" alt="Optimizer choosing between different execution strategies" style="width:100%" />
  </div>
</div>

----

<!-- --------------------------------------------- -->
<!--             APIs and Libraries
<!-- --------------------------------------------- -->

<div class="row" style="padding: 0 0 0 0">
  <div class="col-sm-12" style="text-align: center;">
    <h1 id="apis-and-libs"><b>API 和 类库</b></h1>
  </div>
</div>

----

<!-- Data Streaming API -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="streaming_api"><i>流处理应用</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-5">
    <p class="lead"><i>DataStream</i> API 支持了数据流上的函数式转换，可以使用自定义的状态和灵活的窗口。</p>
    <p class="lead">右侧的示例展示了如何以滑动窗口的方式统计文本数据流中单词出现的次数。</p>
  </div>
  <div class="col-sm-7">
    <p class="lead">WindowWordCount in Flink's DataStream API</p>
{% highlight scala %}
case class Word(word: String, freq: Long)

val texts: DataStream[String] = ...

val counts = text
  .flatMap { line => line.split("\\W+") } 
  .map { token => Word(token, 1) }
  .keyBy("word")
  .timeWindow(Time.seconds(5), Time.seconds(1))
  .sum("freq")
{% endhighlight %}
  </div>
</div>

----

<!-- Batch Processing API -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="batch_api"><i>批处理应用</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-5">
    <p class="lead">Flink 的 <i>DataSet</i> API 可以使你用 Java 或 Scala 写出漂亮的、类型安全的、可维护的代码。它支持广泛的数据类型，不仅仅是 key/value 对，以及丰富的 operators。</p>
    <p class="lead">右侧的示例展示了图计算中 PageRank 算法的一个核心循环。</p>
  </div>
  <div class="col-sm-7">
{% highlight scala %}
case class Page(pageId: Long, rank: Double)
case class Adjacency(id: Long, neighbors: Array[Long])

val result = initialRanks.iterate(30) { pages =>
  pages.join(adjacency).where("pageId").equalTo("pageId") {

    (page, adj, out : Collector[Page]) => {
      out.collect(Page(page.id, 0.15 / numPages))
        
      for (n <- adj.neighbors) {
        out.collect(Page(n, 0.85*page.rank/adj.neighbors.length))
      }
    }
  }
  .groupBy("pageId").sum("rank")
}
{% endhighlight %}
  </div>
</div>

----

<!-- Library Ecosystem -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="libraries"><i>类库生态</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-6">
    <p class="lead">Flink 栈中提供了提供了很多具有高级 API 和满足不同场景的类库：机器学习、图分析、关系式数据处理。</p>
    <p class="lead">当前类库还在 <i>beta</i> 状态，并且在大力发展。</p>
  </div>
  <div class="col-sm-6 img-column">
    <img src="{{ site.baseurl }}/fig/features/stack.png" alt="Flink Stack with Libraries" style="width:60%" />
  </div>
</div>

----

<!-- --------------------------------------------- -->
<!--             Ecosystem
<!-- --------------------------------------------- -->

<div class="row" style="padding: 0 0 0 0">
  <div class="col-sm-12" style="text-align: center;">
    <h1><b>生态系统</b></h1>
  </div>
</div>

----

<!-- Ecosystem -->
<div class="row" style="padding: 0 0 2em 0">
  <div class="col-sm-12">
    <h1 id="ecosystem"><i>广泛集成</i></h1>
  </div>
</div>
<div class="row">
  <div class="col-sm-6">
    <p class="lead">Flink 与开源大数据处理生态系统中的许多项目都有集成。</p>
    <p class="lead">Flink 可以运行在 YARN 上，与 HDFS 协同工作，从 Kafka 中读取流数据，可以执行 Hadoop 程序代码，可以连接多种数据存储系统。</p>
  </div>
  <div class="col-sm-6  img-column">
    <img src="{{ site.baseurl }}/fig/features/ecosystem_logos.png" alt="Other projects that Flink is integrated with" style="width:75%" />
  </div>
</div>
