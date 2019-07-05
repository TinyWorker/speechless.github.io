## TensorFlow学习笔记 —— 文本分类实例 ##
本记针对TensorFlow-Tutorials-Text classification的记录。

### 步骤一：准备数据集
我们会将文本形式的影评内容分为“正面”和“负面”，即二元分类。使用的集合来自于IMDB数据集，包含来自互联网的5w条影评文本，并将其拆分成1:1的训练集与测试集。

使用的是高阶API-keras，在TensorFlow中用于构建和训练模型。
	import tensorflow as tf
	from tensorflow import keras
	
	import numpy as np
	
	print(tf.__version__)#引入依赖项，查看当前TensorFlow版本。
	
	#直接获取keras中预设的数据集，这里无需我们对原始数据做处理，该数据集已经过特征工程转换。
	imdb = keras.datasets.imdb
	#创建训练集与测试集的样本标签数据，num_words=10000是过滤频次在前10000位的字词。
	(train_data, train_labels),(test_data, test_labels) = imdb.load_data(num_words=10000)
	


### 步骤二：查看数据
得到经过特征提取的数据后，需要先了解下数据格式：每个样本都是一个整数数组，表示影评中的字词，标签都是0或1,0表示负面影评，1表示正面。

	print(train_data[0])#查看样本数组格式
	
	len(train_data[0]), len(train_data[1])#查看样本长度

根据输出，可以看到文本已转换为整数，每个整数都表示词典的一个特定字词，同时长度也有所不同，鉴于神经网络的输入必须拥有相同长度，所以需要对长度进行处理。

### 步骤三：整数还原为字词 
首先将数组整数通过字典映射来还原，如下代码：
	
	#a dictionary mapping words to an integer index
	word_index = imdb.get_word_index();

	#the first indices are reserved
	word_index = {k:(v+3) for k,v in word_index.items()}
	word_index["<PAD>"] = 0 #空格
	word_index["<START>"] = 1 #开始标志
	word_index["<UNK>"] = 2 #unknown，未知词语
	word_index["<UNUSED>"] = 3 #不使用
	
	reverse_word_index = dict([(value,key) for (key,value) in word_index.items()])
	def decode_review(text):
	    return ' '.join([reverse_word_index.get(i,'?') for i in text])

### 步骤四：填充数据
数组必须转换为张量后，才能发送到神经网络中。有两种方法能够实现这种转换： 

- 对数组进行独热编码，将其转换为0和1构成的向量。比如将序列转换为N维向量，除序列对应索引为1，其余均为0，并作为网络第一层（密集层）。但这种方法会占据大量内存，需要大小为num_words * num_reviews的矩阵。
- 对数组进行填充，使其具有相同的长度，创建形状为max_length*num_reviews的整数张量，并使用能处理该形状的嵌入层作为网络中的第一层。
代码如下：

		train_data = keras.preprocessing.sequence.pad_sequences(train_data,
		                                                        value=word_index["<PAD>"],
		                                                        padding='post',
		                                                        maxlen=256)		

		test_data = keras.preprocessing.sequence.pad_sequences(test_data,
	                                                       value=word_index["<PAD>"],
	                                                       padding='post',
	                                                       maxlen=256)
 


完成填充后，每个整数组长度均变为256.

### 步骤五：构建模型

构建模型时，通常要做出两个主要决策：

- 要使用多少层？
- 每个层要用多少隐藏单元？

由于本实例中，数据是由词语+索引的数组构成，要预测的标签是0或1，代码构建如下：

	# input shape is the vocabulary count used for the movie reviews (10,000 words)
	vocab_size = 10000
	#四层构建，第一层为嵌入层，第二层，第三次为密集层，使用ReLU，第四层隐藏层（密集层），使用sigmoid算法做分类
	model = keras.Sequential()
	model.add(keras.layers.Embedding(vocab_size, 16))
	model.add(keras.layers.GlobalAveragePooling1D())
	model.add(keras.layers.Dense(16, activation=tf.nn.relu))
	model.add(keras.layers.Dense(1, activation=tf.nn.sigmoid))
	model.summary()


### 步骤六：损失函数与优化器
模型在训练时需要一个损失函数和一个优化器。因为是二元分类问题，且模型是输出一个概率，因此使用binary_corssentropy做损失函数。代码如下：

	model.compile(optimizer=tf.train.AdamOptimizer(),
              loss='binary_crossentropy',
              metrics=['accuracy'])

### 步骤七：创建验证集
训练时，为了检查新数据的准确率，需要从原始数据集中划分出1w个样本做验证集，之所以不使用测试集，因为我们的目的是利用训练数据来开发和调整模型，测试集只会使用一次。

	x_val = train_data[:10000]
	partial_x_train = train_data[10000:]
	
	y_val = train_labels[:10000]
	partial_y_train = train_labels[10000:]

### 步骤八：训练模型/评估模型
实例中用每批次512训练40个周期，即对所有样本进行40次迭代，训练过程中会监控模型的损失和准确率，在完成训练后，可以查看模型的表现，依据损失与准确率来检查。代码如下：

	history = model.fit(partial_x_train,
	                    partial_y_train,
	                    epochs=40,
	                    batch_size=512,
	                    validation_data=(x_val, y_val),
	                    verbose=1)
	
	results = model.evaluate(test_data, test_labels)
	
	print(results)

### 步骤九：创建图来查看变化曲线
先根据模型训练得到的history对象获得数据，在根据结果生成趋势图，代码如下：

	history_dict = history.history
	history_dict.keys()
	
	import matplotlib.pyplot as plt
	
	acc = history.history['acc']
	val_acc = history.history['val_acc']
	loss = history.history['loss']
	val_loss = history.history['val_loss']
	
	epochs = range(1, len(acc) + 1)
	
	# "bo" is for "blue dot"
	plt.plot(epochs, loss, 'bo', label='Training loss')
	# b is for "solid blue line"
	plt.plot(epochs, val_loss, 'b', label='Validation loss')
	plt.title('Training and validation loss')
	plt.xlabel('Epochs')
	plt.ylabel('Loss')
	plt.legend()
	
	plt.show()
	
	plt.clf()   # clear figure
	acc_values = history_dict['acc']
	val_acc_values = history_dict['val_acc']
	
	plt.plot(epochs, acc, 'bo', label='Training acc')
	plt.plot(epochs, val_acc, 'b', label='Validation acc')
	plt.title('Training and validation accuracy')
	plt.xlabel('Epochs')
	plt.ylabel('Accuracy')
	plt.legend()
	
	plt.show()

### 总结
以下是该实例中涉及到的部分API及算法解释。
