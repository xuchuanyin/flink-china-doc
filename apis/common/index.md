---
title: "基本概念"

# Top-level navigation
top-nav-group: apis
top-nav-pos: 1
top-nav-title: <strong>基本概念</strong>

# Sub-level navigation
#sub-nav-group: common
#sub-nav-group-title: Basic Concepts
#sub-nav-pos: 1
#sub-nav-title: Basic Concepts
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


Flink 程序是在分布式集合上实现转换（比如 filtering, mapping, updating state, joining, grouping, defining windows, aggregating）的常用方案。 数据集合根据一些 source（比如 文件，kafka 或本地的集合）初始化创建。 通过 sinks 返回的结果，sink 可以写入到（分布式）文件或标准输出（如终端命令行）。Flink 运行在各种环境下，standalone 或嵌入到其他程序中。 可能在本地 JVM 中执行，或多台机器的集群上运行。

根据数据源的类型，例如有界或无界的数据源，用户可以实现一个批处理(batch program)或一个流式程序(streaming program), 其中 DataSet API 用于前者，DataStream API 用于后者。本文档介绍这两种 API 共通的一些基本概念。更详细的编程指南请参考 [Streaming 指南]({{ site.baseurl }}/apis/streaming/index.html) 和
[Batch 指南]({{ site.baseurl }}/apis/batch/index.html)。

**注意**: 在本文档介绍 API 使用的实例中，我们将使用 `StreamingExecutionEnvironment` 和 DataStream API, 这些概念同样适用于 DataSet API，仅仅是换成 `ExecutionEnvironment` and DataSet 而已。


* This will be replaced by the TOC
{:toc}

关联 Flink
------------------

要用 Flink 编写程序，你需要引入与你使用的编程语言相对应的 Flink library。

最简单的引入 Flink 到项目的方式是使用快速起步中的脚本，比如 [Java API]({{ site.baseurl }}/quickstart/java_api_quickstart.html) 或是 [Scala API]({{ site.baseurl }}/quickstart/scala_api_quickstart.html)。也可以通过下面的 maven 插件来创建一个空的项目，`archetypeVersion` 可以选择其他稳定版本或预发版本（`-SNAPSHOT`）版本。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight bash %}
mvn archetype:generate \
    -DarchetypeGroupId=org.apache.flink \
    -DarchetypeArtifactId=flink-quickstart-java \
    -DarchetypeVersion={{site.version }}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight bash %}
mvn archetype:generate \
    -DarchetypeGroupId=org.apache.flink \
    -DarchetypeArtifactId=flink-quickstart-scala \
    -DarchetypeVersion={{site.version }}
{% endhighlight %}
</div>
</div>

如果想要把 Flink 加入到现有的 maven 项目中，则需要添加依赖，将下面这段添加到项目 *pom.xml* 的 *dependencies* 块中：


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight xml %}
<!-- Use this dependency if you are using the DataStream API -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-java{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
<!-- Use this dependency if you are using the DataSet API -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-java</artifactId>
  <version>{{site.version }}</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight xml %}
<!-- Use this dependency if you are using the DataStream API -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
<!-- Use this dependency if you are using the DataSet API -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}


**重要:** 如果使用 Scala API，则需要 import 下面两者其中一个。
{% highlight scala %}
import org.apache.flink.api.scala._
{% endhighlight %}

or

{% highlight scala %}
import org.apache.flink.api.scala.createTypeInformation
{% endhighlight %}

原因是 Flink 需要分析程序中使用的类型，从而生成 serializers 和 comparaters 。通过 import 上述两者之一，可以开启一个隐式的转换，为 Flink 的操作创建类型信息。

</div>
</div>


#### Scala 依赖版本

因为scala 2.10 binary 不兼容2.11，因此我们提供多个 artifacts 来支持这两种 Scala 版本。

从 0.10 开始，我们为 2.10 和 2.11 交叉编译了所有的 Flink 模块。如果你想要将你的程序跑在 Scala 2.11 的 Flink 上，只需要在对应模块的 `artifactId` 后加上 `_2.11` 版本后缀。

如果你想要用 Scala 2.11 自己编译 Flink，可以参考 [编译 Flink]({{ site.baseurl }}/setup/building.html#scala-versions)。


#### Hadoop 依赖版本

如果 Flink 要运行在 Hadoop 上，请选择正确的 Hadoop 版本，详情请参考 [下载页面](http://flink.apache.org/downloads.html) 可用版本列表，同时了解如何关联自定义版本的 Hadoop。

想要关联最新的 SNAPSHOT 版本的代码，请参考 [这篇教程](http://flink.apache.org/how-to-contribute.html#snapshots-nightly-builds)。

*flink-clients* 依赖仅仅是本地模式需要（如测试调试）。 如果想要 [在集群模式下运行]({{ site.baseurl }}/apis/cluster_execution.html) ，可以忽略这个依赖。


{% top %}

DataSet 和 DataStream
----------------------

Flink 用特别的类 DataSet 和 DataStream 来表示程序中的数据。用户可以认为它们是含有重复数据的不可修改的集合(collection)。当使用 DataSet 时，数据是有限的，而DataStream 中元素的数量是无限的。

在几个关键点上， 这些集合与 Java 集合是不同的。首先，它们是不可修改的，这意味着它们一旦被创建出来，就不能添加或删除元素。你也不能简单地查看内部的元素。

在一个 Flink 程序中，一个集合是通过添加一个 source 来初始化获得的，新的集合可以通过转换（transformation）得到，转换的 API 函数有 `map`，`filter` 等。

<a id="anatomy-of-a-flink-program"></a>

剖析 Flink 程序
--------------------------

Flink 程序看起来就像常规程序一样转换数据集合。每一个程序都由一些基本部分组成：

1. 获得一个 `execution environment`，
2. 加载/创建初始数据，
3. 指定在该数据上进行的转换，
4. 指定计算结果的存储地方，
5. 启动程序执行。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

现在，我们将大致介绍下每一步，你可以参考相应的章节了解更多。注意所有的 Java DataSet API 核心类都可以在 {% gh_link /flink-java/src/main/java/org/apache/flink/api/java "org.apache.flink.api.java" %} 中找到，而所有的 Java DataStream API 核心类可以在 {% gh_link /flink-streaming-java/src/main/java/org/apache/flink/streaming/api "org.apache.flink.streaming.api" %} 中找到。


`StreamExecutionEnvironment` 是所有 Flink 程序的基础，你可以使用 `StreamExecutionEnvironment` 中的这些静态方法来获得：

{% highlight java %}
getExecutionEnvironment()

createLocalEnvironment()

createRemoteEnvironment(String host, int port, String... jarFiles)
{% endhighlight %}

一般来说， 你只需要调用 `getExecutionEnvironment()`，因为 Flink 会根据不同的上下文来做对应的事情，比如你是在一个 IDE 中，或是作为一个常规的 Java 程序来执行 Flink 程序，那么它会创建一个本地环境，即在本地机器上执行你的程序。如果你编译了一个 JAR 文件出来，并通过 [命令行]({{ site.baseurl }}/apis/cli.html) 提交，那么 Flink 集群 manager 会执行你的 main 方法，而 `getExecutionEnvironment()` 会返回一个集群环境，即在集群上执行你的程序。

执行环境（execution environment）有多种读取文件的方法用来指定数据源：可以一行一行来读取，比如csv文件， 或使用自定义的输入格式。 可以通过下列方式，来按行来读取一个文本文件：


{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = env.readTextFile("file:///path/to/file");
{% endhighlight %}

通过这种方式你就能得到一个 DataStream，在这之上你可以接着应用一些转换来得到衍生的 DataStream。

你可以通过调用 DataStream 上的转换函数来应用转换。例如，一个 map 转换看起来是这样的：

{% highlight java %}
DataStream<String> input = ...;

DataStream<Integer> parsed = input.map(new MapFunction<String, Integer>() {
    @Override
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
});
{% endhighlight %}

这会创建一个新的 DataStream，其中原始集合中的每一个 String 都被转换成了 Integer。

一旦你得到了一个包含最终结果的 DataStream，你就可以通过创建一个 sink 将它写入到外部系统中。以下是一些创建 sink 的方法例子：

{% highlight java %}
writeAsText(String path)

print()
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

现在，我们将大致介绍下每一步，你可以参考相应的章节了解更多。注意所有的 Scala DataSet API 核心类都可以在 {% gh_link /flink-scala/src/main/scala/org/apache/flink/api/scala "org.apache.flink.api.scala" %} 中找到，而所有的 Scala DataStream API 核心类可以在 {% gh_link /flink-streaming-scala/src/main/java/org/apache/flink/streaming/api/scala "org.apache.flink.streaming.api.scala" %} 中找到。

`StreamExecutionEnvironment` 是所有 Flink 程序的基础，你可以使用 `StreamExecutionEnvironment` 中的这些静态方法来获得：

{% highlight scala %}
getExecutionEnvironment()

createLocalEnvironment()

createRemoteEnvironment(host: String, port: Int, jarFiles: String*)
{% endhighlight %}

一般来说， 你只需要调用 `getExecutionEnvironment()`，因为 Flink 会根据不同的上下文来做对应的事情，比如你是在一个 IDE 中，或是作为一个常规的 Java 程序来执行 Flink 程序，那么它会创建一个本地环境，即在本地机器上执行你的程序。如果你编译了一个 JAR 文件出来，并通过 [命令行]({{ site.baseurl }}/apis/cli.html) 提交，那么 Flink 集群 manager 会执行你的 main 方法，而 `getExecutionEnvironment()` 会返回一个集群环境，即在集群上执行你的程序。

执行环境（execution environment）有多种读取文件的方法用来指定数据源：可以一行一行来读取，比如csv文件， 或使用自定义的输入格式。 可以通过下列方式，来按行来读取一个文本文件：

{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val text: DataStream[String] = env.readTextFile("file:///path/to/file")
{% endhighlight %}

通过这种方式你就能得到一个 DataStream，在这之上你可以接着应用一些转换来得到衍生的 DataStream。

你可以通过调用 DataStream 上的转换函数来应用转换。例如，一个 map 转换看起来是这样的：

{% highlight scala %}
val input: DataSet[String] = ...

val mapped = input.map { x => x.toInt }
{% endhighlight %}

这会创建一个新的 DataStream，其中原始集合中的每一个 String 都被转换成了 Integer。

一旦你得到了一个包含最终结果的 DataStream，你就可以通过创建一个 sink 将它写入到外部系统中。以下是一些创建 sink 的方法例子：

{% highlight scala %}
writeAsText(path: String)

print()
{% endhighlight %}

</div>
</div>

一旦完成程序，用户需要**启动程序执行**，可以直接调用 `StreamExecutionEnviroment` 的 `execute()`， 根据不同的 `ExecutionEnvironment` 类型，将会决定本地模式执行或是提交到集群上执行。

`execute()` 函数会返回 `JobExecutionResult`，包含执行的次数和累加器（accumulator）结果。

请查阅 [Streaming 指南]({{ site.baseurl }}/apis/streaming/index.html) 了解更多关于流数据源、sink、以及关于 DataStream 所支持转换（transformations）的更深入的信息。

请查阅 [Batch 指南]({{ site.baseurl }}/apis/batch/index.html) 了解更多关于批数据源、sink、以及关于 DataSet 所支持转换（transformations）的更深入的信息。


{% top %}

延迟计算（Lazy Evaluation）
---------------

所有的 Flink 程序都是延迟执行的：当执行程序的 main 函数时，数据加载了但转换并没有马上执行。相反，每个操作都被创建并加入到了程序执行计划中。当调用 `ExecutionEnvironment` 的 `execute()` 时，这些操作才被真正执行。程序是本地执行还是分布式执行取决于执行环境。

延迟计算的机制可以让你能构建复杂的程序，而 Flink 仍会视为一个整体计划执行。

{% top %}

指定 Keys
---------------

一些转换（join，coGroup，keyBy，groupBy）需要在数据集上定义一个 key 。另外一些转换（Reduce，GroupReduce，Aggregate，Window）允许应用在根据 key 分组的数据上（如 KeyedStream）。

一个 DataSet 可以如下这样被分组：
{% highlight java %}
DataSet<...> input = // [...]
DataSet<...> reduced = input
  .groupBy(/*define key here*/)
  .reduceGroup(/*do something*/);
{% endhighlight %}

而在 DataStream 上指定一个 key 时需要：
{% highlight java %}
DataStream<...> input = // [...]
DataStream<...> windowed = input
  .key(/*define key here*/)
  .window(/*window specification*/);
{% endhighlight %}

Flink 的数据模型并不是基于键值对的。因此，你不需要把数据类型转换成 keys/values。Key 实际上是“虚拟”的：它们被定义为实际数据上的函数，用来引导分组操作（grouping operator）。

**注意：**在下面的讨论中我们会使用 `DataStream` API 和 `keyBy`。对于 DataSet API 你只需要替换成 `DataSet` 和  `groupBy`。


### 为 Tuple 定义 key
{:.no_toc}

最简单的例子是，在一个或多个字段上对一组 Tuples 进行 grouping：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long> keyed = input.keyBy(0)
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(Int, String, Long)] = // [...]
val keyed = input.keyBy(0)
{% endhighlight %}
</div>
</div>

这组 Tuples 是在第一个字段上进行分组的（Integer 类型的那个）。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long> keyed = input.keyBy(0,1)
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataSet[(Int, String, Long)] = // [...]
val grouped = input.groupBy(0,1)
{% endhighlight %}
</div>
</div>

这里，我们在第一个和第二个字段的组合 key 上对 Tuples 进行了分组。

嵌套 Tuples 的注意点： 如果你有一个嵌套 tuple 的 DataStream，比如：

{% highlight java %}
DataStream<Tuple3<Tuple2<Integer, Float>,String,Long>> ds;
{% endhighlight %}

指定 `keyBy(0)`，系统会使用整个 `Tuple2` 作为一个 key （Integer 和 Float 字段同时作为 key）。如果你想要“导航”到嵌套 `Tuple2` 里，你还得使用字段表达式，下面将会讲到。

### 使用字段表达式定义 Key
{:.no_toc}

你可以使用基于 String 的字段表达式来引用嵌套字段并定义 keys，用来做 grouping, sorting, joining, 或 coGrouping。

字段表达式使得在（嵌套的）组合类型中选择字段变得非常方便，比如 [Tuple](#tuples-and-case-classes) 和 [POJO](#pojos) 类型.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

下面的例子中，我们有一个 `WC` POJO，带有两个字段 "word" 和 "count"。要按照 `word` 字段来分组，我们只需要将它的名字传到 `groupBy()` 函数中。

{% highlight java %}
// some ordinary POJO (Plain old Java Object)
public class WC {
  public String word;
  public int count;
}
DataStream<WC> words = // [...]
DataStream<WC> wordCounts = words.keyBy("word").window(/*window specification*/);
{% endhighlight %}

**字段表达式语法**:

- 通过字段名来选择 POJO 中的字段。例如 `"user"` 就代表 WC POJO 中的 "user" 字段。

- 通过字段名或者从 0 偏移的字段索引来选择 Tuple 中的字段。例如 `"f0"` 和 `"5"` 就分别代表了一个 Java Tuple 类中的第一个和第六个字段。

- 你可以在 POJOs 和 Tuples 中选择嵌套的字段。例如，`"user.zip"` 代表了一个 POJO 类型中的 "user" 字段中的 "zip" 字段，其中 "user" 也是 POJO 类型。也支持任意嵌套以及 POJOs 与 Tuples 的混合，如 `"f1.user.zip"` 或 `"user.f3.1.zip"`。

- 你可以使用 `"*"` 通配符来选择全部类型。这对于不是 Tuple 和 POJO 类型的也同样适用。


**字段表达式实例**:

{% highlight java %}
public static class WC {
  public ComplexNestedClass complex; //nested POJO
  private int count;
  // getter / setter for private field (count)
  public int getCount() {
    return count;
  }
  public void setCount(int c) {
    this.count = c;
  }
}
public static class ComplexNestedClass {
  public Integer someNumber;
  public float someFloat;
  public Tuple3<Long, Long, String> word;
  public IntWritable hadoopCitizen;
}
{% endhighlight %}

对于上面的实例代码，这些字段表达式都是合法的：

- `"count"`: 指 WC 类中的 `count` 字段。

- `"complex"`: 递归选择了 POJO 类型 `ComplexNestedClass` 的所有字段。

- `"complex.word.f2"`: 选择了嵌套的 `Tuple3` 中的最后一个字段。

- `"complex.hadoopCitizen"`: 选择了这个 Hadoop `IntWritable` 类型的字段。

</div>
<div data-lang="scala" markdown="1">

下面的例子中，我们有一个 `WC` POJO，带有两个字段 "word" 和 "count"。要按照 `word` 字段来分组，我们只需要将它的名字传到 `groupBy()` 函数中。

{% highlight java %}
// some ordinary POJO (Plain old Java Object)
class WC(var word: String, var count: Int) {
  def this() { this("", 0L) }
}
val words: DataStream[WC] = // [...]
val wordCounts = words.keyBy("word").window(/*window specification*/)

// or, as a case class, which is less typing
case class WC(word: String, count: Int)
val words: DataStream[WC] = // [...]
val wordCounts = words.keyBy("word").reduce(/*window specification*/)
{% endhighlight %}

**字段表达式语法**:


- 通过字段名来选择 POJO 中的字段。例如 `"user"` 就代表 WC POJO 中的 "user" 字段。

- 通过从 1 偏移的字段名或者从 0 偏移的字段索引来选择 Tuple 中的字段。例如 `"_1"` 和 `"5"` 就分别代表了一个 Java Tuple 类中的第一个和第六个字段。

- 你可以在 POJOs 和 Tuples 中选择嵌套的字段。例如，`"user.zip"` 代表了一个 POJO 类型中的 "user" 字段中的 "zip" 字段，其中 "user" 也是 POJO 类型。也支持任意嵌套以及 POJOs 与 Tuples 的混合，如 `"_2.user.zip"` 或 `"user._4.1.zip"`。

- 你可以使用 `"_"` 通配符来选择全部类型。这对于不是 Tuple 和 POJO 类型的也同样适用。


**字段表达式实例**:

{% highlight scala %}
class WC(var complex: ComplexNestedClass, var count: Int) {
  def this() { this(null, 0) }
}

class ComplexNestedClass(
    var someNumber: Int,
    someFloat: Float,
    word: (Long, Long, String),
    hadoopCitizen: IntWritable) {
  def this() { this(0, 0, (0, 0, ""), new IntWritable(0)) }
}
{% endhighlight %}


对于上面的实例代码，这些字段表达式都是合法的：

- `"count"`: 指 WC 类中的 `count` 字段。

- `"complex"`: 递归选择了 POJO 类型 `ComplexNestedClass` 的所有字段。

- `"complex.word._3"`: 选择了嵌套的 `Tuple3` 中的最后一个字段。

- `"complex.hadoopCitizen"`: 选择了这个 Hadoop `IntWritable` 类型的字段。

</div>
</div>

### 使用 Key 选择器函数定义 Key
{:.no_toc}

还有一种定义 key 的方法是 "key 选择器"（key selector） 函数。Key 选择器函数接受一个单独的元素作为输入，并返回这个元素的 key。这个 key 可以是任何类型，也可以是任意计算结果。

下面这个例子展示了一个 key 选择器函数：简单地返回了一个对象的字段。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// some ordinary POJO
public class WC {public String word; public int count;}
DataStream<WC> words = // [...]
KeyedStream<WC> kyed = words
  .keyBy(new KeySelector<WC, String>() {
     public String getKey(WC wc) { return wc.word; }
   });
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// some ordinary case class
case class WC(word: String, count: Int)
val words: DataStream[WC] = // [...]
val keyed = words.keyBy( _.word )
{% endhighlight %}
</div>
</div>

{% top %}

定义转换函数
--------------------------

大部分转换都需要一个用户实现的函数。这一节就如何定义转换函数列出了一些不同的方法。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">


#### 实现接口

最常见的方式是实现一个 Flink 提供的接口：

{% highlight java %}
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
});
data.map(new MyMapFunction());
{% endhighlight %}


#### 匿名类

以匿名类的形式传入一个函数：

{% highlight java %}
data.map(new MapFunction<String, Integer> () {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

#### Java 8 Lambdas

Flink 在 Java API 中也支持了 Java 8 Lambdas。请参考完整的 [Java 8 指南]({{ site.baseurl }}/apis/java8.html)。

{% highlight java %}
data.filter(s -> s.startsWith("http://"));
{% endhighlight %}

{% highlight java %}
data.reduce((i1,i2) -> i1 + i2);
{% endhighlight %}

#### 富函数

所有转换中需要的用户自定义函数都可以被替换成*富*函数（rich function）作为参数。例如，代替

{% highlight java %}
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

你可以写成

{% highlight java %}
class MyMapFunction extends RichMapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

然后，与之前一样，将这个函数传到 `map` 转换中：

{% highlight java %}
data.map(new MyMapFunction());
{% endhighlight %}

富函数同样可以被定义成匿名类：
{% highlight java %}
data.map (new RichMapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

#### Lambda 函数

如在上述实例中看到的，所有操作都接受了一个 lambda 函数，用来描述操作的内容：

{% highlight scala %}
val data: DataSet[String] = // [...]
data.filter { _.startsWith("http://") }
{% endhighlight %}

{% highlight scala %}
val data: DataSet[Int] = // [...]
data.reduce { (i1,i2) => i1 + i2 }
// or
data.reduce { _ + _ }
{% endhighlight %}

#### 富函数

所有转换中需要的用户自定义函数都可以被替换成*富*函数（rich function）作为参数。例如，代替

{% highlight scala %}
data.map { x => x.toInt }
{% endhighlight %}

你可以写成

{% highlight scala %}
class MyMapFunction extends RichMapFunction[String, Int] {
  def map(in: String):Int = { in.toInt }
})
{% endhighlight %}

然后，与之前一样，将这个函数传到 `map` 转换中：

{% highlight scala %}
data.map(new MyMapFunction())
{% endhighlight %}

富函数同样可以被定义成匿名类：
{% highlight scala %}
data.map (new RichMapFunction[String, Int] {
  def map(in: String):Int = { in.toInt }
})
{% endhighlight %}
</div>
</div>

除了给用户自定义的函数外（如 map、reduce 等），富函数还提供了四种函数：`open`, `close`, `getRuntimeContext`，以及`setRuntimeContext`。这些对于参数化函数（参考 [给函数传参]({{ site.baseurl }}/apis/batch/index.html#passing-parameters-to-functions))，创建和销毁本地状态，访问广播变量（参考 [广播变量]({{ site.baseurl }}/apis/batch/index.html#broadcast-variables)），以及访问运行时信息如累加器和计数器（参考 [Accumulators 和 Counters](#accumulators--counters)），还有迭代信息（参考 [迭代]({{ site.baseurl }}/apis/batch/index.html#iteration-operators)）都是有很大帮助的。

{% top %}

支持的数据类型
--------------------

Flink 对于 DataSets 和 DataStream 中元素类型有一些限制。原因是系统需要分析这些类型来决定高效执行策略。

有 6 种不同的数据类型：

1. **Java 元组（Tuples）** 和 **Scala 样本类（Case Classes）**
2. **Java POJOs**
3. **基本类型（Primitive Types）**
4. **一般类（Regular Classes）**
5. **Values**
6. **Hadoop Writables**
7. **特殊类型（Special Types）**

#### 元组和样本类（Tuples 和 Case Classes）

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

Tuples 是由固定数量的字段组成，字段类型可以是多样化的。Java API 提供了从 `Tuple1` 到 `Tuple25` 。Tuple 中的每一个字段都可以是任意的 Flink 类型，可以是嵌套 tuple。Tuple 的字段可以直接通过字段名（如 `tuple.f4`）或者 getter 函数（如 `tuple.getField(int position)`）来访问。字段的起始索引是 0。注意，这个和 Scala tuples 不一样，但和 Java 的通用索引是一致的。

{% highlight java %}
DataStream<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));

wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public String map(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }
});

wordCounts.keyBy(0); // also valid .keyBy("f0")


{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

Scala 的样本类（case classes）以及 Scala 元组（tuples，一种特殊的样本类）是包含了数量固定的各种类型的字段的复合类型。元组不能通过名称获取字段，而是使用位置下标来读取对象；而且这个下标基于 1，而不是基于 0。样本类是通过名称来获得字段。

{% highlight scala %}
case class WordCount(word: String, count: Int)
val input = env.fromElements(
    WordCount("hello", 1),
    WordCount("world", 2)) // Case Class Data Set

input.keyBy("word")// key by field expression "word"

val input2 = env.fromElements(("hello", 1), ("world", 2)) // Tuple2 Data Set

input2.keyBy(0, 1) // key by field positions 0 and 1
{% endhighlight %}

</div>
</div>

#### POJOs

如果满足下列所有条件， Java 和 Scala 类就会被当做 POJO 类型：

- 类必须是 public 的

- 含有无参构造函数

- 所有的字段要么是 public 的，要么是可以通过 getter 和 setter 函数访问的。对于一个叫做 `foo` 的字段来说，getter 和 setter 方法必须是 `getFoo()` 和 `setFoo()`。

- 所有的字段类型都能被 Flink 支持。同时，Flink 使用 [Avro](http://avro.apache.org) 来序列化对象（如 `Date`）。

Flink 会分析 POJO 类型的结构，比如 POJO 的字段。所以 POJO 比一般类型更容易使用，而且 Flink 处理 POJO 类型会比一般类型更高效。

下面展示了一个带有两个 public 字段的 POJO 类型。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class WordWithCount {

    public String word;
    public int count;

    public WordWithCount() {}

    public WordWithCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}

DataStream<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2));

wordCounts.keyBy("word"); // key by field expression "word"

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
class WordWithCount(var word: String, var count: Int) {
    def this() {
      this(null, -1)
    }
}

val input = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2)) // Case Class Data Set

input.keyBy("word")// key by field expression "word"

{% endhighlight %}
</div>
</div>


#### 基本类型（Primitive Types）

Flink 支持所有的 Java 和 Scala 的基本类型，例如 `Integer`, `String`, 和 `Double`。

#### 一般类（General Class Types）

Flink 支持大部分的 Java 和 Scala 类（API 或自定义的）。对于一些不能被序列化的字段会有限制，比如文件指针，I/O 流，native
resources。一般遵守 Java Beans 惯例的类都是可以的。

Flink 会把没有被当做 POJO 类型（请参考上面的 POJO 要求）的都当做一般类来处理。Flink 会把它们看做黑盒子并且无法访问它们的内容（例如，高效排序）。一般类型会用序列化框架 [Kryo](https://github.com/EsotericSoftware/kryo) 来做序列化/反序列化。


#### Values

*Value* 类型定义了他们自己的序列化和反序列化方式。用户自己来实现 `org.apache.flinktypes.Value` 接口的 `read` 和 `write` 方法，而不是使用通用的序列化框架。当通用的序列化方式不高效时，选择 Value 类型是合理的。一个例子是一个数据类型以数组实现了一个稀疏向量（sparse vector）。我们知道这个数组里大部分都是零，我们可以为非零元素使用一种特殊编码，然而通用的序列化框架只会简单的写入所有的数组元素。

类似的，`org.apache.flinktypes.CopyableValue` 接口提供了手动内部克隆（clone）逻辑。


Flink 已经预定义了一批基本的 Value 类型（`ByteValue`, `ShortValue`, `IntValue`, `LongValue`, `FloatValue`, `DoubleValue`, `StringValue`, `CharValue`, `BooleanValue`）。这些类型充当了基本数据类型的可变类型：它们的值可以被修改，允许开发者重复使用这些对象来减轻 GC 的压力。


#### Hadoop Writables

你可以使用实现了 `org.apache.hadoop.Writable` 接口的类型。定义在 `write()` 和 `readFields()` 方法中的序列化逻辑会被用来做序列化。


#### 特殊类型（Special Types）

你可以使用特殊类型，包括 Scala 的 `Either`, `Option`, 和 `Try`。Flink 在 Java API 中自己实现了一个 `Either`，很像 Scala 的 `Either`，它代表一个两种可能类型的值，*Left* 或 *Right*。`Either` 在错误处理或是 operator 需要输出两种不同类型的记录的时候是非常有用的。

#### 类型擦除和类型诊断

*注意: 本节只与 Java 有关。*

Java 编译器会在编译后扔掉很多泛型类型信息，这就是 Java 中的*类型擦除*（type erasure）。也就是说在运行时，一个对象实例是不知道它的泛型类型的。举例来说，`DataStream<String>` 和 `DataStream<Long>` 的实例对 JVM 说看起来是一样的。


当 Flink 准备执行程序时（当 main 函数被调用），Flink 会请求类型信息。Flink Java API 会尝试重新构造被类型擦除扔掉的类型信息，并把他们显示存到数据集和 operator 中。你可以通过调用 `DataStream.getType()` 来获取它们。这个函数返回了一个 `TypeInformation` 实例，这是 Flink 内部用来表示类型的类。

类型诊断（type inferrence）有它自己的限制并且在一些情况下需要程序员的“配合”。举例来说， 从集合中创建的数据集的函数，比如 `ExecutionEnvironment.fromCollection()` ，在这些函数中， 用户可以传入一个用来描叙类型的参数。但一些泛型函数比如 `MapFunction<I, O>` 还需要一些额外的类型信息。

{% gh_link /flink-java/src/main/java/org/apache/flink/api/java/typeutils/ResultTypeQueryable.java "ResultTypeQueryable" %} 接口可以通过输入格式和函数来实现明确地告诉 API 他们的返回类型。函数的*输入类型*通常可以通过前一个操作的返回值进行判断。


执行配置
-----------------------

`StreamExecutionEnvironment` 中包含了 `ExecutionConfig`， `ExecutionConfig` 可以用来设定任务在运行期间的具体配置。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
ExecutionConfig executionConfig = env.getConfig();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
var executionConfig = env.getConfig
{% endhighlight %}
</div>
</div>

下面这些配置项都是可以用的：(加粗的是默认项)

- **`enableClosureCleaner()`** / `disableClosureCleaner()`. 默认情况下闭包清理器（closure cleaner）是打开的。闭包清理器会清除对匿名函数的环绕类（surrounding class）的无用引用。如果关闭闭包清理器，有可能一个匿名用户函数还在引用某个环绕类，这个类通常不是可序列化的（Serializable）。这样会导致序列化组件会抛出异常。

- `getParallelism()` / `setParallelism(int parallelism)` 设置任务的并发。

- `getNumberOfExecutionRetries()` / `setNumberOfExecutionRetries(int numberOfExecutionRetries)` 设置失败 task 的重新执行次数。为 0 表示关闭容错。－1 表示使用系统默认值。

- `getExecutionRetryDelay()` / `setExecutionRetryDelay(long executionRetryDelay)` 当一个任务失败时，设置重新运行前的等待时间，单位毫秒。延期（delay）会在 TaskManager 上所有 task 都被成功停止后才开始计时。一旦设置的延期时间到了，所有 task 开始重新运行。这个参数用来延迟重新执行是非常有用的，目的是让某些超时相关的故障场景完全浮出水面（比如断了的连接还没有完全超时）。如果立即重试，容易导致相同的故障。这个参数只有当重试执行次数大于等于 1 时，才有效。

- `getExecutionMode()` / `setExecutionMode()`. 默认执行模式是 PIPELINED。 设置程序的执行模式， 这决定了数据交换是采用 batch 还是 pipeline 的方式。

- `enableForceKryo()` / **`disableForceKryo()`**. 默认没有使用 Kryo。强制 GenericTypeInformation 对 POJO 使用 Kryo 序列化器，即使 Flink 可以识别它们为 POJO。某些情况下推荐使用这种方式，举例来说，当 Flink 内部序列化器不能正确处理 POJO 时。

- `enableForceAvro()` / **`disableForceAvro()`**. 默认强制 Avro 是关闭的。在序列化 Avro POJO 时，强制 AvroTypeInformation 使用 Avro 序列化器代替 Kryo。

- `enableObjectReuse()` / **`disableObjectReuse()`**. 默认对象是不会被重复使用的。开启对象重用，会在运行时重新使用用户的对象，这能获得更好的性能。注意如果用户在定义操作的函数时并不清楚这一行为的话，可能到导致 bug。

- **`enableSysoutLogging()`** / `disableSysoutLogging()`. 默认 Jobmanager 的状态更新会打印到 `System.out`。可以通过这个配置关闭这种行为。

- `getGlobalJobParameters()` / `setGlobalJobParameters()` 这个函数可以设置一个自定义的对象作为任务的全局配置。 因为 `ExecutionConfig` 在所有用户定义的函数中都是可以访问到的，这是设置全局配置的一种方便方式。

- `addDefaultKryoSerializer(Class<?> type, Serializer<?> serializer)` 为一个指定类型（`type`）注册一个 Kryo 序列化实例。

- `addDefaultKryoSerializer(Class<?> type, Class<? extends Serializer<?>> serializerClass)` 为一个指定类型（`type`）注册一个 Kryo 序列化类。

- `registerTypeWithKryoSerializer(Class<?> type, Serializer<?> serializer)` 为一个指定的类型注册 以 Kryo 序列化实例。通过注册类型到 Kryo，这个类型的序列化会更高效。

- `registerKryoType(Class<?> type)`  如果这个类型最终是用 Kryo 序列化的，那么它会在 Kryo 被注册以确保只有标签（整型 ID）会被写入。如果一个类型不是使用 Kryo 注册的，那么它的整个类名（class-name）会被序列化到每个实例中，这会导致更高的 I/O 开销。

- `registerPojoType(Class<?> type)` 用序列化堆栈来注册给定的类型。如果这个类型最终是以 POJO 来序列化的，那么这个类型会使用 POJO 序列化器注册。如果这个类型最终是用 Kryo 被序列化，那么它会在 Kryo 被注册以确保只有标签（整型 ID）会被写入。如果一个类型不是使用 Kryo 注册的，它整个类名（class-name）会被序列化到每个实例中，这会导致更高的 I/O 开销。

注意使用 `registerKryoType()` 注册的类型是不能用于 Flink 的 Kryo 序列化实例的。

- `disableAutoTypeRegistration()` 自动类型注册默认是开启的。"自动类型注册"存储了用户代码中用到的所有类型（包括子类型）。


在 `Rich*` 函数中可以通过 `getRuntimeContext()` 方法获得 `RuntimeContext`，从中可以访问到 `ExecutionConfig`。

{% top %}


程序打包和分布式执行
-----------------------------------------

如前所述，Flink 程序使用一个 `remote enviroment` 可以运行在远程的集群环境中。同样的，程序可以打包成 JAR 来执行。程序的打包是通过 [命令行接口]({{ site.baseurl }}/apis/cli.html) 执行它们的前提条件。


#### 打包程序

为了支持通过命令行或 web 接口执行一个打包的 JAR 文件，一个程序必须使用通过 `StreamExecutionEnvironment.getExecutionEnvironment()` 获得的环境。当 JAR 通过命令行或 web 接口提交上去后，这个环境就会是集群模式环境。如果 Flink 程序不是通过这些接口来调用，这个环境就是本地模式环境。


为了打包程序，可以简单地导出所有涉及到的类到一个 JAR 文件中。这个 JAR 文件的 manifest 必须指向程序的*入口类*。最简单的方式是将 *main-class* 入口加到 mainifest 中（如 `main-class: org.apache.flinkexample.MyProgram`）。 这个 *main-class* 属性和 JVM 通过命令行 `java -jar pathToTheJarFile` 执行一个 JAR 文件找到 main 入口函数是一样的。大部分的 IDE 都提供导出 JAR 时自动添加入口函数。


#### 通过计划打包程序

Flink 还支持通过计划（*Plans*）来打包程序。计划打包返回的是 *Program Plan*（这是一个对程序数据流的描述），而不是在 main 函数中定义程序并在环境上调用 `execute()`。因此，程序必须实现 `org.apache.flink.api.common.Program` 接口，定义 `getPlan(String...)` 方法。传递给这个函数的 String 列表就是命令行参数。这个程序的计划可以通过 `ExecutionEnvironment#createProgramPlan()` 来创建出来。当打包这个程序的计划时，这个 JAR 的 mainfest 必须指向实现 `org.apache.flinkapi.common.Program` 接口的类，而不是带 main 入口函数的类。


#### 总结

调用一个打包的程序的整体流程如下：

1. 搜索 JAR 的 manifest 找到 *main-class* 或 *program－class*。如果这两个属性都被找到了，则 *program-class* 属性优先考虑。 当 JAR 的 manifest 两者都不包含时，命令行和 web 接口都支持输入参数来手动指定入口类名。

2. 如果入口类实现了 `org.apache.flinkapi.common.Program`，则系统会调用它的 `getPlan(String...)` 来获得程序计划并执行。

3. 如果入口类没有实现 `org.apache.flinkapi.common.Program`，则系统调用入口类的 main 函数。

{% top %}


累加器和计数器
---------------------------
累加器（Accumulators）由一个 **add 操作** 和一个 **final 累加结果** 组成，当任务结束后，才能获得累加结果。

最直接的累加器就是**计数器**（counter）。用户可以调用 `Accumulator.add(V value)` 来做累加。任务结束后， Flink 会合并所有的结果并将总结果返回给客户端。累加器在调试时很有作用，另外如果想快速知道数据的更多信息时也很有作用。

Flink 现在有如下的**内置累加器**， 他们都实现了 {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java "Accumulator" %} 接口：

- {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/IntCounter.java "__IntCounter__" %},
  {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/LongCounter.java "__LongCounter__" %}
  和 {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/DoubleCounter.java "__DoubleCounter__" %}:
  请看下方的例子了解如何使用计数器（counter）。
- {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Histogram.java "__Histogram__" %}: Histogram 实现了离散数据的分布。内部实现上它只是一个从 Integer 到 Integer 的 map。你可以使用这个去计算值的分布，例如 word count 程序中每行单词数的分布。

__如何使用累加器:__

首先你需要在自定义的转换函数中创建一个累加器（这里是一个计数器）。

{% highlight java %}
private IntCounter numLines = new IntCounter();
{% endhighlight %}

然后，注册这个累加器对象，一般是在富函数（*rich* function）中的 `open()` 函数内， 如下，可以定义一个名字。

{% highlight java %}
getRuntimeContext().addAccumulator("num-lines", this.numLines);
{% endhighlight %}

现在，你可以在这个 operator 函数的任何地方调用累加器了，包括`open()`和`close()`方法。

{% highlight java %}
this.numLines.add(1);
{% endhighlight %}

最终结果会存储在由执行环境（execution environment）的 `execute()` 函数返回的 `JobExecutionResult` 对象中（当前这只在执行等待作业完成的情况下有效）。

{% highlight java %}
myJobExecutionResult.getAccumulatorResult("num-lines")
{% endhighlight %}

一个任务的所有的累加器共享一个命名空间。因此用户可以在不同的 operator 函数中使用相同的累加器。Flink 内部会合并所有的相同名字的累加器。

关于累加器和迭代器的一个注意点：当前累加器的结果只能在任务结束后才能获得。我们计划实现前一个迭代的结果可以在下一个迭代中获得。用户可以使用 {% gh_link /flink-java/src/main/java/org/apache/flink/api/java/operators/IterativeDataSet.java#L98 "Aggregators" %} 去计算每个迭代的统计，然后基于统计结果来结束迭代。


__自定义累加器:__

用户可以通过实现 `Accumulator` 接口方便地创建自己的累加器。如果想要把自己实现的累加器加入到 Flink，请随意创建一个 pull request。

你可以选择是实现
{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java "Accumulator" %}
还是 {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/SimpleAccumulator.java "SimpleAccumulator" %}。

```Accumulator<V,R>``` 非常灵活: 它定义了需要增加的值的类型 ```V```, 和最终返回结果的类型 ```R```。 例如，对于一个 histogram 来说，```V```  是一个数字，```R``` 是一个 histogram。```SimpleAccumulator``` 就是 V 和 R 是同一种类型，比如计数器。

{% top %}


并发执行
------------------

这个章节将介绍在 Flink 中如何设置并发执行。 一个 Flink 程序由多个 task (transformation/operator, data sources, sinks)组成。一个 task 会切分成几个实例中并发执行，每一个并发实例处理 task 输入数据的一个子集。task 的并发实例数称为 *parallelism*（并行度）。

一个 task 的并行度在 Flink 的不同级别中指定。

### Operator 级别

独立的 operator，data source，data sink 可以通过调用 `setParallelism()` 来设置。例如：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = text
    .flatMap(new LineSplitter())
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5);

wordCounts.print();

env.execute("Word Count Example");
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

val text = [...]
val wordCounts = text
    .flatMap{ _.split(" ") map { (_, 1) } }
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5)
wordCounts.print()

env.execute("Word Count Example")
{% endhighlight %}
</div>
</div>


### 执行环境级别

就像[这里](#anatomy-of-a-flink-program)提到过的，Flink 程序是在一个执行环境的上下文中运行的。一个执行环境为所有 operator，data source，data sink 定义了一个默认的并行度。执行环境的并行度可以被 operator 的并行度覆盖。

一个执行环境的默认并行度可以通过调用 `setParallelism()` 方法指定。要以并行度为 `3` 来执行所有的 operators, data sources, data sinks，可以如下这样设置执行环境的默认并行度：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(3);

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = [...]
wordCounts.print();

env.execute("Word Count Example");
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(3)

val text = [...]
val wordCounts = text
    .flatMap{ _.split(" ") map { (_, 1) } }
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1)
wordCounts.print()

env.execute("Word Count Example")
{% endhighlight %}
</div>
</div>


### 客户端级别

当提交任务到 Flink 时，客户端可以设置并行度。这个客户端可以是一个 Java 或 Scala 程序。 Flink 的命令行客户端(Command-line interface CLI)就是这样一个例子。

在 CLI 客户端中，可以通过 `-p` 参数来设置并行度，例如：


    ./bin/flink run -p 10 ../examples/*WordCount-java*.jar


在 Java/Scala 程序中，并行度可以如下这样设置：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

try {
    PackagedProgram program = new PackagedProgram(file, args);
    InetSocketAddress jobManagerAddress = RemoteExecutor.getInetFromHostport("localhost:6123");
    Configuration config = new Configuration();

    Client client = new Client(jobManagerAddress, config, program.getUserCodeClassLoader());

    // set the parallelism to 10 here
    client.run(program, 10, true);

} catch (ProgramInvocationException e) {
    e.printStackTrace();
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
try {
    PackagedProgram program = new PackagedProgram(file, args)
    InetSocketAddress jobManagerAddress = RemoteExecutor.getInetFromHostport("localhost:6123")
    Configuration config = new Configuration()

    Client client = new Client(jobManagerAddress, new Configuration(), program.getUserCodeClassLoader())

    // set the parallelism to 10 here
    client.run(program, 10, true)

} catch {
    case e: Exception => e.printStackTrace
}
{% endhighlight %}
</div>
</div>

### 系统级别

可以在配置文件 `./conf/flink-conf.yaml` 中设置 `parallelism.default`，从而设置系统范围的默认并行度。查看 [配置]({{ site.baseurl }}/setup/config.html) 文档了解更多。

{% top %}


执行计划
---------------

根据各种参数，比如数据大小、集群机器数，Flink 的优化器会自动为你的程序选择一种执行策略。在很多情况下，对于了解 Flink 究竟是如何执行你的程序是很有帮助的。


__计划可视化工具__

Flink 自带了执行计划的可视化工具，可视化工具位于 `tools/planVisualizer.html`。它接受一个代表任务执行计划的 JSON 数据，并将它展示成一个带执行策略注释的图。

下面代码展示如何打印执行计划 JSON:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

...

System.out.println(env.getExecutionPlan());
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

...

println(env.getExecutionPlan())
{% endhighlight %}
</div>
</div>

要可视化执行计划，需要按以下步骤：

1. **打开** 用浏览器打开 ```planVisualizer.html```,
2. **粘贴** JSON 字符串到文本框中, 然后
3. **点击** draw 按钮。

之后，一个详细的执行计划图就会被展示出来

<img alt="A flink job execution graph." src="fig/plan_visualizer.png" width="80%">


__Web 接口__

Flink提供 web 接口用来提交和执行任务。如果你选择使用 web 接口提交打包的程序，你可以选择看到计划图。

启动 web 接口的脚本位于 `bin/start-webclient.sh`。启动 webclient（8080端口）后，你可以上传程序，上传的程序会列在左边可使用的程序列表中。

你也可以在页面底部的文本框中指定程序的输入参数。选中了 plan visualization 选项框的话，在执行程序前，会展示执行计划图。

{% top %}
