---
title: "Basic Concepts"

# Top-level navigation
top-nav-group: apis
top-nav-pos: 1
top-nav-title: <strong>Basic Concepts</strong>

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


# 基本概念

flink 程序就是在分布式集合（collection）上实现transformation（比如filtering, mapping, updating state, joining, grouping, defining windows, aggregating）的普通程序. 数据集合根据一些source（比如 文件， kafka或本地的集合）初始化创建。 通过sinks返回的结果，sink可以写书到到（分布式）文件或标准输出（如终端命令行）。flink 运行在各种context下，standalone或潜入到其他程序中。 可能在本地jvm中执行，或多台机器的集群上运行。

根据data source的类型， 固定大小或无穷无尽的source， 用户可以实现一个批处理(batch program)或一个流式程序(streaming program), 从而DataSet API 用于前者，而DataStream API 用于后者。 本文档介绍2种api 都需要的基本概念。

<b>注意</b>: 在本文档介绍api使用的example中， 我们将使用 StreamingExecutionEnvironment和DataStream API, 这些概念同样适用于DataSet api, 仅仅是换成ExecutionEnvironment and DataSet。

---
<a name="top"></a>

* <a href="#link">Linking with Flink</a>
* <a href="#DataSet-and-DataStream">DataSet andDataStream</a>
* <a href="#skeleton">程序框架</a>
* <a href="#lazy">延迟执行</a>
* <a href="#specifying-keys">Specifying keys</a>
* <a href="#Specifying-Transformation-Functions">指定转换函数</a>
* <a href="#Supported-Data-Types">数据类型</a>
* <a href="#execution-configuration">执行配置</a>
* <a href="#Program-Packaging-and-Distributed-Execution">程序打包和分布式执行</a>
* <a href="#Accumulators-Counters">累加器和计数器</a>
* <a href="#Parallel-Execution">并发执行</a>
* <a href=“#Execution-Plans”>执行计划</a>
--- 

<a name="link"><a>
# Linking with Flink

最简单link flink到项目的方式是使用 快速入门的脚本，比如[Java API](java-api-quickstart)或[Scala API](scala-api)。也可以通过下面maven插件来创建一个空的项目，archetypeVersion 可以选择其他稳定版本或snapshot版本
```
mvn archetype:generate \
    -DarchetypeGroupId=org.apache.flink \
    -DarchetypeArtifactId=flink-quickstart-java \
    -DarchetypeVersion=1.1-SNAPSHOT
```
```
mvn archetype:generate \
    -DarchetypeGroupId=org.apache.flink \
    -DarchetypeArtifactId=flink-quickstart-scala \
    -DarchetypeVersion=1.1-SNAPSHOT
```

如果想要把flink加入到现有的maven项目中，则需要增加依赖：
```
<!-- Use this dependency if you are using the DataStream API -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-java_2.10</artifactId>
  <version>1.1-SNAPSHOT</version>
</dependency>
<!-- Use this dependency if you are using the DataSet API -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-java</artifactId>
  <version>1.1-SNAPSHOT</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients_2.10</artifactId>
  <version>1.1-SNAPSHOT</version>
</dependency>
```
```
<!-- Use this dependency if you are using the DataStream API -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala_2.10</artifactId>
  <version>1.1-SNAPSHOT</version>
</dependency>
<!-- Use this dependency if you are using the DataSet API -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala_2.10</artifactId>
  <version>1.1-SNAPSHOT</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients_2.10</artifactId>
  <version>1.1-SNAPSHOT</version>
</dependency>
```
注意：如果使用scala api，得二选一import下面：
```
import org.apache.flink.api.scala._
```

```
import org.apache.flink.api.scala.createTypeInformation
```
原因是flink需要分析程序中使用的类型，从而生成出serializers和comparaters。 硬刺程序需要import 他们之一，从而能过enable一个非明确的转化，为flink的一些操作创建类型信息。
## scala 依赖版本
因为scala 2.10 binary 不兼容2.11，因此flink提供多个版本为不通scala version。从0.10开始，flink社区交叉编译了所有的flink 模块为2.10 和2.11. 只需要对应版本的`artifactId`后加上版本信息，比如"_2.11".  当然也可以自己[编译flink](https://ci.apache.org/projects/flink/flink-docs-release-0.10/setup/building.html).

## hadoop 版本依赖
如果flink 运行在hadoop上，请选择正确的hadoop版本，详情请参考[downloads](http://flink.apache.org/downloads.html) 

flink-clients 依赖仅仅是本地模式需要。 如果想要在集群模式下运行， 可以忽略这个依赖。

<a href="#top">返回顶部</a>

<a name="DataSet-and-DataStream"></a>
# DataSet and DataStream

Flink 用特别的类 DataSet和DataStream 来表示程序中的数据。 用户可以认为它们是含有重复数据的不可修改的集合(collection).  当使用DataSet时， 数据是有限的， 而DataStream 中元素的数量是无限的。

在几个关键点上， 这些集合不同于java collections。 首先， 它们是不可修改的， 意味着一旦被创建出来， 用户不能添加或删除元素， 也不能简单检查元素内部。

在一个flink程序中， 添加一个source来初始化一个collection， 可以通过transformation 这些collection得到一个新的collection， transformation的api函数比如 map， filter 等。

<a href="#top">返回顶部</a>

<a name="skeleton"></a>
# 程序框架
就像example所示，  flink DataSet程序含有java常有的main函数， 还包括一些其他的逻辑：
* 获得一个ExecutionEnvironment
* 加载或创建 原始数据
* 确定数据上至下的transformations
* 确定计算结果的存储地方
* 启动程序执行。

我们将展示每一步， 所有的核心class java api都在[org.apache.flink.api.java](https://github.com/apache/flink/blob/master//flink-java/src/main/java/org/apache/flink/api/java)。


ExecutionEnvironment是所有flink DataSet程序的基础。 可以如下来创建它：
```
getExecutionEnvironment()

createCollectionsEnvironment()

createLocalEnvironment()
createLocalEnvironment(int parallelism)
createLocalEnvironment(Configuration customConfiguration)

createRemoteEnvironment(String host, int port, String... jarFiles)
createRemoteEnvironment(String host, int port, int parallelism, String... jarFiles)

```
一般来说， 用户者需要调用getExecutionEnvironment， 因为flink会根据不同的上下文来做对应的事情，比如程序运行在一个ide中， 或者在本地机器上执行程序；如果创建一个jar 文件， 并通过[commandline client](command-client)或[web interface](web-client)来提交任务， flink集群的manager会执行程序的main函数并调用getExecutionEnvironment()得到一个为运行在集群上运行你程序的一个执行环境（execution environment）。

为了去定data source， 执行环境有几种方式读取文件： 可以一行一行来读取，比如csv文件， 或使用自定义的输入格式。 可以通过下列方式，来按行来读取一个text文件：
```
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<String> text = env.readTextFile("file:///path/to/file");
```
这用用户就可以得到一个可以做各种transformation的DataSet. 可以参考<a href="#data-source">Data Sources</a>.

可以使用transformation来转化一个DataSet得到一个新的DataSet, 可以将新的可以使用transformation来转化一个DataSet得到一个新的DataSet写到文件，或再做一次transformation 或和其他可以使用transformation来转化一个DataSet得到一个新的DataSet 合并。 例如：
一个map的transformation：
```
val input: DataSet[String] = ...

val mapped = input.map { x => x.toInt }
```
这个transformation将通过将所有的string转化为integer来创建一个新的DataSet。 可以参考<a href="#transformation">Transformations</a>来进一步了解transformation。

一旦获得最终的DataSet， 用户可以把它写到文件或打印出来
```
def writeAsText(path: String, writeMode: WriteMode = WriteMode.NO_OVERWRITE)
def writeAsCsv(
    filePath: String,
    rowDelimiter: String = "\n",
    fieldDelimiter: String = ',',
    writeMode: WriteMode = WriteMode.NO_OVERWRITE)
def write(outputFormat: FileOutputFormat[T],
    path: String,
    writeMode: WriteMode = WriteMode.NO_OVERWRITE)

def printOnTaskManager()

def print()

def collect()
```
第三个函数可以指定一个文件输出格式。 可以参考<a href="#data-sink">Data Sinks</a>来获得更多sink相关资料。

print（）函数很方便用来debug和开发。 输出DataSet内容到标准输出（在启动flink执行的jvm上）。 注意： 现在的print行为和0.9.x的print行为是不同的， 在0.9.x中，它是输出到worker的日志文件中， 现在是把DataSet发送到客户端并在那里打印。

collection（）是从集群上收集DataSet到本地jvm， 它将返回一个list。

print和collection都会触发程序执行， 用户无需手动调用execute.

注意： print和collect 都会从集群上获取数据到客户端， 当前， collection收集的数据大小会受限于rpc系统，建议不要超过10MB.

printOnTaskManager 可以打印DataSet 内容到taskmanager， 但这个函数不会触发程序执行。

一旦完成程序，用户需要启动程序执行，可以直接调用ExecutionEnviroment的execute， 也可以间接调用通过collect或print。 ExecutionEnvironment将会决定运行在本地模式或集群上。

注意，不能在程序的最后调用collect／print或execute。

execute()函数会返回JobExecutionResult， 包含执行的次数和计算结果。 print和collection 并不会返回结果，但可以通过getLastJobExecutionResult来获得结果。

<a href="#top">返回顶部</a>

<a name="lazy"></a>
# 延迟执行

所有的flink程序都是延迟执行， 当执行程序main函数时， 加载数据但transformation并没有马上执行。相反，创建每一个操作并加到程序执行计划中。当调用ExecutionEnvironment.execute（collect／print也会触发这些操作）时，这些操作才真正执行，不管程序是本地执行还是分布式执行。


<a href="#top">返回顶部</a>

<a name="specifying-keys"></a>
# Specifying Keys
一些transformation（比如join，coGroup）邀请它参数的DataSet 定一个key。 另外一些transformation(比如Reduce, GroupReduce, Aggregate)会对他们用的某个key做group操作
```
DataSet<...> input = // [...]
DataSet<...> reduced = input
  .groupBy(/*define key here*/)
  .reduceGroup(/*do something*/);
```
flink的数据模型并不是基于key-value pair模型。 因此，用户不需要把数据类型转换成key／values。 key实际上是“虚拟”的， 他们被定义到实际数据上的function 来帮助一些grouping操作。

## 为tuple 定义key
最简单的例子：在一个或多个field上对一组tuples进行grouping：
```
DataSet<Tuple3<Integer,String,Long>> input = // [...]
DataSet<Tuple3<Integer,String,Long> grouped = input
  .groupBy(0)
  .reduceGroup(/*do something*/);
```
对第一个filed进行group操作， GroupReduce函数会接受到第一个field相同值的tuples。 下例是组合field。<>
```
DataSet<Tuple3<Integer,String,Long>> input = // [...]
DataSet<Tuple3<Integer,String,Long> grouped = input
  .groupBy(0,1)
  .reduce(/*do something*/);
```
对于复杂的tuple：
```
DataSet<Tuple3<Tuple2<Integer, Float>,String,Long>> ds;
```
如果groupby(0), 则会使用完整的Tuple2  作为key。 如果想进一步到Tuple2内部， 需要使用filed 表达式， 后面会介绍。

## filed 表达式
从0.7 开始， 用户可以使用基于string的filed表达式来引用嵌套filed， 用于grouping／sorting／joining／coGrouping. 除此之外， filed 表达式可以用来定义 <a href="#annotaions">sematic function annotations</a>
```
// some ordinary POJO (Plain old Java Object)
public class WC {
  public String word; 
  public int count;
}
DataSet<WC> words = // [...]
DataSet<WC> wordCounts = words.groupBy("word").reduce(/*do something*/);
```
filed 表达式语法：

* 选择pojo对象的成员作为filed name。 比如上例
* 用filed name or 基于0的offset field index。 比如："f0" 和“5” 代表第一个和第6个field.
* 使用嵌套filed。 举例 “user.zip” 代表"user" pojo的“zip” filed。并且支持和其他方式混用，比如"f1.user.zip"/"user.f3.zip"
* 可以使用"*"来选择所有的类型。 可以用于既不是tuple又不是pojo的类型。

例子：
```
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
```
* "count": 指WC的count 字段
* "complex": 指pojo ComplexNestedClass的所有字段
* "complex.word.f2",  最后一个字段在Tuple3.
* "complex.hadoopCitizen": IntWritable 类型。

## key 选择函数
还有一种定义key的方法是"key selector"函数。 key 选择函数 用一个单独的dataset 元素作为输入， 输出这个元素对应的key。 这个key可以是任何类型， 也可以是任意计算结果。
```
// some ordinary POJO
public class WC {public String word; public int count;}
DataSet<WC> words = // [...]
DataSet<WC> wordCounts = words
                         .groupBy(
                           new KeySelector<WC, String>() {
                             public String getKey(WC wc) { return wc.word; }
                           })
                         .reduce(/*do something*/);
```

<a href="#top">返回顶部</a>

<a name="Specifying-Transformation-Functions"></a>
# 指定转换函数

##  实现一个接口
最常见的是实现一个flink 指定的接口：
```
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
});
data.map (new MyMapFunction());
```

## 匿名类
```
data.map(new MapFunction<String, Integer> () {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```

## java 8 lambda
```
DataSet<String> data = // [...]
data.filter(s -> s.startsWith("http://"));
DataSet<Integer> data = // [...]
data.reduce((i1,i2) -> i1 + i2);
```
## rich 函数
```
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```
可以改成：
```
class MyMapFunction extends RichMapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```
所有使用用户定义函数的transformations都可以换成采用rich 函数作为参数的transformation。
因此， 可以传递这个函数给map transformation：
```
data.map(new MyMapFunction());
```
rich 函数也可以使用匿名类：
```
data.map (new RichMapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```

除了用户自定义的函数map/reduce， rich 函数还提供4种函数：open/close/getRuntimeContext/setRuntimeContext. 当参数话这些函数时， 这些rich函数非常有用(<a href="passing-parameter-to-function"> Passing Parameters to Functions </a>), 创建／结束local state， 访问广播变量(<a href="#broadcast-variables">Broadcast Variable</a>, 访问runtime 信息 如accumulators/counters/iterations).

对于reduceGroup, rich 函数是唯一的定义一个可选combine函数的方式。 

<a href="#top">返回顶部</a>

<a name="Supported-Data-Types"></a>
# 数据类型

DataSets中元素和transformation返回的结果 的类型会有限制。 原因是系统需要分析这些类型来决定高效执行策略。

6种不同的分类：
* java Tuples/Scala Case Classes
* java POJO
* 基本类型(Primitive Types)
* 基本类(Regular Classes)
* Values
* Hadoop Writables

## Tuples and Case Classes
Tuples 是由固定数量的field组成的。 java api 提供Tuple1 到Tuple25. tuple的任意一个field可以任意类型，可以是嵌套tuple。 tuple的field可以直接用类似"tuple.f4"或者getter函数如tuple.getField(int position)来访问。 起始index是0. 注意， 这个和scala的tuples不一样，但和java的通用index是一致的。
```
DataSet<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));

wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public String map(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }
});
```
当grouping／sorting／joining 一组tuples， 可以用field index或express作为key， 可以参考 <a href="#transformation">Transformations</a> 和<a href="#specifying-keys">Specifying keys</a>

## POJO
如果满足下列条件， java 或scala 类就会被当做POJO:
* class 必须是public
* 含有无参构造函数
* 所有的field是public或可以通过getter／setter访问。
* 所有的fileld 类型是flink支持。 同时， flink使用avro来序列号对象。

flink可以分析pojo的类型，分析它的field。 最终pojo 比一般类型更容易使用，而且flink 会比一般类型更高效的处理pojo。
```
public class WordWithCount {

    public String word;
    public int count;

    public WordCount() {}

    public WordCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}
```
## 基本类型(Primitive Types)
java/scala 基本类型，如Integer/String/Double.

## 基本类（general class）
flink支持大部分的java/scala 类。 一般遵守java Beans 惯例的class 都可以， 对于一些不能序列化的filed 比如文件指针／io streams/native resource 的filed 会有限制。

并不是所有的类会当做pojo类型， 当做general的类会被认为黑盒子，并且不会访问他们的内容， 会用kryo来序列化或反序列化general class。

## Values
value 类型定义了他们自己的序列化和反序列化方式。 用户自己来实现"org.apache.flinktypes.Value"接口read／write。当通用的序列化方式不高效时，选择value 类型是合理的。 比如， 一个实现sparse vector（元素是array）作为数据类型 。

“org.apache.flinktypes.CopyableValue”提供手动clone 逻辑。

flink已经预定义了一批基本的value类型（ByteValue, ShortValue, IntValue, LongValue, FloatValue, DoubleValue, StringValue, CharValue, BooleanValue）. 这些类型是一些mutable的基本类型。 他们的值可以被修改，允许开发者重复使用这些对象，减轻gc的压力。

## Hadoop Writables
用户可以使用实现"org.apache.hadoop.Writable" 的类型。

## 类型消除和类型诊断（type erasure & type inference）
编译后， java编译器会扔掉很多基本的类型信息，这就是java中的type erasure。 这意味着， 在runtime时，一个object的instance 的基本类型并不知道， 比如DataSet<String>和DataSet<Long>的instance， 对jvm来说，就是一样的。

flink 会请求类型信息，当它准备执行程序时（当main函数已经被调用）。flink java api会尝试构建扔掉的类型信息并把他们显示存到数据集和operator中。 用户可以通过调用DataSet.getType()来获取它们。 这个函数返回TypeInformation 对象，以flink内在方式来表示类型。

type inferrence有它自己的限制并且在一些情况下需要程序员的配合。 举例来说， 从集合中创建的数据集的函数，比如ExecutionEnvironment.fromCollection(), 在这些函数中， 用户可以传递一些描叙类型的参数。 但一些常见函数比如MapFunction<I, O>需要一些额外的类型信息。

通过明确输入格式和函数返回类型来实现ResultTypeQueryable接口。 函数的输入类型通常可以通过前一个操作返回值进行判断。

## 重复使用object
flink 会尽量减少申请object来获得更好的性能。

默认情况下， 用户定义的函数比如map／groupReduce 会在每次调用时获得新的对象。因此有可能在这些函数内部，保持这些对象的引用。

经常把用户定义的函数串联起来， 举例来说， 定义2个相同并发的mapper， 一个接上另外一个。 在这种串联情况下， 串联的函数都是接受相同的对象。 因此，第二个map函数接受第一个map返回的对象。当第一个函数保留所有对象为一个list，而第二个map会去修改这些对象时，这种情况会引发错误。这种情况下， 用户需要手动创建对象的copy在把它们放到list之前。

注意， 系统是假设用户在filter函数中不会修改输入的object。

ExectionConfig 有一个开关， 它可以允许用户打开 object resue 模式（enableObjectReuse()）。 对于易变类型， flink会重复使用对象。 这样意味着一个map函数总是接受到相同的对象，单它的字段设置为新的值。 对象复用模式会产生更好的性能，因为申请更少的对象， 但用户需要小心处理他们正使用的对象。


<a href="#top">返回顶部</a>

<a name="execution-configuration"></a>
# 执行配置
ExecutionEnvironment 包含ExecutionConfig， 从而可以设定任务在运行期间的特殊的配置。
```
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
ExecutionConfig executionConfig = env.getConfig();
```
下面一些配置项(可以参考源码下面的注释， 源码下面ExecutionConfig有更多的配置项)：
* enableClosureCleaner() / disableClosureCleaner(). 默认情况是 closure cleaner（关闭清理器）是打开的。 关闭清理器会清除对匿名函数内部的surrounding类的引用。如果关闭关闭清理器， 有可能一个用户匿名函数还在引用一个surrounging 类， 这个类通常不是Serializable， 这样会导致序列化组件会出现exception。
* getParallelism() / setParallelism(int parallelism) 设置任务的并发。
* getNumberOfExecutionRetries() / setNumberOfExecutionRetries(int numberOfExecutionRetries) 设置失败task重新执行次数。 为0 表示关闭容错。 －1 表示使用系统默认值。
* getExecutionRetryDelay() / setExecutionRetryDelay(long executionRetryDelay)   设当一个任务失败时， 在重新运行前，设置等待多少个毫秒。 当所有的task被taskmanager成功停止后， 才开始计时delay， 一旦设置的delay时间到了， 所有task开始重新运行。这个参数非常有用， 对于一些time-out 相关的failure场景， 比如一些broken的连接并没有完全timeout， 如果立即重试，容易导致相同的故障。这个参数只有当重试执行次数大于等于1时，才有效。
* getExecutionMode() / setExecutionMode(). 默认执行模式是PIPELINED。 设置程序的执行模式， 决定了数据交换是采用batch或pipeline的方式。
* enableForceKryo() / disableForceKryo, 默认没有使用kryo。 设置GenericTypeInformation 对pojo使用kryo序列化起，即使flink可以识别它们为pojo。 某些情况下推荐使用这种方式， 举例， 当flink 内部序列化器不能正确处理pojo。
* enableForceAvro() / disableForceAvro(),  默认avro关闭。 设置AvroTypeInformation 使用avro序列化器当序列化avro pojo。
* enableObjectReuse() / disableObjectReuse(), 默认object不会被重复使用。参考<a href="#types">数据类型</a>的重复使用章节。
* enableSysoutLogging() / disableSysoutLogging() 默认jobmanager的状态更新会打印到System.out, 这些函数可以对这个进行设置。
* getGlobalJobParameters() / setGlobalJobParameters(), 这个函数可以设置一个自定义的对象作为任务的全局配置。 因为 可以在所有用户定义的函数中访问ExecutionConfig，这是一种可以全局访问配置的方便方式。
* addDefaultKryoSerializer(Class<?> type, Serializer<?> serializer)  为一个指定类型注册一个kryo 序列化instance
* addDefaultKryoSerializer(Class<?> type, Class<? extends Serializer<?>> serializerClass) 为一个指定类型注册kryo序列化类。
* registerTypeWithKryoSerializer(Class<?> type, Serializer<?> serializer)， 注册一个指定的类型给kryo并指定一个序列化对象给它。 通过注册类型到kryo， 这个类型的序列化会更高效。
* registerKryoType(Class<?> type) Registers the given type with the serialization stack. If the type is eventually serialized as a POJO, then the type is registered with the POJO serializer. If the type ends up being serialized with Kryo, then it will be registered at Kryo to make sure that only tags are written.
* registerPojoType(Class<?> type) Registers the given type with the serialization stack. If the type is eventually serialized as a POJO, then the type is registered with the POJO serializer. If the type ends up being serialized with Kryo, then it will be registered at Kryo to make sure that only tags are written. If a type is not registered with Kryo, its entire class-name will be serialized with every instance, leading to much higher I/O costs.
* disableAutoTypeRegistration() Automatic type registration is enabled by default. The automatic type registration is registering all types (including sub-types) used by usercode with Kryo and the POJO serializer.

注意: kryo序列化实例是不能访问用registerKryoType注册的类型。

在Rich*函数中通过getRuntimeContext得到的RuntimeContext 也可以访问ExecutionConfig 在所有用户定义的函数中。

<a href="#top">返回顶部</a>

<a name="Program-Packaging-and-Distributed-Execution"> </a>
# 程序打包和分布式执行
向前面所叙， flink程序可以运行在远程的集群环境中。 程序可以打包成jar来执行。 打包程序是通过命令执行它们的前提条件。

## 打包程序
为了支持通过命令行或web 接口从一个打包的jar执行， 一个程序必须使用通过StreamExecutionEnvironment.getExecutionEnvironment()获得的environment. 当jar通过命令行或web接口提交上去后， 这个environment就会作为集群模式下的environment. 如果flink程序不是通过这些接口来调用， 这个environment就运行在本地environment。

为了打包程序，可以简单export所有涉及到的class到一个jar文件中。这个jar文件的manifest必须指向程序的入口累。 最简单的方式是设置main-class入口到mainifest。 这个main-class设置和jvm执行一个jar文件（java -jar pathtojarfile）找到main入口函数一样。 大部分的ide提供导出jar时自动设置入口函数。

## 通过计划打包程序
flink 还支持通过plans来打包程序。 不是在main函数中定义一个程序并在那个环境上调用execute， plan packaging返回程序计划(program plan), 它是用来描叙程序data flow。 因此， 程序必须实现org.apache.flink.api.common.Program接口， 定义getPlan(String...)。 传递给这个函数的string列表就是命令行参数， 这个程序的plan可以通过 ExecutionEnvironment#createProgramPlan()来创建出来， 这个jar的mainfest必须指向实现org.apache.flinkapi.common.Program接口类的class，而不是带main入口函数的类。

## 总结
调用一个打包的程序的流程如下：
* 搜索jar的manifest 找到main-class或program－class， 如果设置了2个属性， 则program-class属性优先考虑。 当jar的manifest 2者都不包含时， 命令行和web 接口支持输入参数来指定入口函数。
* 如果入口类实现org.apache.flinkapi.common.Program， 则系统会调用它的getPlan来获得程序plan去执行。
* 如果入口类没有实现org.apache.flinkapi.common.Program， 则系统调用对应类的main函数。

<a href="#top">返回顶部</a>

<a name="Accumulators-Counters"></a>
# 累加器和计数器
累加器（Accumulators）由一个add 操作和一个final 累加结果组成， 当任务结束后，可以获得累加结果。

最直接的累加器就是计数器（counter）。 用户可以调用Accumulator.add(V value)来增加。 任务结束后， flink会合并所有的结果并将总结果返回给客户端。累加器在调试时很有作用，另外用户如果想知道数据更多的信息也很有作用。

flink 现在有如下的内置累加器， 他们都实现了“Accumulator”接口：
＊ IntCounter, LongCounter and DoubleCounter: See below for an example using a counter.
＊ Histogram: A histogram implementation for a discrete number of bins. Internally it is just a map from Integer to Integer. You can use this to compute distributions of values, e.g. the distribution of words-per-line for a word count program

## 如何使用累加器:
* 首先在用户自定义的transformation函数中创建一个累加器
```
private IntCounter numLines = new IntCounter();
```
* 注册这个累加器对象， 通过在rich函数内部的open函数内， 如下，可以定义一个名字
```
getRuntimeContext().addAccumulator("num-lines", this.numLines);
```
＊ 可以在这个operator函数的任何地方调用累加器 包括open/close
```
this.numLines.add(1);
```
＊  最终结果会存储在由environment的execute函数返回的JobExecutionResult对象中。
```
myJobExecutionResult.getAccumulatorResult("num-lines")
```
一个任务的所有的累加器共享一个namespace。 因此用户可以在不同的operator函数中使用相同的累加器。 flink内部会merge所有的相同名字的累加器。

关于累加器和迭代器的一个注意：当器累加器的结果只能当任务结束后才能获得。 开发者计划前一个迭代的结果可以在下一个迭代中获得。 用户可以使用迭代器去计算每个迭代的statics。

## 用户自定义累加器
用户可以实现自己的累加器。 如果想要把自定义的累加器推广到flink社区，请随意创建一个pull request。

还有一种选择是实现 Accumulator or SimpleAccumulator.

Accumulator<V,R> 非常灵活， 它定义需要增加的value的类型V, 最终返回类型R. 例如： 对于一个histogram， V 是一个数字， R是一个histogram。 SimpleAccumulator 就是V和R是同一种类型， 比如技术器。

<a href="#top">返回顶部</a>

<a name="Parallel-Execution"></a>
# 并发执行
这个章节将介绍在flink中如何设置并发执行。 一个flink程序由多个task(transformation/operator, data sources/sinks)组成。 一个task会切分到几个实例中并发执行， 每一个并发实例处理task输入数据的一个子集。 task的并发实例数称为并行度。

## operator级别
独立的operator/data source/data sink 可以通过setParallelism来设置
```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = text
    .flatMap(new LineSplitter())
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5);

wordCounts.print();

env.execute("Word Count Example");
```

## 执行环境级别
本章所说， flink程序是在一个执行环境的context中运行。一个执行环境可以定义一个默认的所有operator/data source/data sink的并行度， 通过setParallelism。 执行环境的并行度可以被一个operator的并行度覆盖。
```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(3);

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = [...]
wordCounts.print();

env.execute("Word Count Example");
```

## 客户端级别
当提交任务到flink时， 客户端可以设置并行度。 这个客户端可以是java或scala程序， 举例来说，就是命令行客户端(Command-line interface CLI)

在CLI中， 可以通过－p参数来设置并行度， 例如：
```
./bin/flink run -p 10 ../examples/*WordCount-java*.jar
```
在java程序中：
```
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
```

## 系统级别
可以在配置文件./conf/flink-conf.yaml 中设置“parallelism.default”， 从而设置系统范围的默认并行度。

<a href="#top">返回顶部</a>

<a name=“Execution-Plans”></a>
# 执行计划
根据各种参数，比如数据大小， 集群机器数， flink的优化器会自动选择执行策略。 很多情况下， 了解flink如何执行用户程序是很有帮助的。

## plan可视化工具
flink自带执行计划可视化工具， 可视化工具位于tools/planVisualizer.html. 它接受代表任务执行计划的json， 并将它展示成一个带执行策略注释的图。

下面代码展示如何打印执行计划json:
```
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

...

System.out.println(env.getExecutionPlan());
```

使用步骤：
1. 在浏览器中打开planVisualizer.html
2. 粘贴json 字符串
3. 点击draw 按钮

<img alt="A flink job execution graph." src="fig/plan_visualizer.png" width="80%">

## web接口
flink提供web接口提交和执行任务。 如果用户选择使用web 接口提交打包的程序， 用户可以看得到plan 图。

启动web 接口的脚步位于 bin/start-webclient.sh。 启动webclient（8080端口）后， 用户可以上传程序， 上传的程序会列在左边可使用的程序列表中。

用户也可以在页面底部的textbox中指定程序的参数。 在执行实际程序前，请检查可视化的执行计划。

