---
title:  "集群安装"
top-nav-group: deployment
top-nav-title: 集群 (Standalone)
top-nav-pos: 2
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

本文主要介绍如何将Flink以分布式模式运行在集群上（可能是异构的）。

* This will be replaced by the TOC
{:toc}

## 环境准备

Flink 运行在所有*类 UNIX 环境*上，例如 **Linux**、**Mac OS X** 和 **Cygwin**（对于Windows），而且要求集群由**一个master节点**和**一个或多个worker节点**组成。在安装系统之前，确保**每台机器**上都已经安装了下面的软件：

- **Java 1.7.x**或更高版本
- **ssh**（Flink的脚本会用到sshd来管理远程组件）

如果你的集群还没有完全装好这些软件，你需要安装/升级它们。例如，在 Ubuntu Linux 上， 你可以执行下面的命令安装 ssh 和 Java ：

```bash
sudo apt-get install ssh 
sudo apt-get install openjdk-7-jre
```

<a id="passwordless-ssh"></a>

### SSH 免密码登录

*译注：安装过 Hadoop、Spark 集群的用户应该对这段很熟悉，如果已经了解，可跳过。*

为了能够启动/停止远程主机上的进程，master 节点需要能免密登录所有 worker 节点。最方便的方式就是使用ssh的公钥验证了。要安装公钥验证，首先以最终会运行 Flink 的用户登录 master 节点。**所有的 worker 节点上也必须要有同样的用户（例如：使用相同用户名的用户）**。本文会以 `flink` 用户为例。非常不建议使用 `root` 账户，这会有很多的安全问题。

当你用需要的用户登录了master节点，你就可以生成一对新的公钥/私钥。下面这段命令会在 ~/.ssh 目录下生成一对新的公钥/私钥。

```bash
ssh-keygen -b 2048 -P '' -f ~/.ssh/id_rsa
```

接下来，将公钥添加到用于认证的`authorized_keys`文件中：

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

最后，将`authorized_keys`文件分发给集群中所有的worker节点，你可以重复地执行下面这段命令：

```bash
scp ~/.ssh/authorized_keys <worker>:~/.ssh/
```

将上面的`<worker>`替代成相应worker节点的IP/Hostname。完成了上述拷贝的工作，你应该就可以从master上免密登录其他机器了。

```bash
ssh <worker>
```


### 配置JAVA_HOME

Flink 需要master和worker节点都配置了`JAVA_HOME`环境变量。有两种方式可以配置。

一种是，你可以在`conf/flink-conf.yaml`中设置`env.java.home`配置项为Java的安装路径。

另一种是，`sudo vi /etc/profile`，在其中添加`JAVA_HOME`：

```bash
export JAVA_HOME=/path/to/java_home/
```

然后使环境变量生效，并验证 Java 是否安装成功

```bash
$ source /etc/profile   #生效环境变量
$ java -version         #如果打印出版本信息，则说明安装成功
java version "1.7.0_75"
Java(TM) SE Runtime Environment (build 1.7.0_75-b13)
Java HotSpot(TM) 64-Bit Server VM (build 24.75-b04, mixed mode)
```

{% top %}


## 安装 Flink

进入[下载页面]({{ site.download_url }})。请选择一个与你的Hadoop版本相匹配的Flink包。如果你不打算使用Hadoop，选择任何版本都可以。

在下载了最新的发布包后，拷贝到master节点上，并解压：

```bash
tar xzf flink-*.tgz
cd flink-*
```

## 配置 Flink

在解压完之后，你需要编辑`conf/flink-conf.yaml`配置Flink。

设置`jobmanager.rpc.address`配置项为你的master节点地址。另外为了明确 JVM 在每个节点上所能分配的最大内存，我们需要配置`jobmanager.heap.mb`和`taskmanager.heap.mb`，值的单位是 MB。如果对于某些worker节点，你想要分配更多的内存给Flink系统，你可以在相应节点上设置`FLINK_TM_HEAP`环境变量来覆盖默认的配置。

最后，你需要提供一个集群中worker节点的列表。因此，就像配置HDFS，编辑*conf/slaves*文件，然后输入每个worker节点的 IP/Hostname。每一个worker结点之后都会运行一个 TaskManager。

每一条记录占一行，就像下面展示的一样：

```bash
192.168.0.100
192.168.0.101
.
.
.
192.168.0.150
```

*译注：conf/master文件是用来做 [JobManager HA](setup/jobmanager_high_availability.html) 的，在这里不需要配置*

每一个worker节点上的 Flink 路径必须一致。你可以使用共享的 NSF 目录，或者拷贝整个 Flink 目录到各个worker节点。

```bash
scp -r /path/to/flink <worker>:/path/to/
```

请查阅[配置页面](config.html)了解更多关于Flink的配置。

特别的，这几个

- TaskManager 总共能使用的内存大小（`taskmanager.heap.mb`）
- 每一台机器上能使用的 CPU 个数（`taskmanager.numberOfTaskSlots`）
- 集群中的总 CPU 个数（`parallelism.default`）
- 临时目录（`taskmanager.tmp.dirs`）

是非常重要的配置项。

{% top %}


## 启动 Flink

下面的脚本会在本地节点启动一个 JobManager，然后通过 SSH 连接所有的worker节点（*slaves*文件中所列的节点），并在每个节点上运行 TaskManager。现在你的 Flink 系统已经启动并运行了。跑在本地节点上的 JobManager 现在会在配置的 RPC 端口上监听并接收任务。

假定你在master节点上，并在Flink目录中：

```bash
bin/start-cluster.sh
```

要停止Flink，也有一个 *stop-cluster.sh* 脚本。

{% top %}


## 添加 JobManager/TaskManager 实例到集群中

你可以使用 *bin/jobmanager.sh* 和 *bin/taskmanager* 脚本来添加 JobManager 和 TaskManager 实例到你正在运行的集群中。

### 添加一个 JobManager

```bash
bin/jobmanager.sh (start cluster)|stop|stop-all
```

### 添加一个 TaskManager

```bash
bin/taskmanager.sh start|stop|stop-all
```

确保你是在需要启动/停止相应实例的节点上运行的这些脚本。


{% top %}
