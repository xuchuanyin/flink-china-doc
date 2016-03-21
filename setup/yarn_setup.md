---
title:  "YARN 安装"
top-nav-group: deployment
top-nav-title: YARN
top-nav-pos: 3
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

* This will be replaced by the TOC
{:toc}

## 快速起步

### 在 YARN 上启动一个 Flink 集群

启动一个有 4 个 TaskManager （每个都有 4GB 的堆内存）的 YARN 会话：

~~~bash
# get the hadoop2 package from the Flink download page at
# {{ site.download_url }}
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/yarn-session.sh -n 4 -jm 1024 -tm 4096
~~~

`-s` 标记可以用来指定每个 TaskMangager 的 slot 个数。我们建议设置 slot 的个数为每个机器的核数。

当会话成功启动后，你就可以使用 `./bin/flink` 工具提交任务到集群了。

### 在 YARN 上运行 Flink 任务

~~~bash
# get the hadoop2 package from the Flink download page at
# {{ site.download_url }}
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/flink run -m yarn-cluster -yn 4 -yjm 1024 -ytm 4096 ./examples/batch/WordCount.jar
~~~

## Flink YARN 会话

Apache [Hadoop YARN](http://hadoop.apache.org/) 是一个集群资源管理框架。它允许在一个集群之上运行多种分布式应用。Flink 与其他应用程序一同运行在 YARN 之上。如果用户已经安装 YARN，就不需要安装其他东西了。

**环境要求**

- 至少 Apache Hadoop 2.2
- HDFS (Hadoop Distributed File System) (或者 Hadoop 支持的其他的分布式文件系统)

如果你在使用 Flink YARN 客户端的过程中遇到了问题，请参考 [FAQ 章节](http://flink.apache.org/faq.html#yarn-deployment)。

### 启动 Flink 会话

按照下面的操作指南学习如何在 YARN 集群中启动一个 Flink 会话。

会话会启动所有需要的 Flink 服务（JobManager 和 TaskManager），因此可以提交程序到集群中。注意每个会话都可以运行多个程序。

#### 下载 Flink 

从[下载页面]({{ site.download_url }}) 下载 Hadoop 版本 >= 2 对应的 Flink 包。这里面就包含了所需要的文件了。

解压压缩包：

~~~bash
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{site.version }}/
~~~

#### 启动会话

使用下面的命令启动一个会话：

~~~bash
./bin/yarn-session.sh
~~~

这个命令会显示如下的概览：

~~~bash
Usage:
   Required
     -n,--container <arg>   Number of YARN container to allocate (=Number of Task Managers)
   Optional
     -D <arg>                        Dynamic properties
     -d,--detached                   Start detached
     -jm,--jobManagerMemory <arg>    Memory for JobManager Container [in MB]
     -nm,--name                      Set a custom name for the application on YARN
     -q,--query                      Display available YARN resources (memory, cores)
     -qu,--queue <arg>               Specify YARN queue.
     -s,--slots <arg>                Number of slots per TaskManager
     -tm,--taskManagerMemory <arg>   Memory per TaskManager Container [in MB]
~~~

请注意客户端需要提前设置环境变量 `YARN_CONF_DIR` 或 `HADOOP_CONF_DIR`，用来读取 YARN 和 HDFS 配置。

**示例：** 执行下面的命令来分配 10 个 TaskManager，每个都拥有 8 GB 的内存和 32 个 slot。

~~~bash
./bin/yarn-session.sh -n 10 -tm 8192 -s 32
~~~

系统会使用 `conf/flink-config.yaml` 中的配置。如果你想要更改配置，请参考 [配置指南](config.html)。

Flink on YARN 会覆盖这些配置参数 `jobmanager.rpc.address`（因为 JobManager 一直被分配在不同的机器上），`taskmanager.tmp.dirs`（我们使用 YARN 提供了的 tmp 目录）和 `parallelism.default`（如果指定了 slot 数目）。

如果你不想通过修改配置文件的方法来设置配置参数，你可以通过 `-D` 标记传入动态属性。所以你可以这样传递参数：`-Dfs.overwrite-files=true -Dtaskmanager.network.numberOfBuffers=16368`

这个例子请求启动 11 个容器，因为对于 ApplicationMaster 和 JobManager 还需要一个额外的容器。

一旦 Flink 在 YARN 集群中部署了，它会显示 JobManager 连接的详细信息。

要停止 YARN 会话，可以通过结束 unix 进程（使用 CTRL+C）或者通过在客户端中输入 'stop'。

#### 分离的 YARN 会话

如果你不希望 Flink YARN 客户端一直在运行，也可以启动一个 **分离的** （detached）YARN 会话。只需要加上 `-d` 或 `--detached` 参数。

在这种情况下，Flink YARN 客户端只会提交 Flink 到集群中然后关闭自己。注意在这种情况下，不能像上面这样停止 YARN 会话了。

使用 YARN utilities（`yarn application -kill <appId>`）来结束 YARN 会话。

### 提交任务到 Flink

使用下面的命令提交一个 Flink 程序到 YARN 集群：

~~~bash
./bin/flink
~~~

请参考 [命令行客户端]({{ site.baseurl }}/apis/cli.html) 文档。

该命令会显示一个如下的帮助菜单：

~~~bash
[...]
Action "run" compiles and runs a program.

  Syntax: run [OPTIONS] <jar-file> <arguments>
  "run" action arguments:
     -c,--class <classname>           Class with the program entry point ("main"
                                      method or "getPlan()" method. Only needed
                                      if the JAR file does not specify the class
                                      in its manifest.
     -m,--jobmanager <host:port>      Address of the JobManager (master) to
                                      which to connect. Use this flag to connect
                                      to a different JobManager than the one
                                      specified in the configuration.
     -p,--parallelism <parallelism>   The parallelism with which to run the
                                      program. Optional flag to override the
                                      default value specified in the
                                      configuration
~~~

使用 *run* 操作来提交一个任务到 YARN。客户端自己就能确定 JobManager 的地址。在遇到罕见的问题时，你可以使用 `-m` 参数传入 JobManager 的地址。JobManager 的地址可以在 YARN 控制台找到。

**示例**

~~~bash
wget -O LICENSE-2.0.txt http://www.apache.org/licenses/LICENSE-2.0.txt
hadoop fs -copyFromLocal LICENSE-2.0.txt hdfs:/// ...
./bin/flink run ./examples/batch/WordCount.jar \
        hdfs:///..../LICENSE-2.0.txt hdfs:///.../wordcount-result.txt
~~~

如果遇到了下面的错误，请确保所有的 TaskManager 都已经启动了：

~~~bash
Exception in thread "main" org.apache.flink.compiler.CompilerException:
    Available instances could not be determined from job manager: Connection timed out.
~~~

你可以在 JobManager 的 Web 界面上检查 TaskManager 的数目是否正确。JobManager 的地址会在 YARN 会话控制台中打印出来。

如果一分钟后 TaskManager 还没有出现，你就需要通过日志文件查找问题原因了。

## 在 YARN 上运行单个 Flink 任务

上文描述了如何在 Hadoop YARN 环境中启动一个 Flink 集群。另外，也可以在 YARN 中启动只执行单个任务的 Flink。

请注意该客户端需要提供 `-yn` 参数值（TaskManager 的数量）。

***示例:***

~~~bash
./bin/flink run -m yarn-cluster -yn 2 ./examples/batch/WordCount.jar
~~~

通过运行 `./bin/flink`，也可以看到YARN会话的命令行选项。这些选项都带了一个 `y` 或 `yarn` （长参数选项）的前缀。

注：通过为每个任务设置不同的环境变量 `FLINK_CONF_DIR`，可以为每个任务使用不同的配置目录。从 Flink 分发包中复制 `conf` 目录，然后修改配置，例如，每个任务不同的日志设置。

注：可以结合 `-m yarn-cluster` 和分离的 YARN 提交形式（`yd`）来“提交并遗忘”一个 Flink 任务给 YARN 集群。在这种情况下，你的应用程序将无法获得任何累加器的结果或者来自调用 `ExecutionEnvironment.execute()` 的异常。

## Flink on YARN 的恢复机制

Flink 的 YARN 客户端有以下的配置参数来控制在容器故障情况下的行为。这些参数可以通过 `conf/flink-conf.yaml` 来设置，也可以通过启动 YARN 会话时加入 `-D` 参数来设置。

- `yarn.reallocate-failed`: 该参数控制了 Flink 是否该重新分配失败的 TaskManager 容器。默认：true。
- `yarn.maximum-failed-containers`: ApplicationMaster 能接受最多的失败容器的数量，直到 YARN 会话失败。默认：初始请求的 TaskManager 个数（`-n`）。
- `yarn.application-attempts`: ApplicationMaster（以及它的 TaskManager 容器）的尝试次数。默认值为 1， 当 ApplicationMaster 失败了，整个 YARN 会话也会失败。可以通过设置更大的值来更改 YARN 重启A pplicationMaster 的次数。

## 调试失败的 YARN 会话

有许多原因会导致 Flink YARN 会话部署失败，如一个错误的 Hadoop 安装配置（HDFS 权限，YARN 配置），版本不兼容（在 Cloudera Hadoop 上，运行带有普通 Hadoop 依赖的 Flink），或是其他原因。

### 日志文件

Flink YARN 会话 在部署期间自己挂了的情况下，用户只能依赖 Hadoop YARN 的日志来进行排查。其中最有用的特性是 [YARN 日志聚合](http://hortonworks.com/blog/simplifying-user-logs-management-and-access-in-yarn/)。要启用该功能，用户必须在 `yarn-site.xml` 中设置 `yarn.log-aggregation-enable` 属性为 `true`。一旦被启用了，用户就能通过下面的命令获得一个（失败的）YARN 会话的所有日志。

~~~
yarn logs -applicationId <application ID>
~~~

注意在会话结束后到日志展现之间需要等一段时间。

### YARN 客户端控制台 & Web 界面

如果 Flink YARN 客户端在运行时遇到了错误（例如一个 TaskManager 在一段时间后停止工作了），那么它也会在终端打印出错误信息。

除此之外，在 YARN 资源管理 Web 界面（默认端口在 8088）也能看到错误信息。资源管理 Web 界面的端口由 `yarn.resourcemanager.webapp.address` 配置项决定。

它允许访问运行中的 YARN 应用程序的日志文件，并且能展示失败应用的诊断信息。

## 为特定的 Hadoop 版本构建 YARN 客户端

用户使用来自像 Hortonworks, Cloudera 和 MapR 等公司的 Hadoop 发行版，这也许需要针对特定的 Hadoop（HDFS）和 YARN 版本构建 Flink。请阅读 [构建指南](building.html) 了解更多。

## 在防火墙背后运行 Flink on YARN

一些 YARN 集群使用防火墙来控制集群和其他网络之间的网络流量。在这些情况下，Flink 任务只能从集群内部网络（防火墙背后）提交到 YARN 会话。如果这对于生产用途不是很可行，Flink 允许为所有相关服务配置端口区间。配置了端口区间之后，用户就可以跨防火墙提交任务到 Flink 了。

当前，提交任务需要两个服务：

 * JobManager (YARN 中的 ApplicatonMaster)
 * BlobServer （运行在 JobManager 中）。

当提交一个任务给 Flink，BlobServer 会分发带有用户代码的 jar 文件到所有 worker 节点（TaskManager）。JobManager 收到了该任务，然后触发执行过程。

用来指定端口号的这两个配置参数如下：

 * `yarn.application-master.port`
 * `blob.server.port`

这两个配置选项可以接受单个端口（如："50010"），区间端口（"50000-50025"），或者以上两者的结合（"50010,50011,50020-50025,50050-50075"）。

（Hadoop 正在使用一个类似的机制，那个配置参数叫做 `yarn.app.mapreduce.am.job.client.port-range` ）

## 内部实现

本章节简要介绍 Flink 是如何与 YARN 进行交互的。

<img src="fig/FlinkOnYarn.svg" class="img-responsive">

YARN 客户端需要访问 Hadoop 配置，从而连接 YARN 资源管理器和 HDFS。可以使用下面的策略来决定 Hadoop 配置：

* 测试 `YARN_CONF_DIR`, `HADOOP_CONF_DIR` 或 `HADOOP_CONF_PATH` 环境变量是否设置了（按该顺序测试）。如果它们中有一个被设置了，那么它们就会用来读取配置。
* 如果上面的策略失败了（如果正确安装了 YARN 的话，这不应该会发生），客户端会使用 `HADOOP_HOME` 环境变量。如果该变量设置了，客户端会尝试访问 `$HADOOP_HOME/etc/hadoop` (Hadoop 2) 和 `$HADOOP_HOME/conf` (Hadoop 1)。

当启动一个新的 Flink YARN 会话，客户端首先会检查所请求的资源（容器和内存）是否可用。之后，它会上传包含了 Flink 和配置的 jar 到 HDFS（步骤 1）。

客户端的下一步是请求（步骤 2）一个 YARN 容器启动 *ApplicationMaster* （步骤 3）。因为客户端将配置和 jar 文件作为容器的资源注册了，所以运行在特定机器上的 YARN 的 NodeManager 会负责准备容器（例如，下载文件）。一旦这些完成了，*ApplicationMaster* (AM) 就启动了。

*JobManager* 和 AM 运行在同一个容器中。一旦它们成功地启动了，AM 知道 JobManager 的地址（它自己）。它会为 TaskManager 生成一个新的 Flink 配置文件（这样它们才能连上 JobManager）。该文件也同样会上传到 HDFS。另外，*AM* 容器同时提供了 Flink 的 Web 界面服务。Flink 用来提供服务的端口是由用户 + 应用程序 id 作为偏移配置的。这使得用户能够并行执行多个 Flink YARN 会话。

之后，AM 开始为 Flink 的 TaskManager 分配容器，这会从 HDFS 下载 jar 文件和修改过的配置文件。一旦这些步骤完成了，Flink 就安装完成并准备接受任务了。
