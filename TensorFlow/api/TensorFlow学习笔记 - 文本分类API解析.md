## TensorFlow学习笔记 —— 文本分类API解析 [【上页】](https://tinyworker.github.io/TensorFlow/index)  ##
在高阶API-keras中，自带有一部分预处理好的数据集合，通过datasets包进行调用，本实例中使用的是imdb数据集。
其他的数据集包括：

- boston_housing
- cifar10
- cifar100
- fashion_mnist
- imdb
- mnist
- reuters

### Dataset - 数据集
其中，imdb数据集包含两个方法：

- load_data()，从文件中加载数据，其中包含多个参数，返回的是训练集与测试集的元组对象（x_train, y_train), (x_test, y_test）
  - path，相对路径及文件，默认是~/.keras/dataset，数据集为imdb.npz，若有需要可指定路径。
  - num_words，需要的关键字，词语会根据频率排序，取该参数设置的个数。
  - skip_top，会跳过参数个数的词语。
  - maxlen，序列需要的最大长度，超过部分就会被过滤掉。
  - seed，用于洗牌的随机种子
  - start_char，序列的起始会用指定字符标记。
  - oov\_char，被排除的词用此字符代替。该字符仅作用于训练集，且不再num_words范围内，训练集若没有但测试集有，会直接跳过。
  - index_from，索引实际词语。
  - **kwargs，向后兼容的参数。
- get_word_index()，获取词语映射字典，可将整数数组转换为字符串。

### Preprocessing - 预处理
本实例中仅使用了sequence.pad_sequence方法，用于统一填充数组的长度。该方法包含以下几个参数：

- sequences，包含序列的队列。 
- maxlen，序列最大长度。
- dtype，输出序列的类型。
- padding，有pre和post两种，意味着是前置填充还是后置填充。
- truncating，同上，当超过maxlen时做截取，决定前置截取还是后置截取。
- value，用于填充的值，只能是Float或String。

该方法是将一个包含整型序列的队列转换成2D的Numpy数组，其格式为（num_samples, maxlen)。maxlen可以是提供的maxlen参数，也可以是最长序列的长度。

当序列长度小于maxlen时，会通过value填充补齐。大于maxlen时，会被截取。默认是执行的前置填充。


### Modeling - 构建模型

**建模的第一步**，是创建层的线性栈，使用keras.Sequential()构造方法初始化。仅接受一个参数，即layers，层列表，通常为空。Sequential类是继承于model类。具有如下属性：

- input_spec
- layers
- metrics_names，为所有输出返回模型的展示标签
- run_eagerly，模型是否要按步骤运行，像代码一般，这样能逐层跟踪代码运行情况，默认是将模型编译成静态图来获取最优执行效率。
- sample_weights
- state_updates，返回所有层的状态，来分割训练的更新与状态的更新。
- stateful


**建模的第二步**，就是添加层，model使用add方法做层的添加。

**建模的第三步**，编译模型，model.compile，内置参数如下：

- **optimizer**，选择优化器实例，文末附录查看优化器类型。
- **loss**，损失函数名称，若模型有多个输出，是可以为每个输出都指定不同的损失函数的，通过列表或字典传参。最小化的损失值将是所有单个损失的总和。文末附录查看损失函数列表。
- **metrics**，用于评估的指标，比如准确率。同样可以用字典或列表来指明多输出对应多指标的情况。
- **loss_weights**，指定标量系数（Python浮点数）的可选列表或字典，用于加权不同模型输出的损失贡献。最后的值与loss逻辑一致。在此参数中，如果是列表，需要1对1的映射到输出，如果是张量，需要映射输出的名称。


### Training - 训练模型
在上面的操作完成后，就可以开始训练模型了，使用model.fit()开始对模型的训练及追踪。参数如下：

- x，输入数据集
- y，目标数据集
- batch_size，批量训练的数量，默认32
- epochs，训练周期，表示完整遍历一遍x，y的数据。
- verbose，0/1/2，详细模式，0是静默，1是进度条，2是每个周期一行。通常建议选择2。
- validation_data，用于每个周期结束时评估损失和模型指标的。该部分数据不会用作训练。

fit方法会在完成后返回一个history对象，该对象内部记录了每个周期训练的结果，可以用来生成图表以便于观察过程。

### 附录

优化器列表：

- Adadelta，算法实现，随机梯度下降原理，基于每个维度的自适应学习速率来解决两个缺点：
	1. 在训练过程中持续的学习速率下降。
	2. 需要手动选择的全局学习速率。
	
	有两个累积步骤：
	1. 梯度的累积平方。
	2. 更新的累积平方。
	 
	算法的初始化步骤详见[链接](https://tensorflow.google.cn/api_docs/python/tf/keras/optimizers/Adadelta#initialization)，具体分析后续出专题文章。
- Adagrad，参数指定的学习速率，根据参数在训练期间的更新频率来调整，接收到的更新越多，更新幅度越小。

	算法的初始化步骤详见[链接](https://tensorflow.google.cn/api_docs/python/tf/keras/optimizers/Adagrad#initialization)，具体分析后续出专题文章。
- Adam，随机梯度下降原理，基于一阶和二阶矩阵的自适应预估。算法特点有：计算效率，有少量内存要求，对对角梯度重新缩放的不变性，以及对大量数据/参数问题的良好适应性。
- Adamax，基于无限规范的Adam算法变体。有时候作为Adam的监督，特别是在嵌入式模型中。
- Ftrl，详见论文，提供在线L2及缩减类L2算法支持。
- Nadam，和Adam类似，Adam是动量RMSprop，Nadam是动量Nesterov。
- RMSprop，详见论文。
- SGD，随机梯度下降和动量优化器。
- Optimizer，是优化器的基础类，所有算法类优化器均基于此类扩展。


损失函数列表：
数量过多，优先级滞后，后续补充。

### 实例完整代码
	import tensorflow as tf
	from tensorflow import keras
	
	import numpy as np
	
	imdb = keras.datasets.imdb
	(train_data, train_labels),(test_data, test_labels) = imdb.load_data(num_words=10000)
	
	#a dictionary mapping words to an integer index
	word_index = imdb.get_word_index();
	
	#the first indices are reserved
	word_index = {k:(v+3) for k,v in word_index.items()}
	word_index["<PAD>"] = 0
	word_index["<START>"] = 1
	word_index["<UNK>"] = 2 #unknown
	word_index["<UNUSED>"] = 3
	
	reverse_word_index = dict([(value,key) for (key,value) in word_index.items()])
	def decode_review(text):
	    return ' '.join([reverse_word_index.get(i,'?') for i in text])
	
	train_data = keras.preprocessing.sequence.pad_sequences(train_data,
	                                                       value=word_index["<PAD>"],
	                                                       padding='post',
	                                                       maxlen=256)
	test_data = keras.preprocessing.sequence.pad_sequences(test_data,
	                                                       value=word_index["<PAD>"],
	                                                       padding='post',
	                                                       maxlen=256)
	
	vocab_size = 10000
	
	model = keras.Sequential()
	model.add(keras.layers.Embedding(vocab_size,16))
	model.add(keras.layers.GlobalAveragePooling1D())
	model.add(keras.layers.Dense(16,activation=tf.nn.relu))
	model.add(keras.layers.Dense(1, activation=tf.nn.sigmoid))
	model.summary()
	model.compile(optimizer=tf.train.AdamOptimizer(),
	             loss='binary_crossentropy',
	             metrics=['accuracy'])
	
	x_val = train_data[:10000]
	partial_x_train = train_data[10000:]
	
	y_val = train_labels[:10000]
	partial_y_train = train_labels[10000:]
	
	history = model.fit(partial_x_train,
	                   partial_y_train,
	                   epochs=40,
	                   batch_size=512,
	                   validation_data=(x_val,y_val),
	                   verbose=1)

	results = model.evaluate(test_data,test_labels)
	print(results)

	history_dict = history.history
	history_dict.keys()

	import matplotlib.pyplot as plt
	
	acc = history.history['acc']
	val_acc = history.history['val_acc']
	loss = history.history['loss']
	val_loss = history.history['val_loss']
	
	epochs = range(1, len(acc) + 1)
	
	plt.plot(epochs, loss, 'bo', label='Training loss')
	plt.plot(epochs, val_loss, 'b', label='Validation loss')
	plt.title('Training and validation loss')
	plt.xlabel('Epochs')
	plt.ylabel('Loss')
	plt.legend()
	
	plt.show()
	
	plt.clf()
	acc_values = history_dict['acc']
	val_acc_values = history_dict['val_acc']
	
	plt.plot(epochs, acc, 'bo', label='Training acc')
	plt.plot(epochs, val_acc, 'b', label='Validation acc')
	plt.title('Training and validation accuracy')
	plt.xlabel('Epochs')
	plt.ylabel('Accuracy')
	plt.legend()
	
	plt.show()
### 运行结果

