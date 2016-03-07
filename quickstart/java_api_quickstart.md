---
title: "Quickstart: Java API"
# Top navigation
top-nav-group: quickstart
top-nav-pos: 3
top-nav-title: Java API
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

简单几步开启你的Flink Java程序。


## 要求

唯一的要求是使用 __Maven 3.0.4__ (或者更高) 和 __Java 7.x__ (或者更高) 版本。

## 创建工程

使用下列命令中的一个 __创建工程__:

<ul class="nav nav-tabs" style="border-bottom: none;">
    <li class="active"><a href="#maven-archetype" data-toggle="tab">使用 <strong>Maven archetypes</strong></a></li>
    <li><a href="#quickstart-script" data-toggle="tab">运行 <strong>quickstart script</strong></a></li>
</ul>
<div class="tab-content">
    <div class="tab-pane active" id="maven-archetype">
    {% highlight bash %}
    $ mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-java      \
      -DarchetypeVersion={{site.version}}
    {% endhighlight %}
        这种创建方式允许你 <strong>给新创建的工程命名</strong>。 他会提示你输入 groupId、 artifactId， 以及 package name。
    </div>
    <div class="tab-pane" id="quickstart-script">
    {% highlight bash %}
    $ curl https://flink.apache.org/q/quickstart.sh | bash
    {% endhighlight %}
    </div>
</div>

## 检查工程

您的工作目录中会出现一个新的目录。如果你使用了 _curl_ 建立工程，这个目录就是 `quickstart`。否则，就以你输入的 artifactId 命名。

这个示例工程是一个 __Maven 工程__, 包含两个类。 _Job_ 是一个基本的框架程序， _WordCountJob_ 是一个示例。 请注意，这两个类的_main_方法允许你在开发/测试模式下启动Flink。

推荐 __把这个工程导入你的IDE__ ，进行测试和开发。 如果用的是Eclipse， 可以用 [m2e plugin](http://www.eclipse.org/m2e/)  [导入Maven工程](http://books.sonatype.com/m2eclipse-book/reference/creating-sect-importing-projects.html#fig-creating-import)。有些Eclipse默认捆绑了这个插件，有些的需要你手动安装。 IntelliJ IDE 也提供了对Maven工程的支持。


给 Mac OS X 用户的提示：默认的JVM 堆内存对Flink来说太小了，你必须手动调高它。 在Eclipse里，选择 “运行时配置” -> 参数， 在“VM 参数” 里写入： "-Xmx800m"。

## 构建工程

如果你想要 __构建你的工程__, 进入工程目录，输入 `mvn clean install -Pbuild-jar` 命令。 你会__找到一个jar包__：__target/your-artifact-id-1.0-SNAPSHOT.jar__，它可以在任意Flink机器运行。 还有一个 fat-jar，  __target/your-artifact-id-1.0-SNAPSHOT-flink-fat-jar.jar__，包含了所有添加到Maven工程的依赖。

## 下一步

编写你的应用程序

Quickstart包含了一个WordCount的实现，也就是大数据处理系统的"Hello World"。WordCount的目标是计算文本中单词的出现的频率。比如： 单词"the"或者"house"在所有的维基百科文本中出现了多少次。

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

下面的代码就是Quickstart工程的WordCount实现，它使用两种操作(FlatMap and Reduce)处理了一些文本，并且在标准输出中打印了单词的计数结果。

~~~java
public class WordCount {
  
  public static void main(String[] args) throws Exception {
    
    // set up the execution environment
    final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
    
    // get input data
    DataSet<String> text = env.fromElements(
        "To be, or not to be,--that is the question:--",
        "Whether 'tis nobler in the mind to suffer",
        "The slings and arrows of outrageous fortune",
        "Or to take arms against a sea of troubles,"
        );
    
    DataSet<Tuple2<String, Integer>> counts = 
        // split up the lines in pairs (2-tuples) containing: (word,1)
        text.flatMap(new LineSplitter())
        // group by the tuple field "0" and sum up tuple field "1"
        .groupBy(0)
        .aggregate(Aggregations.SUM, 1);

    // emit result
    counts.print();
  }
}
~~~

这些操作是在专门的类中定义的，下面是LineSplitter类。

~~~java
public class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {

  @Override
  public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
    // normalize and split the line into words
    String[] tokens = value.toLowerCase().split("\\W+");
    
    // emit the pairs
    for (String token : tokens) {
      if (token.length() > 0) {
        out.collect(new Tuple2<String, Integer>(token, 1));
      }
    }
  }
}
~~~

在{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/wordcount/WordCount.java "GitHub"%}中检索所有的示例代码。 
想要完整的API，看这里[Programming Guide]({{ site.baseurl }}/apis/programming_guide.html) 和这里 [further example programs](examples.html). 如果你有任何疑问，咨询我们的 [Mailing List](http://mail-archives.apache.org/mod_mbox/flink-dev/)。我很高兴可以提供帮助。

