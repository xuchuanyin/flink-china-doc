---
title: "快速起步: 安装"
# Top navigation
top-nav-group: quickstart
top-nav-pos: 1
top-nav-title: 安装
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

只需要简单几步就能把一个 Flink 示例程序跑起来。

## 安装：下载和运行

Flink 可以运行在 **Linux**，**Mac OS X** 和 **Windows** 上，唯一要求是需要安装 **Java 7.x** 或更高版本。 对于 Window 用户，请参考 [Flink on Windows]({{ site.baseurl }}/setup/local_setup.html#flink-on-windows)。

### 下载

从 [下载页面](http://flink.apache.org/downloads.html) 下载所需的二进制包。你可以选择任何与 Hadoop/Scala 结合的版本。比如 [Flink for Hadoop 2]({{ site.FLINK_DOWNLOAD_URL_HADOOP2_STABLE }})。

### 运行一个本地的 Flink 集群

1. 进入下载的目录。
2. 解压下载的压缩包。
3. 运行 Flink 。


```bash
$ cd ~/Downloads        # 进入下载的目录
$ tar xzf flink-*.tgz   # 解压下载的压缩包
$ cd flink-0.10.2
$ bin/start-local.sh    # 运行 Flink
```

打开 Web UI [http://localhost:8081](http://localhost:8081) 检查 Jobmanager 和其他组件是否正常运行。Web 前端应该显示了只有一个可用的 TaskManager 。如果想打开流优化模式，可以调用 `bin/start-local-streaming.sh` 来替换 `bin/start-local.sh`。

<a href="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-1.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-1.png" alt="JobManager: Overview"/></a>

## 运行例子


现在我们准备开始运行 [SocketTextStreamWordCount 实例](https://github.com/apache/flink/blob/release-1.0.0/flink-quickstart/flink-quickstart-java/src/main/resources/archetype-resources/src/main/java/SocketTextStreamWordCount.java) 了。我们会从一个 socket 中读取文本然后统计不同单词出现的次数。

* 首先，我们使用 **netcat** 来启动本地服务器：

  ~~~bash
  $ nc -l -p 9000
  ~~~ 

* 提交 Flink 程序：

  ~~~bash
  $ bin/flink run examples/streaming/SocketTextStreamWordCount.jar \
    --hostname localhost \
    --port 9000
  Printing result to stdout. Use --output to specify output path.
  03/08/2016 17:21:56 Job execution switched to status RUNNING.
  03/08/2016 17:21:56 Source: Socket Stream -> Flat Map(1/1) switched to SCHEDULED
  03/08/2016 17:21:56 Source: Socket Stream -> Flat Map(1/1) switched to DEPLOYING
  03/08/2016 17:21:56 Keyed Aggregation -> Sink: Unnamed(1/1) switched to SCHEDULED
  03/08/2016 17:21:56 Keyed Aggregation -> Sink: Unnamed(1/1) switched to DEPLOYING
  03/08/2016 17:21:56 Source: Socket Stream -> Flat Map(1/1) switched to RUNNING
  03/08/2016 17:21:56 Keyed Aggregation -> Sink: Unnamed(1/1) switched to RUNNING
  ~~~

  程序会去连接 socket 然后等待输入。你可以通过 web 界面验证任务是否如预期正常运行了：

  <div class="row">
    <div class="col-sm-6">
      <a href="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-2.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-2.png" alt="JobManager: Overview (cont'd)"/></a>
    </div>
    <div class="col-sm-6">
      <a href="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-3.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-3.png" alt="JobManager: Running Jobs"/></a>
    </div>
  </div>

* 计数会打印到标准输出 `stdout`。监控 JobManager 的输出文件(`.out`文件)，并在 `nc` 中敲入一些单词：

  ~~~bash
  $ nc -l -p 9000
  lorem ipsum
  ipsum ipsum ipsum
  bye
  ~~~

  `.out` 文件会立即打印出单词的计数：

  ~~~bash
  $ tail -f log/flink-*-jobmanager-*.out
  (lorem,1)
  (ipsum,1)
  (ipsum,2)
  (ipsum,3)
  (ipsum,4)
  (bye,1)
  ~~~~
  
  要**停止** Flink，只需要运行：

  ~~~bash
  $ bin/stop-local.sh
  ~~~

  <a href="{{ site.baseurl }}/page/img/quickstart-setup/setup.gif" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-setup/setup.gif" alt="Quickstart: Setup"/></a>

## 下一步

想要体验 Flink 编程 API 的初恋感觉可以阅读 [一步步的例子](run_example_quickstart.html)。当你完成了那个，继续阅读 [streaming 指南]({{ site.baseurl }}/apis/streaming/) 吧。

### 集群安装

**在集群上运行 Flink** 是和在本地运行一样简单的。需要先配置好 [SSH 免密码登录]({{ site.baseurl }}/setup/cluster_setup.html#passwordless_ssh) 和保证所有节点的**目录结构是一致的**，这是保证我们的脚本能正确控制任务启停的关键。

1. 在每台节点上，复制解压出来的 **flink** 目录到同样的路径下。
2. 选择一个 **master 节点** (JobManager) 然后在 `conf/flink-conf.yaml` 中设置 `jobmanager.rpc.address` 配置项为该节点的 IP 或者主机名。确保所有节点有有一样的 `jobmanager.rpc.address` 配置。
3. 将所有的 **worker 节点** （TaskManager）的 IP 或者主机名（一行一个）填入 `conf/slaves` 文件中。

现在，你可以在 master 节点上**启动集群**：`bin/start-cluster.sh` 。

下面的**例子**阐述了三个节点的集群部署（IP地址从 _10.0.0.1_ 到 _10.0.0.3_，主机名分别为 _master_, _worker1_, _worker2_）。并且展示了配置文件，以及所有机器上一致的可访问的安装路径。


<div class="row">
  <div class="col-md-6 text-center">
    <img src="{{ site.baseurl }}/page/img/quickstart_cluster.png" style="width: 85%">
  </div>
<div class="col-md-6">
  <div class="row">
    <p class="lead text-center">
      /path/to/<strong>flink/conf/<br>flink-conf.yaml</strong>
    <pre>jobmanager.rpc.address: 10.0.0.1</pre>
    </p>
  </div>
<div class="row" style="margin-top: 1em;">
  <p class="lead text-center">
    /path/to/<strong>flink/<br>conf/slaves</strong>
  <pre>
10.0.0.2
10.0.0.3</pre>
  </p>
</div>
</div>
</div>


访问文档的[配置章节]({{ site.baseurl }}/setup/config.html)查看更多可用的配置项。为了使 Flink 更高效的运行，还需要设置一些配置项。

以下都是非常重要的配置项：

- TaskManager 总共能使用的内存大小（`taskmanager.heap.mb`）
- 每一台机器上能使用的 CPU 个数（`taskmanager.numberOfTaskSlots`）
- 集群中的总 CPU 个数（`parallelism.default`）
- 临时目录（`taskmanager.tmp.dirs`）


### Flink on YARN

你可以很方便地将 Flink 部署在现有的 **YARN 集群**上：

1. 下载  __Flink Hadoop2 包__: [Flink with Hadoop 2]({{site.FLINK_DOWNLOAD_URL_HADOOP2_STABLE}})
2. 确保你的 __HADOOP_HOME__ (或 _YARN_CONF_DIR_ 或 _HADOOP_CONF_DIR_) __环境变量__设置成你的 YARN 和 HDFS 配置。
3. 运行 **YARN 客户端**：`./bin/yarn-session.sh` 。你可以带参数运行客户端 `-n 10 -tm 8192` 表示分配 10 个 TaskManager，每个拥有 8 GB 的内存。

更多详细的操作指南，请查阅编程指南和范例。

