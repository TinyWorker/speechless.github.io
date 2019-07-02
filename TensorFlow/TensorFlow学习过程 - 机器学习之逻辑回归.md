## TensorFlow学习过程 - 机器学习之逻辑回归与分类 [【上页】](https://tinyworker.github.io/TensorFlow/index)  ##


逻辑回归是一种极其高效的概率计算机制。通常我们对概率的使用有两种方式：

- 按原样。
- 转换为二元类别。

假设有个逻辑回归模型来预测狗在半夜发出叫声的概率为p(bark|night)，如果模型预测值为0.05，那么一年内狗会大约叫18次。

那么为什么逻辑回归模型会确保输出值在0-1间，我们先来看S型函数，其输出值正好具有这些特性，其定义为：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>y</mi>
  <mo>=</mo>
  <mfrac>
    <mn>1</mn>
    <mrow>
      <mn>1</mn>
      <mo>+</mo>
      <msup>
        <mi>e</mi>
        <mrow class="MJX-TeXAtom-ORD">
          <mo>&#x2212;<!-- − --></mo>
          <mi>z</mi>
        </mrow>
      </msup>
    </mrow>
  </mfrac>
</math>

S型函数会产生以下曲线图：
![](https://i.imgur.com/8FALtiX.png)

如果Z表示逻辑回归模型的线性层输出，则S型函数会生成一个介于0-1的值，用数学方法表示为：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <msup>
    <mi>y</mi>
    <mo>&#x2032;</mo>
  </msup>
  <mo>=</mo>
  <mfrac>
    <mn>1</mn>
    <mrow>
      <mn>1</mn>
      <mo>+</mo>
      <msup>
        <mi>e</mi>
        <mrow class="MJX-TeXAtom-ORD">
          <mo>&#x2212;<!-- − --></mo>
          <mo stretchy="false">(</mo>
          <mi>z</mi>
          <mo stretchy="false">)</mo>
        </mrow>
      </msup>
    </mrow>
  </mfrac>
</math>
其中，y`是逻辑回归模型针对特定样本的输出，Z是b+w1x1+...+wnxn，b是偏差，w是权重，x是特征值。
同时，Z也称为对数几率，因为S型函数的反函数表明，Z可定义为标签“1”的概率除以标签“0”的概率得出值的对数：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>z</mi>
  <mo>=</mo>
  <mi>l</mi>
  <mi>o</mi>
  <mi>g</mi>
  <mo stretchy="false">(</mo>
  <mfrac>
    <mi>y</mi>
    <mrow>
      <mn>1</mn>
      <mo>&#x2212;<!-- − --></mo>
      <mi>y</mi>
    </mrow>
  </mfrac>
  <mo stretchy="false">)</mo>
</math>

### 逻辑回归的损失函数 ###
线性回归的损失函数是平方损失，而逻辑回归的损失函数是**对数损失函数**，定义如下：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>L</mi>
  <mi>o</mi>
  <mi>g</mi>
  <mi>L</mi>
  <mi>o</mi>
  <mi>s</mi>
  <mi>s</mi>
  <mo>=</mo>
  <munder>
    <mo>&#x2211;<!-- ∑ --></mo>
    <mrow class="MJX-TeXAtom-ORD">
      <mo stretchy="false">(</mo>
      <mi>x</mi>
      <mo>,</mo>
      <mi>y</mi>
      <mo stretchy="false">)</mo>
      <mo>&#x2208;<!-- ∈ --></mo>
      <mi>D</mi>
    </mrow>
  </munder>
  <mo>&#x2212;<!-- − --></mo>
  <mi>y</mi>
  <mi>l</mi>
  <mi>o</mi>
  <mi>g</mi>
  <mo stretchy="false">(</mo>
  <msup>
    <mi>y</mi>
    <mo>&#x2032;</mo>
  </msup>
  <mo stretchy="false">)</mo>
  <mo>&#x2212;<!-- − --></mo>
  <mo stretchy="false">(</mo>
  <mn>1</mn>
  <mo>&#x2212;<!-- − --></mo>
  <mi>y</mi>
  <mo stretchy="false">)</mo>
  <mi>l</mi>
  <mi>o</mi>
  <mi>g</mi>
  <mo stretchy="false">(</mo>
  <mn>1</mn>
  <mo>&#x2212;<!-- − --></mo>
  <msup>
    <mi>y</mi>
    <mo>&#x2032;</mo>
  </msup>
  <mo stretchy="false">)</mo>
</math>
其中y是有标签样本的标签，在逻辑回归中，y的每个值必须是0或1，y`是对于特征集的预测值（0-1）。

对数损失函数的方程式与 Shannon 信息论中的熵测量密切相关。它也是似然函数的负对数（假设“y”属于伯努利分布）。实际上，最大限度地降低损失函数的值会生成最大的似然估计值。


### 逻辑回归的正则化 ###
正则化对于逻辑回归很重要，因为逻辑回归的渐近性会不断促使损失在高纬度空间达到0，因此，大多数逻辑回归模型会使用策略来降低复杂度：

- L2正则化。
- 早停法，即限制训练步数或学习速率。

### 指定阈值 ###
阈值是针对二元分类的一种映射方案，模型返回预测分数后，根据阈值来判定属于哪种分类，阈值根据具体的问题来调整。

### 真与假，正类别与负类别 ###
- 正类别，在二元分类中是我们要寻找的类别。
- 负类别，在二元分类中非目标的对象。
- 真正例，是指模型将正类别样本正确的预测为正类别。
- 真负例，是指模型将负类别样本正确的预测为负类别。
- 假正例，是指模型将负类别样本错误的预测为正类别。
- 假负例，是指模型将正类别样本错误的预测为负类别。


**准确率**就是真正例和真负例的和（正确的结果）除以样例总数的比率。
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mtext>Accuracy</mtext>
  <mo>=</mo>
  <mfrac>
    <mrow>
      <mi>T</mi>
      <mi>P</mi>
      <mo>+</mo>
      <mi>T</mi>
      <mi>N</mi>
    </mrow>
    <mrow>
      <mi>T</mi>
      <mi>P</mi>
      <mo>+</mo>
      <mi>T</mi>
      <mi>N</mi>
      <mo>+</mo>
      <mi>F</mi>
      <mi>P</mi>
      <mo>+</mo>
      <mi>F</mi>
      <mi>N</mi>
    </mrow>
  </mfrac>
</math>

**精确率**是指在被识别为正类别的样本中，确定为正类别的比例。
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mtext>Precision</mtext>
  <mo>=</mo>
  <mfrac>
    <mrow>
      <mi>T</mi>
      <mi>P</mi>
    </mrow>
    <mrow>
      <mi>T</mi>
      <mi>P</mi>
      <mo>+</mo>
      <mi>F</mi>
      <mi>P</mi>
    </mrow>
  </mfrac>
</math>

**召回率**是所有正类别样本中，被正确识别为正类别的比例。
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mtext>&#x53EC;&#x56DE;&#x7387;</mtext>
  <mo>=</mo>
  <mfrac>
    <mrow>
      <mi>T</mi>
      <mi>P</mi>
    </mrow>
    <mrow>
      <mi>T</mi>
      <mi>P</mi>
      <mo>+</mo>
      <mi>F</mi>
      <mi>N</mi>
    </mrow>
  </mfrac>
</math>

### ROC曲线 ###
ROC曲线是一种显示分类模型在所有分类阈值下的效果图表。该曲线绘制了两个参数，真正例率（TPR），假正例率（FPR）。

真正例率与召回率同义，因此定义为为：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>T</mi>
  <mi>P</mi>
  <mi>R</mi>
  <mo>=</mo>
  <mfrac>
    <mrow>
      <mi>T</mi>
      <mi>P</mi>
    </mrow>
    <mrow>
      <mi>T</mi>
      <mi>P</mi>
      <mo>+</mo>
      <mi>F</mi>
      <mi>N</mi>
    </mrow>
  </mfrac>
</math>

假正例率定义为：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>F</mi>
  <mi>P</mi>
  <mi>R</mi>
  <mo>=</mo>
  <mfrac>
    <mrow>
      <mi>F</mi>
      <mi>P</mi>
    </mrow>
    <mrow>
      <mi>F</mi>
      <mi>P</mi>
      <mo>+</mo>
      <mi>T</mi>
      <mi>N</mi>
    </mrow>
  </mfrac>
</math>

### 曲线下面积（AUC）：ROC ###
曲线下面积表示“ROC曲线下面积”，该面积测量是从（0,0）到（1,1）之间整个ROC曲线以下的二维面积。

曲线下面积可以表示随机正类别样本位于随机负类别样本右侧的概率，则当下面积为0时代表100%错误，当下面积为1时代表100%正确。

曲线下面积有两个特点：

- 尺度不变，它用于测量预测的排名情况，而不是其绝对值。
- 分类阈值不变，测量的是模型预测的质量，而不考虑所选的分类阈值。

也因此有其局限性，因为不是所有场景都希望尺度和分类阈值保持不变的。
