---
title: "快速起步: Scala API"
# Top navigation
top-nav-group: quickstart
top-nav-pos: 4
top-nav-title: Scala API
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


## 构建工具

Flink 工程可以用不同的构建工具构建。
为了让用户快速开始, Flink 为以下构建工具提供了工程模版：

- [SBT](#sbt)
- [Maven](#maven)

这些模版帮助用户创建工程结构并初始化一些构建文件。

## SBT

### 创建工程

<ul class="nav nav-tabs" style="border-bottom: none;">
    <li class="active"><a href="#giter8" data-toggle="tab">使用 <strong>Giter8</strong></a></li>
    <li><a href="#clone-repository" data-toggle="tab">克隆 <strong>repository</strong></a></li>
    <li><a href="#quickstart-script-sbt" data-toggle="tab">运行 <strong>quickstart 脚本</strong></a></li>
</ul>

<div class="tab-content">
    <div class="tab-pane active" id="giter8">
    {% highlight bash %}
    $ g8 tillrohrmann/flink-project
    {% endhighlight %}
    这将在 <strong>指定</strong> 的工程目录下，从 <a href="https://github.com/tillrohrmann/flink-project.g8">flink-project 模版</a> 创建一个 Flink 工程。
    如果你没有安装 <a href="https://github.com/n8han/giter8">giter8</a>, 请参照此 <a href="https://github.com/n8han/giter8#installation">安装指南</a>。
    </div>
    <div class="tab-pane" id="clone-repository">
    {% highlight bash %}
    $ git clone https://github.com/tillrohrmann/flink-project.git
    {% endhighlight %}
    这将在 <strong>flink-project</strong> 目录下创建 Flink 工程。
    </div>
    <div class="tab-pane" id="quickstart-script-sbt">
    {% highlight bash %}
    $ bash <(curl https://flink.apache.org/q/sbt-quickstart.sh)
    {% endhighlight %}
    这将在 <strong>指定</strong> 的工程目录下创建 Flink 工程
    </div>
</div>

### 构建工程

为了构建工程， 只需要执行 `sbt clean assembly` 命令。
这将会在目录 __target/scala_your-major-scala-version/__ 下创建 fat-jar __your-project-name-assembly-0.1-SNAPSHOT.jar__。

### 运行工程

如果想要运行你的工程， 输入 `sbt run` 命令。

作业将会默认运行在 `sbt` 运行的 JVM 上。
如果你想让作业运行在不同的 JVM 上， 将以下代码加入至 `build.sbt`：

~~~scala
fork in run := true
~~~
 

#### IntelliJ

我们推荐使用 [IntelliJ](https://www.jetbrains.com/idea/) 作为你的 Flink job 开发环境。
首先， 你需要将新创建的工程导入至 IntelliJ。
你可以通过打开 `File -> New -> Project from Existing Sources...` ， 然后选择你的工程目录来导入工程。
IntelliJ 会检测到 `build.sbt` 文件并自动导入。

如果你想要运行 Flink 作业, 建议将 `mainRunner` 模块 作为 __Run/Debug Configuration__ 的 classpath 路径。
这将保证在执行过程中， 所有标识为"provided"的依赖都可用。
你可以通过打开 `Run -> Edit Configurations...` 来配置 __Run/Debug Configurations__， 然后从 _Use classpath of module_ 下拉框中选择 `mainRunner`。

#### Eclipse

如果你想要将新建的工程导入至 [Eclipse](https://eclipse.org/), 首先你得为它创建 Eclipse 工程文件。
这些工程文件可以通过 [sbteclipse](https://github.com/typesafehub/sbteclipse) 插件来创建。
将以下代码加入至 `PROJECT_DIR/project/plugins.sbt` 文件:

~~~bash
addSbtPlugin("com.typesafe.sbteclipse" % "sbteclipse-plugin" % "4.0.0")
~~~

在 `sbt` 交互式环境下使用下列命令创建 Eclipse 工程文件：

~~~bash
> eclipse
~~~

现在你可以通过打开 `File -> Import... -> Existing Projects into Workspace` 并选择你的工程目录， 将之导入至 Eclipse。

## Maven

### 要求

唯一的要求是使用 __Maven 3.0.4__ (或者更高) 和 __Java 7.x__ (或者更高) 版本。


### 创建工程

使用下列命令中的一个 __创建工程__:

<ul class="nav nav-tabs" style="border-bottom: none;">
    <li class="active"><a href="#maven-archetype" data-toggle="tab">使用 <strong>Maven archetypes</strong></a></li>
    <li><a href="#quickstart-script" data-toggle="tab">运行 <strong>quickstart 脚本</strong></a></li>
</ul>

<div class="tab-content">
    <div class="tab-pane active" id="maven-archetype">
    {% highlight bash %}
    $ mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-scala     \
      -DarchetypeVersion={{site.version}}
    {% endhighlight %}
    这种创建方式允许你 <strong>给新创建的工程命名</strong>。它会提示你输入 groupId、 artifactId， 以及 package name。
    </div>
    <div class="tab-pane" id="quickstart-script">
{% highlight bash %}
$ curl https://flink.apache.org/q/quickstart-scala.sh | bash
{% endhighlight %}
    </div>
</div>


### 检查工程

您的工作目录中会出现一个新的目录。如果你使用了 _curl_ 建立工程, 这个目录就是 `quickstart`。 否则， 就以你输入的 artifactId 命名。

这个示例工程是一个包含两个类的  __Maven 工程__。 _Job_ 是一个基本的框架程序， _WordCountJob_ 是一个示例程序。 请注意，这两个类的 _main_ 方法都允许你在开发/测试模式下启动 Flink。

推荐 __把这个工程导入你的 IDE__ 进行测试和开发。 如果用的是 Eclipse， 你需要从常用的 Eclipse 更新站点上下载并安装以下插件:

* _Eclipse 4.x_
  * [Scala IDE](http://download.scala-ide.org/sdk/e38/scala210/stable/site)
  * [m2eclipse-scala](http://alchim31.free.fr/m2e-scala/update-site)
  * [Build Helper Maven Plugin](https://repository.sonatype.org/content/repositories/forge-sites/m2e-extras/0.15.0/N/0.15.0.201206251206/)
* _Eclipse 3.7_
  * [Scala IDE](http://download.scala-ide.org/sdk/e37/scala210/stable/site)
  * [m2eclipse-scala](http://alchim31.free.fr/m2e-scala/update-site)
  * [Build Helper Maven Plugin](https://repository.sonatype.org/content/repositories/forge-sites/m2e-extras/0.14.0/N/0.14.0.201109282148/)

IntelliJ IDE 也支持 Maven 并提供了一个用于 Scala 开发的插件。


### 构建工程

如果想要 __构建你的工程__， 进入工程目录并输入 `mvn clean package -Pbuild-jar` 命令。 你会 __找到一个jar包__: __target/your-artifact-id-1.0-SNAPSHOT.jar__ , 它可以在任意 Flink 集群上运行。还有一个 fat-jar,  __target/your-artifact-id-1.0-SNAPSHOT-flink-fat-jar.jar__， 包含了所有添加到 Maven 工程的依赖。

## 下一步

编写你自己的应用程序！

Quickstart 工程包含了一个 WordCount 的实现， 也就是大数据处理系统的 “Hello World”。 WordCount 的目标是计算文本中单词出现的频率。 比如： 单词 “the” 或者 “house” 在所有的维基百科文本中出现了多少次。

__示例输入__:

~~~bash
big data is big
~~~

__示例输出__:

~~~bash
big 2
data 1
is 1
~~~

下面的代码就是 Quickstart 工程的 WordCount 实现， 它使用两种操作( FlatMap 和 Reduce )处理了一些文本， 并且在标准输出中打印了单词的计数结果。

~~~scala
object WordCountJob {
  def main(args: Array[String]) {

    // set up the execution environment
    val env = ExecutionEnvironment.getExecutionEnvironment

    // get input data
    val text = env.fromElements("To be, or not to be,--that is the question:--",
      "Whether 'tis nobler in the mind to suffer", "The slings and arrows of outrageous fortune",
      "Or to take arms against a sea of troubles,")

    val counts = text.flatMap { _.toLowerCase.split("\\W+") }
      .map { (_, 1) }
      .groupBy(0)
      .sum(1)

    // emit result
    counts.print()
  }
}
~~~

在 {% gh_link /flink-examples/flink-scala-examples/src/main/scala/org/apache/flink/examples/scala/wordcount/WordCount.scala "GitHub" %} 中查看完整的示例代码。

想要了解完整的 API, 可以查阅  [编程指南]({{ site.baseurl }}/apis/programming_guide.html) 和 [更多的实例]({{ site.baseurl }}/apis/examples.html)。 如果你有任何疑问， 咨询我们的 [Mailing List](http://mail-archives.apache.org/mod_mbox/flink-dev/)， 我们很高兴可以提供帮助。

