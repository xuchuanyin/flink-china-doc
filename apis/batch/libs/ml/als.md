---
mathjax: include
title: 交替最小二乘法(ALS算法)

# Sub navigation
sub-nav-group: batch
sub-nav-parent: flinkml
sub-nav-title: ALS
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

交替最小二乘算法(ALS)将一个矩阵$R$分解成$U$和$V$两个矩阵，使得$R$、$U$、$V$满足：$R \approx U^TV$。
其中未知的行向量作为算法的参数输入，被称为隐藏因子。

矩阵分解常用于推荐场景中，因此$U$、$V$又分别被称为用户矩阵和商品矩阵。
用户矩阵的第i列用$u_i$来表示，商品矩阵的第i列用$v_i$来表示。
矩阵$R$又被称为评分矩阵，且有：$$(R)_{i,j} = r_{i,j}$$。

为了得到用户和商品矩阵，需要对以下公式进行最小化求值：

$$\arg\min_{U,V} \sum_{\{i,j\mid r_{i,j} \not= 0\}} \left(r_{i,j} - u_{i}^Tv_{j}\right)^2 + 
\lambda \left(\sum_{i} n_{u_i} \left\lVert u_i \right\rVert^2 + \sum_{j} n_{v_j} \left\lVert v_j \right\rVert^2 \right)$$

其中$\lambda$是正则化项的系数，$$n_{u_i}$$为用户$i$有过评分的商品数，$$n_{v_j}$$为商品$j$总计被评分的次数。
这种防止过拟合的正则化方案被称作加权$\lambda$正则化，具体细节可以参考论文：[Zhou et al.](http://dx.doi.org/10.1007/978-3-540-68880-8_32)。

通过确定矩阵$U$、$V$两个矩阵之一，便能得到一个二次型，从而可以直接求解另一个矩阵。
通过对$U$和$V$矩阵的交替迭代求解，可以逐步优化分解的矩阵，且能够保证损失函数是单调递减的。

矩阵$R$以稀疏矩阵来表示，矩阵的元组$(i, j, r)$中$i$表示行下标， $j$表示列下标，$r$为矩阵在$(i,j)$的值。

## 操作

由于`ALS`类是一个`Predictor`，因此支持`fit`和`predict`操作。

### 训练

ALS通过稀疏的评分矩阵来进行训练：
* `fit: DataSet[(Int, Int, Double)] => Unit` 

### 预测

ALS对每个输入tuple的行列下标预测评分： 

* `predict: DataSet[(Int, Int)] => DataSet[(Int, Int, Double)]`

## 参数

最小二乘法的实现可以通过以下参数来控制：

   <table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">说明</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>NumFactors</strong></td>
        <td>
          <p>
          模型的隐藏因子数量，等价于求解出来的用户和商品特征向量的维数。
            (默认值：<strong>10</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>Lambda</strong></td>
        <td>
          <p>
          正则化因子。通过调整此参数，可以避免由于强泛化导致的过拟合或者欠拟合。
            (默认值：<strong>1</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>Iterations</strong></td>
        <td>
          <p>
          最大迭代次数。
            (默认值：<strong>10</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>Blocks</strong></td>
        <td>
          <p>
          用户和商品矩阵划分的block数量。block数量越少，则被冗余发送的数据也越少，但是较大的block会导致更大的更新消息被存储在堆内存中。
          如果算法由于内存不足抛出OutOfMemoryException失败，则需要增加block的数量。
            (默认值：<strong>None</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>Seed</strong></td>
        <td>
          <p>
          随机种子，用于生成算法中初始商品矩阵。
            (默认值：<strong>0</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>TemporaryPath</strong></td>
        <td>
          <p>
          用于存放中间结果的临时目录。
          如果这个值被设置，则算法会被分成两个预处理步骤：ALS迭代和(预处理中的)后处理，其中后处理步骤计算出最后的ALS矩阵。
          预处理步骤通过给定的评分矩阵，计算出<code>OutBlockInformation</code>和<code>InBlockInformation</code>，每一步的结果都会存储于指定的临时目录中。
          通过将算法分成多个比较小的步骤，Flink无须将内存分拆成多块以满足众多的操作需要，使得系统能够处理更大的单条消息，提升了整体的性能。
            (默认值：<strong>None</strong>)
          </p>
        </td>
      </tr>
    </tbody>
  </table>

## 代码示例

{% highlight scala %}
// Read input data set from a csv file
val inputDS: DataSet[(Int, Int, Double)] = env.readCsvFile[(Int, Int, Double)](
  pathToTrainingFile)

// Setup the ALS learner
val als = ALS()
.setIterations(10)
.setNumFactors(10)
.setBlocks(100)
.setTemporaryPath("hdfs://tempPath")

// Set the other parameters via a parameter map
val parameters = ParameterMap()
.add(ALS.Lambda, 0.9)
.add(ALS.Seed, 42L)

// Calculate the factorization
als.fit(inputDS, parameters)

// Read the testing data set from a csv file
val testingDS: DataSet[(Int, Int)] = env.readCsvFile[(Int, Int)](pathToData)

// Calculate the ratings according to the matrix factorization
val predictedRatings = als.predict(testingDS)
{% endhighlight %}
