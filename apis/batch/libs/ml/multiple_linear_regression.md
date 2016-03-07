---
mathjax: include
title: 多元线性回归

# Sub navigation
sub-nav-group: batch
sub-nav-parent: flinkml
sub-nav-title: 多元线性回归
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

## 描述

 多元线性回归的目标是找到一个最佳拟合输入数据的线性函数。通过输入数据集：$(\mathbf{x}, y)$，
 多元线性回归将得到一个向量$\mathbf{w}$，从而最小化残差平方(squared residuals)和： 

 $$ S(\mathbf{w}) = \sum_{i=1} \left(y - \mathbf{w}^T\mathbf{x_i} \right)^2$$

 通过矩阵的表示方法，得到以下公式：
 $$\mathbf{w}^* = \arg \min_{\mathbf{w}} (\mathbf{y} - X\mathbf{w})^2$$

  从而得到一个确定解：
  $$\mathbf{w}^* = \left(X^TX\right)^{-1}X^T\mathbf{y}$$

 但是，如果输入数据集过大从而导致无法完全解析所有数据时，可以通过随机梯度下降(SGD)来得到一个近似解。
 SGD首先用输入数据集的随机子集求得一个梯度，特定点$\mathbf{x}_i$上的梯度为：

  $$\nabla_{\mathbf{w}} S(\mathbf{w}, \mathbf{x_i}) = 2\left(\mathbf{w}^T\mathbf{x_i} -
    y\right)\mathbf{x_i}$$

 求解出来的梯度被归一化，公式为：$\gamma = \frac{s}{\sqrt{j}}$，其中$s$为初始步长，$j$为当前迭代次数。
 当前梯度的权重向量减去归一化后的当前梯度值，即得到下一次迭代的权重向量：

  $$\mathbf{w}_{t+1} = \mathbf{w}_t - \gamma \frac{1}{n}\sum_{i=1}^n \nabla_{\mathbf{w}} S(\mathbf{w}, \mathbf{x_i})$$

 多元线性回归可以根据输入的SGD迭代次数终止，也可以在达到给定的收敛条件后终止。其中收敛条件为两次迭代之间残差平方和满足：

  $$\frac{S_{k-1} - S_k}{S_{k-1}} < \rho$$
  
## 操作

`MultipleLinearRegression`是`Predictor`，因此支持训练和预测操作。

### 训练

多元线性回归通过输入一个`LabeledVector`来进行训练： 

* `fit: DataSet[LabeledVector] => Unit`

### 预测

多元线性回归预测所有`Vector`子类型的回归值：

* `predict[T <: Vector]: DataSet[T] => DataSet[LabeledVector]`

通过`DataSet[LabeledVector]`来预测，将得到每条数据的回归值，且返回`DataSet[(Double, Double)]`。
在每个元组中，第一个元素为对应输入`DataSet[LabeledVector]`中的真实值，第二个元素为预测值。
可以通过`(truth, prediction)`的对比来评估算法的好坏。

* `predict: DataSet[LabeledVector] => DataSet[(Double, Double)]`

## 参数列表

多元线性回归可以通过以下参数来控制：
  
   <table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">说明</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>Iterations</strong></td>
        <td>
          <p>
          最大迭代次数(默认值：<strong>10</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>Stepsize</strong></td>
        <td>
          <p>
          梯度下降的初始步长。该值确定每次进行梯度下降时，反向移动的距离。调整步长对于算法的稳定收敛及性能非常重要。
            (默认值：<strong>0.1</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>ConvergenceThreshold</strong></td>
        <td>
          <p>
          算法停止迭代的残差平方和变化程度的收敛阈值。
            (默认值：<strong>None</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>LearningRateMethod</strong></td>
        <td>
            <p>
            学习率方法，用于计算每次迭代的有效学习率。Flink ML支持的学习率方法列表见：
                <a href="optimization.html">learning rate methods</a>。
                (默认值：<strong>LearningRateMethod.Default</strong>)
            </p>
        </td>
      </tr>
    </tbody>
  </table>

## 示例代码

{% highlight scala %}
// Create multiple linear regression learner
val mlr = MultipleLinearRegression()
.setIterations(10)
.setStepsize(0.5)
.setConvergenceThreshold(0.001)

// Obtain training and testing data set
val trainingDS: DataSet[LabeledVector] = ...
val testingDS: DataSet[Vector] = ...

// Fit the linear model to the provided data
mlr.fit(trainingDS)

// Calculate the predictions for the test data
val predictions = mlr.predict(testingDS)
{% endhighlight %}
