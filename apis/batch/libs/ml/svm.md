---
mathjax: include
title: SVM using CoCoA
# Sub navigation
sub-nav-group: batch
sub-nav-parent: flinkml
sub-nav-title: SVM (CoCoA)
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

## Description
## 描述

Implements an SVM with soft-margin using the communication-efficient distributed dual coordinate
ascent algorithm with hinge-loss function.
The algorithm solves the following minimization problem:

使用具有合页损失函数的分布式对偶坐标上升算法(COCOA)来实现软间隔支持向量机
此算法解决了如下损失函数的最小化问题

$$\min_{\mathbf{w} \in \mathbb{R}^d} \frac{\lambda}{2} \left\lVert \mathbf{w} \right\rVert^2 + \frac{1}{n} \sum_{i=1}^n l_{i}\left(\mathbf{w}^T\mathbf{x}_i\right)$$

with $\mathbf{w}$ being the weight vector, $\lambda$ being the regularization constant,
$$\mathbf{x}_i \in \mathbb{R}^d$$ being the data points and $$l_{i}$$ being the convex loss
functions, which can also depend on the labels $$y_{i} \in \mathbb{R}$$.
$\mathbf{w}$ 代表权值向量，$\lambda$ 代表正则常数。$$\mathbf{x}_i \in \mathbb{R}^d$$ 代表样本值$$l_{i}$$ 代表凸损失函数,并依赖于输出分类$$y_{i} \in \mathbb{R}$$.

In the current implementation the regularizer is the $\ell_2$-norm and the loss functions are the hinge-loss functions:
在当前的实现中正则化项为L2范数，损失函数为合页损失函数

  $$l_{i} = \max\left(0, 1 - y_{i} \mathbf{w}^T\mathbf{x}_i \right)$$

With these choices, the problem definition is equivalent to a SVM with soft-margin.
Thus, the algorithm allows us to train a SVM with soft-margin.
基于前面的选择，问题的定义就等价于软间隔支持向量机(SVM)

The minimization problem is solved by applying stochastic dual coordinate ascent (SDCA).
In order to make the algorithm efficient in a distributed setting, the CoCoA algorithm calculates
several iterations of SDCA locally on a data block before merging the local updates into a
valid global state.

极小值通过SDCA算法求得
为了让算法在分布式环境下更加高效，COCOA算法首先在本地的一个数据块上计算若干次SDCA迭代，然后再将本地更新合并到有效全局状态中。
This state is redistributed to the different data partitions where the next round of local SDCA
iterations is then executed.
全局状态被重新分配到下一轮本地SDCA迭代的数据分区，然后执行。
The number of outer iterations and local SDCA iterations control the overall network costs, because
there is only network communication required for each outer iteration.
The local SDCA iterations are embarrassingly parallel once the individual data partitions have been
distributed across the cluster.
因为只有外层迭代需要网络通信，外层迭代的次数和本地SDCA迭代决定了全部的网络消耗。一旦独立的数据分区分布在集群中本地SDCA是不容易并行的。

The implementation of this algorithm is based on the work of
算法实现基于如下链接
[Jaggi et al.](http://arxiv.org/abs/1409.1458)

## Operations
## 操作

`SVM` is a `Predictor`.
As such, it supports the `fit` and `predict` operation.
`SVM` 支持学习和预测两种操作

### Fit
### 学习

SVM is trained given a set of `LabeledVector`:
SVM通过`LabeledVector`集合进行训练

* `fit: DataSet[LabeledVector] => Unit`

### Predict
### 预测

SVM predicts for all subtypes of FlinkML's `Vector` the corresponding class label:
传入任意FlinkML的`Vector`子类，SVM会预测出相应的分类标签值

* `predict[T <: Vector]: DataSet[T] => DataSet[(T, Double)]`, where the `(T, Double)` tuple
  corresponds to (original_features, label)

* `predict[T <: Vector]: DataSet[T] => DataSet[(T, Double)]`,  `(T, Double)` e
  对应 (原始输入值, 预测的分类)

If we call evaluate with a `DataSet[(Vector, Double)]`, we make a prediction on the class label
for each example, and return a `DataSet[(Double, Double)]`. In each tuple the first element
is the true value, as was provided from the input `DataSet[(Vector, Double)]` and the second element
is the predicted value. You can then use these `(truth, prediction)` tuples to evaluate
the algorithm's performance.
如果想要对模型的预测结果进行评估，可以对已正确分类的样本集做预测（prediction),传入`DataSet[(Vector, Double)]`,返回`DataSet[(Double, Double)]`。返回结构的首元素为传入参数提供的真值，第二个元素为预测值，
可以使用这个`(真值, 预测值)`集合来评估算法的准确率和执行情况
* `predict: DataSet[(Vector, Double)] => DataSet[(Double, Double)]`

## Parameters
## 参数

The SVM implementation can be controlled by the following parameters:

<table class="table table-bordered">
<thead>
  <tr>
    <th class="text-left" style="width: 20%">Parameters</th>
    <th class="text-center">Description</th>
  </tr>
</thead>

<tbody>
  <tr>
    <td><strong>Blocks</strong></td>
    <td>
      <p>
        Sets the number of blocks into which the input data will be split.
        On each block the local stochastic dual coordinate ascent method is executed.
        This number should be set at least to the degree of parallelism.
        If no value is specified, then the parallelism of the input DataSet is used as the number of blocks.
        (Default value: <strong>None</strong>)
        设定输入数据被切分后的块数量
        每块数据都会执行一个本地SDCA(随机对偶坐标上升)方法
        设定值应至少相当于并发总数
        若没有指定，那么将使用输入DataSet的并发值作为块的数量
        (默认值: <strong>None</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>Iterations</strong></td>
    <td>
      <p>
        Defines the maximum number of iterations of the outer loop method.
        In other words, it defines how often the SDCA method is applied to the blocked data.
        After each iteration, the locally computed weight vector updates have to be reduced to update the global weight vector value.
        The new weight vector is broadcast to all SDCA tasks at the beginning of each iteration.
        (Default value: <strong>10</strong>)
        定义外层方法的最大迭代次数。
        也可以认为它定义了SDCA方法应用于块数据的频繁程度。
        每次迭代后，本地计算的权值向量更新必须被归纳更新至全局的权值向量
        新的权值向量将会在每次迭代开始时广播至所有的SDCA任务
        (默认值: <strong>10</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>LocalIterations</strong></td>
    <td>
      <p>
        Defines the maximum number of SDCA iterations.
        In other words, it defines how many data points are drawn from each local data block to calculate the stochastic dual coordinate ascent.
        (Default value: <strong>10</strong>)
        定义SDCA的最大迭代次数
        也可以认为它定义了每次SDCA迭代有多少数据点会从本地数据块中被取出
        (默认值: <strong>10</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>Regularization</strong></td>
    <td>
      <p>
        Defines the regularization constant of the SVM algorithm.
        The higher the value, the smaller will the 2-norm of the weight vector be.
        In case of a SVM with hinge loss this means that the SVM margin will be wider even though it might contain some false classifications.
        (Default value: <strong>1.0</strong>)
        定义SVM算法的正则常数
        正则常数越大，权值向量的L2范数所起的作用就越小
        在使用合页损失函数的情况下，这意味着SVM支持向量的边界间隔会越来越大，哪怕这样的分隔包含了一些错误的分类
        (默认值: <strong>1.0</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>Stepsize</strong></td>
    <td>
      <p>
        Defines the initial step size for the updates of the weight vector.
        The larger the step size is, the larger will be the contribution of the weight vector updates to the next weight vector value.
        The effective scaling of the updates is $\frac{stepsize}{blocks}$.
        This value has to be tuned in case that the algorithm becomes unstable.
        (Default value: <strong>1.0</strong>)
        定义更新权值向量的初始步长。
        步长越大，每次权重向量值更新的就越多。
        $\frac{stepsize}{blocks}$这个比例实际影响着更新操作。
        如果算法变得不稳定，此值需要被调整
        (默认值: <strong>1.0</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>ThresholdValue</strong></td>
    <td>
      <p>
        Defines the limiting value for the decision function above which examples are labeled as
        positive (+1.0). Examples with a decision function value below this value are classified
        as negative (-1.0). In order to get the raw decision function values you need to indicate it by
        using the OutputDecisionFunction parameter.  (Default value: <strong>0.0</strong>)
        设定一个边界值，若决策函数返回值超过了它，则标记为正类(+1.0)。若决策函数的返回值低于它，标记为负类(-1.0)。
        若想要得到原始的决策函数返回值，需要使用OutputDecisionFunction参数来表明。(默认值: <strong>0.0</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>OutputDecisionFunction</strong></td>
    <td>
      <p>
        Determines whether the predict and evaluate functions of the SVM should return the distance
        to the separating hyperplane, or binary class labels. Setting this to true will 
        return the raw distance to the hyperplane for each example. Setting it to false will 
        return the binary class label (+1.0, -1.0) (Default value: <strong>false</strong>)
        决定SVM的预测和评估方法是返回与分离超平面的距离还是二分类的标签值。设定为true返回每个输入与超平面的原始距离。
        设置为false返回二分类标签值。(默认值: <strong>false</strong>)
      </p>
    </td>
  </tr>
  <tr>
  <td><strong>Seed</strong></td>
  <td>
    <p>
      Defines the seed to initialize the random number generator.
      The seed directly controls which data points are chosen for the SDCA method.
      (Default value: <strong>Random Long Integer</strong>)
      定义随机数生成器的种子。
      此值决定了SDCA方法将会选择哪一个数据点
      (默认值: <strong>Random Long Integer</strong>)
    </p>
  </td>
</tr>
</tbody>
</table>

## Examples
## 例子

{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.ml.math.Vector
import org.apache.flink.ml.common.LabeledVector
import org.apache.flink.ml.classification.SVM
import org.apache.flink.ml.RichExecutionEnvironment

val pathToTrainingFile: String = ???
val pathToTestingFile: String = ???
val env = ExecutionEnvironment.getExecutionEnvironment

// Read the training data set, from a LibSVM formatted file
val trainingDS: DataSet[LabeledVector] = env.readLibSVM(pathToTrainingFile)

// Create the SVM learner
val svm = SVM()
  .setBlocks(10)

// Learn the SVM model
svm.fit(trainingDS)

// Read the testing data set
val testingDS: DataSet[Vector] = env.readLibSVM(pathToTestingFile).map(_.vector)

// Calculate the predictions for the testing data set
val predictionDS: DataSet[(Vector, Double)] = svm.predict(testingDS)

{% endhighlight %}
