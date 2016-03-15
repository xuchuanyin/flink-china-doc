---
title: Build Flink from Source
top-nav-group: setup
top-nav-pos: 1
top-nav-title: Build Flink from Source
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

本文主要介绍如何从源码构建Flink {{ site.version }}

* This will be replaced by the TOC
{:toc}

## Flink 构建

构建 Flink 之前你需要获得源代码。 [下载发布版本源代码]({{ site.download_url }}) 或者 [克隆 git 版本库]({{ site.github_url }}) 均可。

另外你需要 **Maven 3** 和 **JDK**。 构建Flink **至少需要 Java 7**。 我们推荐使用 Java 8。

从git版本库克隆源码：

~~~bash
git clone {{ site.github_url }}
~~~

以最简单的方式构建 Flink 只需运行：

~~~bash
mvn clean install -DskipTests
~~~

这条命令的意思是[Maven](http://maven.apache.org) (`mvn`)首先移除所有已经存在的构建(`clean`)并且创建一个新的Flink二进制发布版本(`install`)。`-DskipTests`参数禁止Maven执行测试程序。

以默认方式构建源码将包含Hadoop 2 YARN客户端

{% top %}

## Hadoop 版本

{% info %} 大多数用户并不需要手动执行此操作。 [download page]({{ site.download_url }})  含有常见Hadoop 版本的 Flink 二进制包。

Flink 所依赖的 HDFS 和 YARN 均来自于[Apache Hadoop](http://hadoop.apache.org)。目前存在多个不同Hadoop版本（包括上游项目及不同Hadoop发行版）。如果使用错误的版本组合，可能会导致异常。

我们需要区分两个Hadoop主要发布版本：
- **Hadoop 1**, 以0或1开头的所有版本, 如 *0.20*, *0.23* 和 *1.2.1*。
- **Hadoop 2**, 以2开头的所有版本, 如 *2.6.0*。

Hadoop 1 和 Hadoop 2 中间主要区别在于是否使用 [Hadoop YARN](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html)管理Hadoop集群资源。

**默认情况下, 构建 Flink 将使用 Hadoop 2 依赖**。

### Hadoop 1

使用下列命令构建基于Hadoop 1版本的Flink：

~~~bash
mvn clean install -DskipTests -Dhadoop.profile=1
~~~

`-Dhadoop.profile=1` 指明Maven使用 Hadoop 1构建 Flink。需要注意的是在不同Hadoop版本中包含的特性可能发生改变。特别是在Hadoop 1 版本中不支持 YARN 和 HBase。

### Hadoop 2.x

构建 Flink Hadoop 2.X版本只支持 Hadoop 2.3.0 以上。
你可以指定一个Hadoop版本用于源码构建：

~~~bash
mvn clean install -DskipTests -Dhadoop.version=2.6.1
~~~

#### Hadoop 2.3.0 之前版本

Hadoop 2.X 版本仅在 2.3.0 之后支持 YARN 特性。如果需要使用低于 2.3.0 的版本，你可以使用 `-P!include-yarn` 参数移除对于 YARN 的支持。

使用下列命令将会使用Hadoop `2.2.0`版本进行构建：

~~~bash
mvn clean install -Dhadoop.version=2.2.0 -P!include-yarn
~~~

### 发行商版本

使用下列命令指定一个发行商版本：

~~~bash
mvn clean install -DskipTests -Pvendor-repos -Dhadoop.version=2.6.1-cdh5.0.0
~~~


使用 `-Pvendor-repos` 表示启动了包含Cloudera, Hortonworks 和 MapR这些当前流行的Hadoop发行版本 Maven [build profile](http://maven.apache.org/guides/introduction/introduction-to-profiles.html)

{% top %}

## Scala 版本

{% info %} 用户如果仅使用Java API则可以 *忽略* 这部分内容。

Flink 有一套[Scala](http://scala-lang.org)编写的API，代码库和运行时模块。用户在使用Scala API和代码库时需要和自己工程中的Scala版本相匹配。

**默认情况下, Flink 使用 Scala 2.10 版本进行构建**。你可以使用如下脚本变更默认Scala版本，用来基于Scala *2.11* 构建Flink。

~~~bash
# 从 Scala 2.10 版本 切换到 Scala 2.11 版本
tools/change-scala-version.sh 2.11
# 基于Scala 2.11 版本构建Flink
mvn clean install -DskipTests
~~~

为了根据特定 Scala 版本进行构建，需要切换到相应二进制版本并添加 *语言版本* 作为附加构建属性。
例如，使用 Scala 2.11.4 版本进行构建需要执行：

~~~bash
# 切换到 Scala 2.11 版本
tools/change-scala-version.sh 2.11
# 使用 Scala 2.11.4 版本进行构建
mvn clean install -DskipTests -Dscala.version=2.11.4
~~~

Flink 基于 Scala *2.10* 版本开发且额外经过 Scala *2.11* 版本测试。这两个版本是支持的。更早的版本 (如 Scala *2.9*) *不再*支持.

是否兼容 Scala 的新版本，取决于 Flink 所使用的语言特性是否有重大改变, 以及 Flink 所依赖的组件在新版本Scala的兼容情况。 由Scala编写的依赖库包括*Kafka*, *Akka*, *Scalatest*, 和 *scopt*。

{% top %}

## 加密文件系统

如果你的 home 目录是加密文件系统可能会发生 `java.io.IOException: File name too long` 异常. 一些加密文件系统，比如Ubuntu所使用的encfs，不允许长文件名，这会导致这种错误。

修改办法是添加如下配置:

~~~xml
<args>
    <arg>-Xmax-classfile-name</arg>
    <arg>128</arg>
</args>
~~~

进入导致这个错误的模块的 `pom.xml` 文件的编译配置中。如果错误出现在 `flink-yarn` 模块中，上面的配置需要加入到 `scala-maven-plugin` 中`<configuration>` 标签下。更多信息见 [这个 issue](https://issues.apache.org/jira/browse/FLINK-2003)。

{% top %}

## 内部细节

[properties](http://maven.apache.org/pom.html#Properties) 和 [build profiles](http://maven.apache.org/guides/introduction/introduction-to-profiles.html) 用来控制Maven 的构建流程。 Flink 有两个profile，分别用来控制Hadoop 1 和 Hadoop 2。在 `hadoop2` 配置打开启的情况下（默认开启），系统会构建 YARN 客户端。  

构建时设置 `-Dhadoop.profile=1` 将使用 `hadoop1` profile。根据 profile ，可以设置两个Hadoop版本。对于 `hadoop1` 默认使用 1.2.1版本，`hadoop2` 默认使用2.3.0版本。

你可以使用 `hadoop-two.version`(或者 `hadoop-one.version`) 属性变更版本。例如 `-Dhadoop-two.version=2.4.0`。

{% top %}
