---
title: "Hadoop Compatibility"
is_beta: true
# Sub-level navigation
sub-nav-group: batch
sub-nav-pos: 7
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

>注：本节未经校验，如有问题欢迎提issue

flink 兼容hadoop mapreduce的接口， 因此可以允许使用基于mapreduce的代码。

You can:

- use Hadoop's `Writable` [data types](index.html#data-types) in Flink programs.
- use any Hadoop `InputFormat` as a [DataSource](index.html#data-sources).
- use any Hadoop `OutputFormat` as a [DataSink](index.html#data-sinks).
- use a Hadoop `Mapper` as [FlatMapFunction](dataset_transformations.html#flatmap).
- use a Hadoop `Reducer` as [GroupReduceFunction](dataset_transformations.html#groupreduce-on-grouped-dataset).

这篇文档展示如何在flink中使用现存的hadoop mapreduce 代码。 可以参考[Connecting to other systems]({{ site.baseurl }}/apis/connectors.html)
来了解如何从hadoop支持的文件系统中读取数据。

* This will be replaced by the TOC
{:toc}

### Project Configuration


支持hadoop的input／output format仅仅是`flink-java` and`flink-scala`maven模块的部分。
`mapred` and `mapreduce` api 代码在`org.apache.flink.api.java.hadoop` 和
`org.apache.flink.api.scala.hadoop` 在一个额外的子package中

支持hadoop mapreduce是在`flink-hadoop-compatibility` maven模块中。
代码具体在`org.apache.flink.hadoopcompatibility` package中

如果想要重复使用Mappers and Reducers， 得在maven中添加下面依赖

~~~xml
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>flink-hadoop-compatibility{{ site.scala_version_suffix }}</artifactId>
	<version>{{site.version}}</version>
</dependency>
~~~

### Using Hadoop Data Types

flink 支持所有的hadoop `Writable` and `WritableComparable` 数据类型。 不用额外添加hadoop Compatibility 依赖。
可以参考[Programming Guide](index.html#data-types) 了解如何使用hadoop data type

### Using Hadoop InputFormats


通过`ExecutionEnvironment`的`readHadoopFile` or `createHadoopInput`， 
用hadoop input format来创建data source。 前者用于format来自`FileInputFormat`
后者用于普通的inpout format。


创建的`DataSet`包含2-tuple， 第一个字段是key，第二个字段是从hadoop InputFormat获取的值。

The following example shows how to use Hadoop's `TextInputFormat`.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Tuple2<LongWritable, Text>> input =
    env.readHadoopFile(new TextInputFormat(), LongWritable.class, Text.class, textPath);

// Do something with the data.
[...]
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val env = ExecutionEnvironment.getExecutionEnvironment

val input: DataSet[(LongWritable, Text)] =
  env.readHadoopFile(new TextInputFormat, classOf[LongWritable], classOf[Text], textPath)

// Do something with the data.
[...]
~~~

</div>

</div>

### Using Hadoop OutputFormats


flink 提供兼容Hadoop `OutputFormats`的封装。 可以支持任何实现`org.apache.hadoop.mapred.OutputFormat`
或继承`org.apache.hadoop.mapreduce.OutputFormat`的类。


OutputFormat 封装需要输入的DataSet 是一个key／value的 2-tuple格式。 它们会由Hadoop OutputFormat 进行处理。

The following example shows how to use Hadoop's `TextOutputFormat`.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// Obtain the result we want to emit
DataSet<Tuple2<Text, IntWritable>> hadoopResult = [...]

// Set up the Hadoop TextOutputFormat.
HadoopOutputFormat<Text, IntWritable> hadoopOF =
  // create the Flink wrapper.
  new HadoopOutputFormat<Text, IntWritable>(
    // set the Hadoop OutputFormat and specify the job.
    new TextOutputFormat<Text, IntWritable>(), job
  );
hadoopOF.getConfiguration().set("mapreduce.output.textoutputformat.separator", " ");
TextOutputFormat.setOutputPath(job, new Path(outputPath));

// Emit data using the Hadoop TextOutputFormat.
hadoopResult.output(hadoopOF);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
// Obtain your result to emit.
val hadoopResult: DataSet[(Text, IntWritable)] = [...]

val hadoopOF = new HadoopOutputFormat[Text,IntWritable](
  new TextOutputFormat[Text, IntWritable],
  new JobConf)

hadoopOF.getJobConf.set("mapred.textoutputformat.separator", " ")
FileOutputFormat.setOutputPath(hadoopOF.getJobConf, new Path(resultPath))

hadoopResult.output(hadoopOF)


~~~

</div>

</div>

### Using Hadoop Mappers and Reducers


Hadoop Mappers 语法上等价于flink的[FlatMapFunctions](dataset_transformations.html#flatmap) ， 
Hadoop Reducers 语法上等价于flink的[GroupReduceFunctions](dataset_transformations.html#groupreduce-on-grouped-dataset)。
flink 同样封装了hadoop mapreduce的`Mapper` and `Reducer`接口的实现。 用户可以在普通flink程序中再次使用hadoop的Mappers and Reducers。 
但仅仅`org.apache.hadoop.mapred`的Mapper and Reducer接口被支持


封装函数用`DataSet<Tuple2<KEYIN,VALUEIN>>`作为输入， 产生`DataSet<Tuple2<KEYOUT,VALUEOUT>>` 作为输出， `KEYIN` and `KEYOUT`是key ， 
`VALUEIN` and `VALUEOUT` 是values， 他们是hadoop函数处理的key／value对。 对于Reducers， flink 用`HadoopReduceCombineFunction` 封装GroupReduceFunction，
但没有Combiner (`HadoopReduceFunction`)。 封装函数接收可选的`JobConf` 来配置hadoop的Mapper or Reducer。

Flink's function wrappers are

- `org.apache.flink.hadoopcompatibility.mapred.HadoopMapFunction`,
- `org.apache.flink.hadoopcompatibility.mapred.HadoopReduceFunction`, and
- `org.apache.flink.hadoopcompatibility.mapred.HadoopReduceCombineFunction`.

他们可以被用于[FlatMapFunctions](dataset_transformations.html#flatmap) or [GroupReduceFunctions](dataset_transformations.html#groupreduce-on-grouped-dataset).

The following example shows how to use Hadoop `Mapper` and `Reducer` functions.

~~~java
// Obtain data to process somehow.
DataSet<Tuple2<Text, LongWritable>> text = [...]

DataSet<Tuple2<Text, LongWritable>> result = text
  // use Hadoop Mapper (Tokenizer) as MapFunction
  .flatMap(new HadoopMapFunction<LongWritable, Text, Text, LongWritable>(
    new Tokenizer()
  ))
  .groupBy(0)
  // use Hadoop Reducer (Counter) as Reduce- and CombineFunction
  .reduceGroup(new HadoopReduceCombineFunction<Text, LongWritable, Text, LongWritable>(
    new Counter(), new Counter()
  ));
~~~

**Please note:** Reducer封装处理由[groupBy()](dataset_transformations.html#transformations-on-grouped-dataset) 定义的groups。
它并不考虑任何在`JobConf`定义的自定义的分区器(partitioners), sort 或 grouping comparator。

### Complete Hadoop WordCount Example

下面给出一个完整的使用hadoop 数据类型， InputFormat/OutputFormat/Mapper/Redueer 的example。

~~~java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// Set up the Hadoop TextInputFormat.
Job job = Job.getInstance();
HadoopInputFormat<LongWritable, Text> hadoopIF =
  new HadoopInputFormat<LongWritable, Text>(
    new TextInputFormat(), LongWritable.class, Text.class, job
  );
TextInputFormat.addInputPath(job, new Path(inputPath));

// Read data using the Hadoop TextInputFormat.
DataSet<Tuple2<LongWritable, Text>> text = env.createInput(hadoopIF);

DataSet<Tuple2<Text, LongWritable>> result = text
  // use Hadoop Mapper (Tokenizer) as MapFunction
  .flatMap(new HadoopMapFunction<LongWritable, Text, Text, LongWritable>(
    new Tokenizer()
  ))
  .groupBy(0)
  // use Hadoop Reducer (Counter) as Reduce- and CombineFunction
  .reduceGroup(new HadoopReduceCombineFunction<Text, LongWritable, Text, LongWritable>(
    new Counter(), new Counter()
  ));

// Set up the Hadoop TextOutputFormat.
HadoopOutputFormat<Text, IntWritable> hadoopOF =
  new HadoopOutputFormat<Text, IntWritable>(
    new TextOutputFormat<Text, IntWritable>(), job
  );
hadoopOF.getConfiguration().set("mapreduce.output.textoutputformat.separator", " ");
TextOutputFormat.setOutputPath(job, new Path(outputPath));

// Emit data using the Hadoop TextOutputFormat.
result.output(hadoopOF);

// Execute Program
env.execute("Hadoop WordCount");
~~~
