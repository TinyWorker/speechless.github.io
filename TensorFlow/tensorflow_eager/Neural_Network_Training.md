## 自定义训练：神经网络的构建第一原则

之前我们了解到张量是tf中的不可变、无状态对象，但在ML中，需要可变得状态：随着模型的训练，计算的预测结果应该随时间改变。所以依赖Python做状态是个很好的选择。

	# Using python state
	x = tf.zeros([10, 10])
	x += 2  # This is equivalent to x = x + 2, which does not mutate the original
	        # value of x
	print(x)

然而，tf有内置的有状态操作，这些操作要比用低阶Python更轻松些。要表示模型的权重，通常用tf的变量要更加方便和高效。
	
	v = tf.Variable(1.0)
	assert v.numpy() == 1.0
	
	# Re-assign the value
	v.assign(3.0)
	assert v.numpy() == 3.0
	
	# Use `v` in a TensorFlow operation like tf.square() and reassign
	v.assign(tf.square(v))
	assert v.numpy() == 9.0

计算梯度时会自动跟踪使用变量的计算。 对于表示嵌入的变量，TensorFlow默认会进行稀疏更新，这样可以提高计算效率和内存效率。

使用变量也是一种快速让代码的读者知道这段状态是可变的方法。

### 实例：填充一个线性模型
在学习了张量，渐变带，变量这些概念后，我们来搭建一个简单模型。会包含几个典型的步骤：

- 构建一个模型
- 选择损失函数
- 准备训练数据
- 遍历数据并使用优化器来适应变量以匹配数据

本实例中，我们针对一个简单线性模型：f(x) = x * W + b。该函数有两个变量：W和b。我们期望这两个变量为W=3.0，b=2.0

步骤一：构建模型

	class Model(object):
	  def __init__(self):
	    # Initialize variable to (5.0, 0.0)
	    # In practice, these should be initialized to random values.
	    self.W = tf.Variable(5.0)
	    self.b = tf.Variable(0.0)
	
	  def __call__(self, x):
	    return self.W * x + self.b
	
	model = Model()
	
	assert model(3.0).numpy() == 15.0

步骤二：选择损失函数，这里我们选择标准L2

	def loss(predicted_y, desired_y):
	  return tf.reduce_mean(tf.square(predicted_y - desired_y))

步骤三：准备训练数据，这里合成一点噪声。

	TRUE_W = 3.0
	TRUE_b = 2.0
	NUM_EXAMPLES = 1000
	
	inputs  = tf.random_normal(shape=[NUM_EXAMPLES])
	noise   = tf.random_normal(shape=[NUM_EXAMPLES])
	outputs = inputs * TRUE_W + TRUE_b + noise

检查模型状态：

	import matplotlib.pyplot as plt
	
	plt.scatter(inputs, outputs, c='b')
	plt.scatter(inputs, model(inputs), c='r')
	plt.show()
	
	print('Current loss: '),
	print(loss(model(inputs), outputs).numpy())

步骤四：构建训练循环。使用训练数据来更新模型变量，这样损失通过梯度下降减小。这里用基本的数学思维来实现，常规情况使用tf.train.Optimizer中已实现的接口即可。

	def train(model, inputs, outputs, learning_rate):
	  with tf.GradientTape() as t:
	    current_loss = loss(model(inputs), outputs)
	  dW, db = t.gradient(current_loss, [model.W, model.b])
	  model.W.assign_sub(learning_rate * dW)
	  model.b.assign_sub(learning_rate * db)

接着就是训练的循环

	model = Model()
	
	# Collect the history of W-values and b-values to plot later
	Ws, bs = [], []
	epochs = range(10)
	for epoch in epochs:
	  Ws.append(model.W.numpy())
	  bs.append(model.b.numpy())
	  current_loss = loss(model(inputs), outputs)
	
	  train(model, inputs, outputs, learning_rate=0.1)
	  print('Epoch %2d: W=%1.2f b=%1.2f, loss=%2.5f' %
	        (epoch, Ws[-1], bs[-1], current_loss))
	
	# Let's plot it all
	plt.plot(epochs, Ws, 'r',
	         epochs, bs, 'b')
	plt.plot([TRUE_W] * len(epochs), 'r--',
	         [TRUE_b] * len(epochs), 'b--')
	plt.legend(['W', 'b', 'true W', 'true_b'])
	plt.show()