## TensorFlow学习笔记 - 回归，预测燃料效率 [【上页】](https://tinyworker.github.io/TensorFlow/index)  ##

在一个回归问题中，我们需要预测一个连续值的输出，比如一个价格或概率。与分类问题对比，我们需要选择一个分类。

### 步骤

由于框架本身的特点，基础步骤与文本分类问题基本一致，都是：

- 准备数据
- 构建模型
- 配置优化器和损失函数
- 准备训练/验证/测试集
- 评估损失
- 生成图进行过程查看

### 准备数据 - pandas上线
实例的数据来源是文本文件，通过pandas的数据读取及自动封装来获得一个标准的数据集，pandas对于文档格式的适应见参数。

	column_names = ['MPG','Cylinders','Displacement','Horsepower','Weight',
	                'Acceleration', 'Model Year', 'Origin']
	raw_dataset = pd.read_csv(dataset_path, names=column_names,
	                      na_values = "?", comment='\t',
	                      sep=" ", skipinitialspace=True)
	
	dataset = raw_dataset.copy()
	dataset.tail()

得到原始数据后，通常都会做数据的清理，因为原始数据通常包含很多无用信息或零项，所以，优先去空记录。

	dataset.isna().sum() #统计空值列
	dataset = dataset.dropna() #去掉空值列

同时，有列属于分类值，不是数字型，需要转换为独热编码。

	origin = dataset.pop('Origin')
	dataset['USA'] = (origin == 1)*1.0
	dataset['Europe'] = (origin == 2)*1.0
	dataset['Japan'] = (origin == 3)*1.0
	dataset.tail()

处理完成后，用seaborn查看数据各列的分布情况。

	sns.pairplot(train_dataset[["MPG", "Cylinders", "Displacement", "Weight"]], diag_kind="kde")

使用Pandas进行常规统计。

	train_stats = train_dataset.describe()
	train_stats.pop("MPG")
	train_stats = train_stats.transpose()
	train_stats

将标签从特征中分离，最后标准化数据

	train_labels = train_dataset.pop('MPG')
	test_labels = test_dataset.pop('MPG')
	def norm(x):
	  return (x - train_stats['mean']) / train_stats['std']
	normed_train_data = norm(train_dataset)
	normed_test_data = norm(test_dataset)


### 构建模型
由于构建模型的过程基本一致，不单独聊过程了，说明下内部的一些不同。

首先模型构建时并没有太多层，设置的是3个密集层，使用RMSprop优化器，以及使用平均绝对值误差（MAE），平均平方误差（MSE）做评估矩阵。

	def build_model():
	  model = keras.Sequential([
	    layers.Dense(64, activation=tf.nn.relu, input_shape=[len(train_dataset.keys())]),
	    layers.Dense(64, activation=tf.nn.relu),
	    layers.Dense(1)
	  ])
	
	  optimizer = tf.keras.optimizers.RMSprop(0.001)
	
	  model.compile(loss='mean_squared_error',
	                optimizer=optimizer,
	                metrics=['mean_absolute_error', 'mean_squared_error'])
	  return model

### 训练与查看过程
该实例由于训练集特别少（数据集不超过500条），所以没有划分验证集，仅设置了80%的训练集，训练时添加了一个callbacks的配置，是用于每个训练周期结束后进行测试，如果一个周期测试结果并没有任何提高，则会自动停止训练过程。

	early_stop = keras.callbacks.EarlyStopping(monitor='val_loss', patience=10)
	
	history = model.fit(normed_train_data, train_labels, epochs=EPOCHS,
	                    validation_split = 0.2, verbose=0, callbacks=[early_stop, PrintDot()])

### 总结

这次实例介绍了回归问题的几个处理技术（不同于分类问题）：

- 平均平方误差（MSE），是常见的用于回归问题的损失函数。
- 平均绝对误差（MAE），同样的，常见用于回归问题的评估矩阵。
- 当数字类型输入有不同范围的值时，每个特性应该单独划分到同一个范围。
- 如果没有太多训练数据，一个技巧是用一个小型的带很少的隐藏层网络来避免过拟合。
- 提前停止是一个用来避免过拟合的有用技巧。


### 完整代码

	from __future__ import absolute_import, division, print_function, unicode_literals
	
	import pathlib
	
	import matplotlib.pyplot as plt
	import pandas as pd
	import seaborn as sns
	import tensorflow as tf
	from tensorflow import keras
	from tensorflow.keras import layers
	
	print(tf.__version__)
	
	dataset_path = keras.utils.get_file("auto-mpg.data","http://archive.ics.uci.edu/ml/machine-learning-databases/auto-mpg/auto-mpg.data")
	
	column_names = ['MPG','Cylinders','Displacement','Horsepower','Weight',
	              'Acceleration','Model Year','Origin']
	raw_dataset = pd.read_csv(dataset_path, names=column_names,
	                         na_values= "?", comment = '\t',
	                         sep = " ", skipinitialspace=True)
	dataset = raw_dataset.copy()
	
	dataset.tail()
	dataset.isna().sum()
	
	dataset = dataset.dropna()
	origin = dataset.pop('Origin')
	
	dataset['USA'] = (origin == 1) * 1.0
	dataset['Europe'] = (origin == 2) * 1.0
	dataset['Japan'] = (origin == 3) * 1.0
	dataset.tail()
	
	train_dataset = dataset.sample(frac=0.8, random_state=0)
	test_dataset = dataset.drop(train_dataset.index)
	
	sns.pairplot(train_dataset[["MPG","Cylinders","Displacement","Weight"]], diag_kind="kde")
	
	train_stats = train_dataset.describe()
	train_stats.pop("MPG")
	train_stats = train_stats.transpose()
	train_stats
	
	train_labels = train_dataset.pop('MPG')
	test_labels = test_dataset.pop('MPG')
	
	def norm(x):
	    return (x - train_stats['mean']) / train_stats['std']
	
	normed_train_data = norm(train_dataset)
	normed_test_data = norm(test_dataset)
	
	def build_model():
	    model = keras.Sequential([
	        layers.Dense(64, activation=tf.nn.relu, input_shape=[len(train_dataset.keys())]),
	        layers.Dense(64, activation=tf.nn.relu),
	        layers.Dense(1)
	    ])
	    
	    optimizer = tf.keras.optimizers.RMSprop(0.001)
	    
	    model.compile(loss='mean_squared_error',
	                  optimizer=optimizer,
	                 metrics=['mean_absolute_error','mean_squared_error'])
	    return model
	
	model = build_model()
	model.summary()
	
	example_batch = normed_train_data[:10]
	example_result = model.predict(example_batch)
	example_result
	
	class PrintDot(keras.callbacks.Callback):
	    def on_epoch_end(self, epoch, logs):
	        if epoch % 100 == 0 : print('')
	        print('.', end='')
	        
	EPOCHS = 1000
	
	history = model.fit(normed_train_data, train_labels,
	                   epochs=EPOCHS, validation_split=0.2, verbose=0,
	                   callbacks=[PrintDot()])
	
	hist = pd.DataFrame(history.history)
	hist['epoch'] = history.epoch
	hist.tail()
	
	def plot_history(history):
	    hist = pd.DataFrame(history.history)
	    hist['epoch'] = history.epoch
	    
	    plt.figure()
	    plt.xlabel('Epoch')
	    plt.ylabel('Mean Abs Error [MPG]')
	    plt.plot(hist['epoch'], hist['mean_absolute_error'],label='Train Error')
	    plt.plot(hist['epoch'], hist['val_mean_absolute_error'], label = 'Val Error')
	    plt.ylim([0,5])
	    plt.legend()
	    
	    plt.figure()
	    plt.xlabel('Epoch')
	    plt.ylabel('Mean Square Error [$MPG^2$]')
	    plt.plot(hist['epoch'], hist['mean_squared_error'],label='Train Error')
	    plt.plot(hist['epoch'], hist['val_mean_squared_error'],label='Val Error')
	    plt.ylim([0,20])
	    plt.legend()
	    plt.show()
	    
	plot_history(history)
	
	model = build_model()
	early_stop = keras.callbacks.EarlyStopping(monitor='val_loss', patience=10)
	history = model.fit(normed_train_data, train_labels, epochs=EPOCHS,
	                   validation_split = 0.2, verbose=0,
	                    callbacks=[early_stop, PrintDot()])
	plot_history(history)
	
	loss, mae, mse = model.evaluate(normed_test_data, test_labels, verbose=0)
	print("Testing set Mean Abs Error: {:5.2f} MPG".format(mae))
	
	test_predictions = model.predict(normed_test_data).flatten()
	
	plt.scatter(test_labels, test_predictions)
	plt.xlabel('True Values [MPG]')
	plt.ylabel('Predictions [MPG]')
	plt.axis('equal')
	plt.axis('square')
	plt.xlim([0,plt.xlim()[1]])
	plt.ylim([0,plt.ylim()[1]])
	_ = plt.plot([-100, 100], [-100, 100])
	
	error = test_predictions - test_labels
	plt.hist(error, bins = 25)
	plt.xlabel("Prediction Error [MPG]")
	_ = plt.ylabel("Count")