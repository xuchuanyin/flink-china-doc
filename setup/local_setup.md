---
title:  "本地安装"
top-nav-group: deployment
top-nav-title: 本地
top-nav-pos: 1
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

本文主要介绍如何将Flink以本地模式运行在单机上。

* This will be replaced by the TOC
{:toc}

## 下载

进入[下载页面]({{ site.download_url }})。如果你想要Flink和Hadoop一起用（如HDFS或者HBase），请选择一个与你的Hadoop版本相匹配的Flink包。当你不确定或者只是想运行在本地文件系统上，请选择Hadoop 1.2.x对应的包。

{% top %}


## 环境准备

Flink 可以运行在 **Linux**、**Mac OS X** 和 **Windows** 上。本地模式的安装唯一需要的只是 Java 1.7.x或更高版本。接下来的指南假定是类Unix环境，Windows用户请参考 [Flink on Windows](#flink-on-windows)。

你可以执行下面的命令来查看是否已经正确安装了Java了。

```bash
java -version
```

这条命令会输出类似于下面的信息：

```bash
java version "1.8.0_51"
Java(TM) SE Runtime Environment (build 1.8.0_51-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.51-b03, mixed mode)
```


{% top %}


## 配置

**对于本地模式，Flink是可以开箱即用的，你不用去更改任何的默认配置。**

开箱即用的配置会使用默认的Java环境。如果你想更改Java的运行环境，你可以手动地设置环境变量`JAVA_HOME`或者`conf/flink-conf.yaml`中的配置项`env.java.home`。你可以查阅[配置页面](config.html)了解更多关于Flink的配置。

{% top %}


## 启动 Flink

你现在就可以开始运行Flink了。解压已经下载的压缩包，然后进入新创建的`flink`目录。在那里，你就可以本地模式运行Flink了：

```bash
$ tar xzf flink-*.tgz
$ cd flink-*
$ bin/start-local.sh
Starting job manager
```

你可以通过观察logs目录下的日志文件来检查系统是否正在运行了：

```bash
$ tail log/flink-*-jobmanager-*.log
INFO ... - Initializing memory manager with 409 megabytes of memory
INFO ... - Trying to load org.apache.flinknephele.jobmanager.scheduler.local.LocalScheduler as scheduler
INFO ... - Setting up web info server, using web-root directory ...
INFO ... - Web info server will display information about nephele job-manager on localhost, port 8081.
INFO ... - Starting web info server for JobManager on port 8081
```

JobManager 同时会在8081端口上启动一个web前端，你可以通过 http://localhost:8081 来访问。

{% top %}


## Flink on Windows

如果你想要在 Windows 上运行 Flink，你需要如上文所述地下载、解压、配置 Flink 压缩包。之后，你可以使用使用 Windows 批处理文件（.bat文件）或者使用 **Cygwin** 运行 Flink 的 JobMnager。

### 使用 Windows 批处理文件启动

使用 Windows 批处理文件本地模式启动Flink，首先打开命令行窗口，进入 Flink 的 `bin/` 目录，然后运行 `start-local.bat` 。

注意：Java运行环境必须已经加到了 Windows 的`%PATH%`环境变量中。按照[本指南](http://www.java.com/en/download/help/path.xml)添加 Java 到`%PATH%`环境变量中。

```bash
$ cd flink
$ cd bin
$ start-local.bat
Starting Flink job manager. Webinterface by default on http://localhost:8081/.
Do not close this batch window. Stop job manager by pressing Ctrl+C.
```

之后，你需要打开新的命令行窗口，并运行`flink.bat`。


{% top %}


### 使用 Cygwin 和 Unix 脚本启动

使用 Cygwin 你需要打开 Cygwin 的命令行，进入 Flink 目录，然后运行`start-local.sh`脚本：

```bash
$ cd flink
$ bin/start-local.sh
Starting Nephele job manager
```

{% top %}


### 从 Git 安装 Flink

如果你是从 git 安装的 Flink，而且使用的 Windows git shell，Cygwin会产生一个类似于下面的错误：

```bash
c:/flink/bin/start-local.sh: line 30: $'\r': command not found
```

这个错误的产生是因为 git 运行在 Windows 上时，会自动地将 UNIX 换行转换成 Windows 换行。问题是，Cygwin 只认 Unix 换行。解决方案是调整 Cygwin 配置来正确处理换行。步骤如下：

1.&nbsp;打开 Cygwin 命令行

2.&nbsp;确定 home 目录，通过输入

```bash
cd;pwd
```
  
它会返回 Cygwin 根目录下的一个路径。
  
3.&nbsp;在home目录下，使用 NotePad, WordPad 或者其他编辑器打开`.bash_profile`文件，然后添加如下内容到文件末尾：（如果文件不存在，你需要创建它）

```bash
export SHELLOPTS
set -o igncr
```

保存文件，然后打开一个新的bash窗口。


{% top %}

