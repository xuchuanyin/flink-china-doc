---
title: "Fault Tolerance"

# Sub-level navigation
sub-nav-group: batch
sub-nav-pos: 2
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

>本文暂未校对

flink 的容错机制保障出错时恢复程序并继续执行。 这些错误包括机器硬件错误，网络错误，程序错误等。

* This will be replaced by the TOC
{:toc}

Batch Processing Fault Tolerance (DataSet API)
----------------------------------------------


*DataSet API* 程序的容错就是重试失败的任务。 在定义任务前， flink 会设置重试的最大次数通过*execution retries* 参数。 
*0* 表示关掉容错， 如果想要打开容错， *execution retries*  这个值必须设置大于0， 一半会设置为3.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setNumberOfExecutionRetries(3);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setNumberOfExecutionRetries(3)
{% endhighlight %}
</div>
</div>


其实可以在`flink-conf.yaml`设置默认的重试次数
~~~
execution-retries.default: 3
~~~


Retry Delays
------------

重试执行可以被配置延迟执行。 这意味着当发生失败时，不是立即重新执行，而是等待一段时间。

等待一段时间非常有用， 当程序同外部系统交互式或者堆积的事物需要一个timeout在重新运行前。

可以在每个程序里面设置delay时间（例子是使用DataStream， DataSet其实是类似的）：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setExecutionRetryDelay(5000); // 5000 milliseconds delay
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.getConfig.setExecutionRetryDelay(5000) // 5000 milliseconds delay
{% endhighlight %}
</div>
</div>

同样可以在`flink-conf.yaml`中设置系统默认的延迟时间：

~~~
execution-retries.delay: 10 s
~~~

{% top %}
