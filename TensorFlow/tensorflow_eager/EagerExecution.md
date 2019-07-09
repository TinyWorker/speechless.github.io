## Eager Execution - 基本概念与组件

### Eager的使用

eager是TensorFlow的一个交互式前端组件，使用方式如下：

	from __future__ import absolute_import, division, print_function
	
	import tensorflow as tf
	
	tf.enable_eager_execution()

但在使用Eager前，还有一些组件和概念需要了解。

### 张量（Tensors）

张量就是一个多维数组，与Numpy的ndarray类似，张量拥有数据类型和形状。另外，张量可以在加速器（如GPU）内存中存在。TensorFlow提供了丰富的操作库来创建和使用张量。而这些操作会自动转换成Python内置类型。比如：
	
	print(tf.add(1, 2))
	print(tf.add([1, 2], [3, 4]))
	print(tf.square(5))
	print(tf.reduce_sum([1, 2, 3]))
	print(tf.encode_base64("hello world"))
	
	# Operator overloading is also supported
	print(tf.square(2) + tf.square(3))

	x = tf.matmul([[1]], [[2, 3]])
	print(x.shape)
	print(x.dtype)

张量与Numpy数组的区别主要在于：

- 张量能在加速器内存中备份。
- 张量是固定板不变的。

### Numpy的兼容性
在TensorFlow的操作中，有如下转换：
 
- tf的操作会自动将ndarrays转换为张量。
- numpy的操作会自动将张量转换为ndarrays。

### GPU加速
多数的tf操作可以通过GPU运算进行加速。在没有任何声明的情况下，tf会自主决定是用GPU还是CPU进行操作。
	
	x = tf.random_uniform([3, 3])
	
	print("Is there a GPU available: "),
	print(tf.test.is_gpu_available())
	
	print("Is the Tensor on GPU #0:  "),
	print(x.device.endswith('GPU:0'))

### 设备名（Device Name）
Tensor.device属性，为托管张量的设备提供了一个完全限定字符串名称。这个名字编码了许多细节，比如正在执行此程序的主机的网络地址标识符及该主机的设备。这是分布式执行的必要要求。这个字符串结束于"GPU:<N>"，N代表存在张量的GPU序号。

### 显式设备放置
这是指单独的操作如何分配到设备中执行的。当没有显式声明时，tf会自主决定选择某个设备执行操作，并在必要时复制张量到设备。但是，tf的操作也可以显式的放置到指定设备。

	import time
	
	def time_matmul(x):
	  start = time.time()
	  for loop in range(10):
	    tf.matmul(x, x)
	
	  result = time.time()-start
	
	  print("10 loops: {:0.2f}ms".format(1000*result))
	
	
	# Force execution on CPU
	print("On CPU:")
	with tf.device("CPU:0"):
	  x = tf.random_uniform([1000, 1000])
	  assert x.device.endswith("CPU:0")
	  time_matmul(x)
	
	# Force execution on GPU #0 if available
	if tf.test.is_gpu_available():
	  with tf.device("GPU:0"): # Or GPU:1 for the 2nd GPU, GPU:2 for the 3rd etc.
	    x = tf.random_uniform([1000, 1000])
	    assert x.device.endswith("GPU:0")
	    time_matmul(x)

### 数据集（Datasets）
这节展示了使用Dataset API来构建管道为模型提供数据。

创建数据，使用了Dataset.from_tensor_slices或data对象的TextLineDataset()或TFRecordDataset()：

	ds_tensors = tf.data.Dataset.from_tensor_slices([1, 2, 3, 4, 5, 6])
	
	# Create a CSV file
	import tempfile
	_, filename = tempfile.mkstemp()
	
	with open(filename, 'w') as f:
	  f.write("""Line 1
	Line 2
	Line 3
	  """)
	
	ds_file = tf.data.TextLineDataset(filename)

转换数据，使用如map，batch，shuffle等将记录转换为dataset：

	ds_tensors = ds_tensors.map(tf.square).shuffle(2).batch(2)
	
	ds_file = ds_file.batch(2)

遍历数据，当eager激活时，dataset支持遍历。

	print('Elements of ds_tensors:')
	for x in ds_tensors:
	  print(x)
	
	print('\nElements in ds_file:')
	for x in ds_file:
	  print(x)