---
title:  "本地执行"
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

Flink 不但可以单机运行，即使是单个 Java 虚拟机也可以。
这使得用户可以在本地调试。
本节主要讲述本地模式的运行机制。

Flink 可以运行在本地 Java 虚拟机中，或者带 JVM 环境的程序中。
对于大部分示例程序而言，你只需简单的地点击你IDE上的运行按钮就可以运行。

Flink在本地模式下支持两种运行方式。

1. `LocalExecutionEnvironment`  启动了完整的 Flink 运行环境，包括了一个 JobManager 和一个 TaskManager。还包含了内存管理和所有在集群模式下的内部算法。
2. `CollectionEnvironment` 这个环境使用Java集合来执行Flink程序。这个模式不会启动一个完全的 Flink 运行环境，因此会开销非常低并且很轻量。例如 `DataSet.map()`-转换操作，执行机制将是对Java list里的所有元素进行`map()` 函数的操作。

* TOC
{:toc}


## 调试

如果你本地运行Flink程序，你可以像普通的Java程序一样调试Flink程序。
你既可以用`System.out.println()` 打印一些内部变量，也可以使用 IDE 提供的调试工具。
你还可以在`map()`, `reduce()` 或者其他方法中打断点。
请参见 [Batch 调试指南]({{ site.baseurl }}/apis/batch/debugging) 和 
[Streaming 调试指南]({{ site.baseurl }}/apis/streaming/#debugging) 
了解更多关于如何测试以及 API 中提供的本地调试工具。

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

 `LocalEnvironment` 是用来本地执行 Flink 程序的句柄。
可以用它在本地的 JVM （standalone 或嵌入其他程序）里运行程序。

本地环境通过调用方法`ExecutionEnvironment.createLocalEnvironment()`来获得。
默认情况下，Flink 会根据你的CPU核数（硬件环境）来在本地开启同样多的线程。
你可以指定你需要的并行度。
本模式的日志可以通过`enableLogging()`/`disableLogging()`来设置是否向标准输出输出。

在大多数情况下，更推荐使用`ExecutionEnvironment.getExecutionEnvironment()` 来获得 Flink 的运行环境。
当程序是在本地（未使用命令行接口）启动时，该方法会返回`LocalEnvironment` ，
当程序是通过命令行接口[命令行接口（CLI）](cli.html)提交时，则该方法会返回集群的执行环境。

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

*注意：*本地执行环境不会启动任何 web 前端来监控执行过程。

## 集合环境

集合环境使用`CollectionEnvironment` 类，这个类执行Flink程序时使用了一些低开销的方法。
这种模式通常用于自动化测试、调试、代码重用等场景。

用户可以将实现于批处理的算法用于更具交互性的场景中，一个Flink程序稍加修改即可用于处理请求的Java应用服务器。

**基于集合环境的执行框架**

~~~java
public static void main(String[] args) throws Exception {
    // initialize a new Collection-based execution environment
    final ExecutionEnvironment env = new CollectionEnvironment();

    DataSet<User> users = env.fromCollection( /* 从Java集合中获得元素 */);

    /* 数据集转换 ... */

    // 将转换后的 Tuple2 结果元素导入到 ArrayList 中。
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

在`flink-examples-batch` 模块中有完整的示例，叫 `CollectionExecutionExample` .

值得注意的是，基于集合环境的 Flink 程序只适用于小数据， 不能超过 JVM 的堆内存大小
集合模式只的执行器只使用了单线程，而非多线程。
