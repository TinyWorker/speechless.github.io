## 保存与恢复

模型的训练进度可以在期间和之后保存，意味着可以在上次暂停的地方继续训练，这就避免了一次性训练时间过长的问题。同时，保存也意味着可以分享。

### 训练期间保存检查点
keras提供了用于保存模型训练节点的功能，使用callbacks.ModelCheckpoint来执行回调。

回调函数样例如下：

	checkpoint_path = "training_1/cp.ckpt"
	checkpoint_dir = os.path.dirname(checkpoint_path)
	
	# Create checkpoint callback
	cp_callback = tf.keras.callbacks.ModelCheckpoint(checkpoint_path,
	                                                 save_weights_only=True,
	                                                 verbose=1)
	
	model = create_model()
	
	model.fit(train_images, train_labels,  epochs = 10,
	          validation_data = (test_images,test_labels),
	          callbacks = [cp_callback])  # pass callback to training

可以看出，在使用检查点时，需要配置保存文件路径，是否仅保存权重，以及存储模式，最后在fit方法中配置callbacks参数即可。

如果想要加载最新检查点，如下：

	latest = tf.train.latest_checkpoint(checkpoint_dir)
	model = create_model()
	model.load_weights(latest)
	loss, acc = model.evaluate(test_images, test_labels)
	print("Restored model, accuracy: {:5.2f}%".format(100*acc))

如果想手动的保存，也是可以的，调用Model.save_weights方法：

    # Save the weights
    model.save_weights('./checkpoints/my_checkpoint')
    
    # Restore the weights
    model = create_model()
    model.load_weights('./checkpoints/my_checkpoint')
    
    loss,acc = model.evaluate(test_images, test_labels)
    print("Restored model, accuracy: {:5.2f}%".format(100*acc))

### 保存整个模型

模型可以完整的保存，包含其权重值，模型配置甚至优化器配置，这样模型的训练可以无需访问原始代码。keras对模型是生成一个HDF5标准的格式文件，可以视为一个二进制blob。

	model = create_model()
	
	model.fit(train_images, train_labels, epochs=5)
	
	# Save entire model to a HDF5 file
	model.save('my_model.h5')

	new_model = keras.models.load_model('my_model.h5')
	new_model.summary()

目前keras无法保存TensorFlow优化器（来自tf.train），在用这一类优化器时，需要加载模型后对其进行重新编译，使优化器状态变松散。


### 文件信息
keras的保存是将每次训练后的权重信息存储在检查点格式文件中，检查点包括：模型权重的一个或多个分片，指示哪些权重存储在哪些分片中的索引文件。


### 后续进阶

上述是tf.keras保存和加载模型的基本操作。

- tf.keras的指南中详细介绍了如何使用该api保存和加载模型。
- Eager Execution也提供了保存模型操作。
- 低阶API也具备此功能