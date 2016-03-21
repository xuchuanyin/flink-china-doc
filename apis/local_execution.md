---
title:  "Local Execution"
# Top-level navigation
top-nav-group: apis
top-nav-pos: 7
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

Flink 可以单机运行，也可以是一台Java虚拟机。
这使得用户可以在本地debug。
本节主要讲述本地模式的运行机制。

本地的JVM运行环境或者是带JVM运行环境的程序都可以运行Flink程序。
大部分示例程序，你只需简单的地点击你IDE上的运行按钮就可以运行。

Flink在本地模式下支持两种运行方式。
1.`LocalExecutionEnvironment` 这个环境包含了所有 Flink 的运行环境，不但包括 JobManager 和一个 TaskManager。
还包含内存管理和所有在集群模式下的内部算法。

2.`CollectionEnvironment` 这个环境使用Java集合来执行Flink程序。
这个模式不会启动一个完全的Flink运行环境。
因此他会非常低开销、轻量地来执行。
例如 `DataSet.map()`-转换操作，执行机制将是对JAVA list里的所有元素进行`map()` 函数的操作。

* TOC
{:toc}


## 调试

如果你本地运行Flink程序，你可以像普通的Java程序一样调试Flink程序。
你既可以用`System.out.println()` 打印一些内部变量，也可以使用其它的调试器。
你还可以在`map()`, `reduce()` 或者其它方法上下断点。
详细请看 [调试部分](programming_guide.html#debugging) 里的 Java API 测试与本地调试工具API指南文档。

## Maven 依赖

如果你使用Maven开发你的程序，你需要使用下面方式加入 `flink-clients` 模块的依赖:

~~~xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version}}</version>
</dependency>
~~~

## 本地环境

本地环境 `LocalEnvironment` 来处理一个本地的 Flink 执行程序.
用来在本地的JVM里来运行程序 - 独立模式或嵌入到其它程序中。

本地环境通过调用方法`ExecutionEnvironment.createLocalEnvironment()`来获得。
默认情况下，他根据你的CPU核数（硬件环境）来在本地开启同样多的线程。
你可以指定你需要的并行度。
本模式的日志可以通过`enableLogging()`/`disableLogging()`来设置是否向标准输出输出。

大多数情况，调用`ExecutionEnvironment.getExecutionEnvironment()` 来获得Flink运行环境是一个好方式。
这个方法在程序在本运行的时候（在CLI范围外）会返回一个`LocalEnvironment` 类，
通过[命令行接口（CLI）](cli.html)来调用这个类的话，就已经预先配置好了以集群模式的执行。 

~~~java
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

    DataSet<String> data = env.readTextFile("file:///path/to/file");

    data
        .filter(new FilterFunction<String>() {
            public boolean filter(String value) {
                return value.startsWith("http://");
            }
        })
        .writeAsText("file:///path/to/result");

    JobExecutionResult res = env.execute();
}
~~~

`JobExecutionResult` 对象在程序执行结束时会返回，这个类中包含了程序的运行环境和累加器的结果。

`LocalEnvironment` 类也可以向Flink传入一个用户自定义的配置选项。

~~~java
Configuration conf = new Configuration();
conf.setFloat(ConfigConstants.TASK_MANAGER_MEMORY_FRACTION_KEY, 0.5f);
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment(conf);
~~~

*Note:* The local execution environments do not start any web frontend to monitor the execution.

## 集合环境

集合环境使用`CollectionEnvironment` 类，这个类执行Flink程序时使用了一些低开销的方法。
通过这个模式是用来进行自动化测试、调试、代码重用等这些场景来使用的。

在互动性强的场景下，用户可以使用批处理的实现算法。
稍微改动下Flink程序就可以将其变成用于处理请求的Java应用服务器。

**基于集合环境的执行框架**

~~~java
public static void main(String[] args) throws Exception {
    // initialize a new Collection-based execution environment
    final ExecutionEnvironment env = new CollectionEnvironment();

    DataSet<User> users = env.fromCollection( /* 从Java集合中获得元素 */);

    /* 数据集转换 ... */

    // 在ArrayList中查询 Tuple2 元素。
    Collection<...> result = new ArrayList<...>();
    resultDataSet.output(new LocalCollectionOutputFormat<...>(result));

    // 开始执行。
    env.execute();

    // 对结果进行其它处理 ArrayList (=Collection).
    for(... t : result) {
        System.err.println("Result = "+t);
    }
}
~~~

所有的示例在`flink-examples-batch` 模块的 `CollectionExecutionExample` 中.

值得注意的是，基于集合环境的 Flink 程序只适用于小数据，和JVM的堆大小相匹配。
集合模式只的执行器只使用了单线程，而非多线程。
