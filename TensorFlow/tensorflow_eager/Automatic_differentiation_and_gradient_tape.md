## Eager Execution - 自动微分与梯度记录

###自动微分（Automatic differentation）
自动微分是一个优化模型的关键技术。在激活了Eager后使用对应API。

###渐变带（Gradient Tapes）
tf提供了tf.GradientTape API，用于计算相对于输入参数的计算梯度，将所有执行过的操作记录在context中，放置在一个tape上。接着TF使用该tape与每个记录操作相关联，用于计算反向模式计算的记录梯度。

	x = tf.ones((2, 2))
	
	with tf.GradientTape() as t:
	  t.watch(x)
	  y = tf.reduce_sum(x)
	  z = tf.multiply(y, y)
	
	# Derivative of z with respect to the original input tensor x
	dz_dx = t.gradient(z, x)
	for i in [0, 1]:
	  for j in [0, 1]:
	    assert dz_dx[i][j].numpy() == 8.0

	# Use the tape to compute the derivative of z with respect to the
	# intermediate value y.
	dz_dy = t.gradient(z, y)
	assert dz_dy.numpy() == 8.0

默认情况下，GradientTape持有的资源几乎在gradient()方法被调用时就释放了。若要计算相同计算的多个梯度，要创建persistent渐变带，该对象能允许多次调用gradient()方法，而资源的释放是在对象被回收的时候。

	x = tf.constant(3.0)
	with tf.GradientTape(persistent=True) as t:
	  t.watch(x)
	  y = x * x
	  z = y * y
	dz_dx = t.gradient(z, x)  # 108.0 (4*x^3 at x = 3)
	dy_dx = t.gradient(y, x)  # 6.0
	del t  # Drop the reference to the tape

因为tape记录已经执行的操作，所有Python的控制流自然会被处理：

	def f(x, y):
	  output = 1.0
	  for i in range(y):
	    if i > 1 and i < 5:
	      output = tf.multiply(output, x)
	  return output
	
	def grad(x, y):
	  with tf.GradientTape() as t:
	    t.watch(x)
	    out = f(x, y)
	  return t.gradient(out, x)
	
	x = tf.convert_to_tensor(2.0)
	
	assert grad(x, 6).numpy() == 12.0
	assert grad(x, 5).numpy() == 12.0
	assert grad(x, 4).numpy() == 4.0

### 高阶梯度(Higher-order gradients)

	x = tf.Variable(1.0)  # Create a Tensorflow variable initialized to 1.0
	
	with tf.GradientTape() as t:
	  with tf.GradientTape() as t2:
	    y = x * x * x
	  # Compute the gradient inside the 't' context manager
	  # which means the gradient computation is differentiable as well.
	  dy_dx = t2.gradient(y, x)

	d2y_dx2 = t.gradient(dy_dx, x)
	
	assert dy_dx.numpy() == 3.0
	assert d2y_dx2.numpy() == 6.0
