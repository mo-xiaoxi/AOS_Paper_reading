# DeepXplore: Automated Whitebox Testing of Deep Learning Systems

## 1. 简介

深度学习（DL）系统在当下应用的越来越广泛，而现有的DL测试很大程度上依赖于手工标记数据，因此往往无法完全测试到所有情况，导致DL系统在实际运行中会在某些边缘情况可能存在误判。而这种误判，尤其是在类似自动驾驶领域，会导致十分严重的后果。针对这个问题，研究者设计与实现了DeepXplore，一个白盒DL测试审计框架。该框架引入神经元覆盖，系统地测量由测试输入执行的DL系统的部分，并利用多个具有类似功能的DL系统作为交叉引用预言，以避免手动检查。此外，DeepXplore生成的测试输入也可用于重新训练相应的DL模型，以将模型的精度提高3%

## 2. 相关介绍

##### 自动化大规模测试DL系统的挑战：

- 1. 如何生成能够触发不同逻辑行为的测试数据
- 1. 如何（在无标记情况下）识别DL系统的错误行为。

##### 传统测试的限制

- 1. 昂贵的标签成本：对大量DNN测试集打标签是一个很耗费人力成本的工作，而且我们也无法保证人力模式下打出来的标签是否正确。
- 1. 低测试覆盖率

##### 文章主要贡献

- 1. 引入神经元覆盖技术作为DL系统的第一个白盒测试指标，可以估计一组测试输入所探测的DL逻辑数量
- 1. 证明寻找DL系统之间的大量行为差异并同时最大化神经元覆盖旅的问题可以被表述为联合优化问题，并提出了一种基于梯度的算法来有效解决这个问题
- 1. 研究者们将所有这些技术都实施为DeepXplore（一个白盒DL测试框架）
- 1. DeepXplore生成的测试集也可以用于重新培训相应的DL系统，并能将分类准确度提高3%



## 3. 相关实现

- 在TensorFlow和Keras深度学习框架上进行的实现

- DeepXplore主要解决联合优化问题，保证使差异行为和神经元覆盖率最大化。

- 对于一个软件系统测试来说，测试的完备性是一个重要的指标。而对于一个DL系统来说，其具体逻辑并不完全体现在代码的分支流程图上，而是其具体神经元相关（训练数据定义训练后代码执行逻辑）。因此，论文提出neuron coverage的指标来度量测试完备程度。对于DL系统来说，最后的结果实际上受到神经网络中每个神经元的影响。简单来看，神经元输出越大则对最终结果影响越大，因此作者提出设置一个threshold，如果神经元输出超过这个threshold则算是激活。因此，测试用例生成的目的则是最大化被激活的neuron总量。

- 工作示意图:

  ![4](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/4.png?raw=true)

  DeepXplore基于差分测试运行。其从一组测试输入开始探索模型空间，它将未标记的测试输入作为种子，并生成大量神经元的新测试(即将其激活为高于可定制阈值的值），同时使测试过的 DNN生成不同的行为。

  在生成新测试的同时，DeepXplore试图最大限度地提高神经元的覆盖率和尽可能多地揭示出让 DNN产生行为区别的测试。这两个目标都是通过完整的测试来发现错误的极端方案所必需的。在这个过程中，也可以对 DeepXplore进行自定义约束，以确保生成的测试案例保持在给定范围内（与实际相符合，比如pdf文件要符合pdf规范）

- 示例：两个深度神经网络(NN1、NN2）用于将图像分类为汽车或者人脸。一开始，NN1和NN2都判定图片为汽车（0.95、0.98）

  ![5](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/5.png?raw=true)

  然后，DeepXplore会尝试修改图片，使达到令一个神经网络将图像归类为汽车，另一个归类为人脸，从而最大化差异，从而发现增大发现异常的可能性。

- 原理：这种测试主要是在探测输入空间介于不同DNN决策边界的部分。（参考4）

  ![6](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/6.png?raw=true))

  此时的神经网络已经被训练过了，所以权重是固定的。DeepXplore设计了一个算法，用梯度上升来解决联合	优化问题。这种生成测试用例的目标是诱导DL系统给出错误的预测。

  首先，我们将输出值作为变量，权重参数作为常量，计算出输出层和隐藏层中神经元输出的梯度。对于大多数的 DNN而言，相关的梯度都可以被有效的计算...接下来，我们通过迭代执行梯度来修改测试的输入，以最大化连接地联合优化问题的目标函数。

  联合优化函数如下：

  ![2](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/2.png?raw=true)

  - 在函数的第一项，我们试图在其他模型保持当前预测的情况下改变一个模型的输入方向，从而让其做出区别于其他模型的预测。用超参数 平衡了这两个因素的相对重要性。
  - 在函数的第二项，我们试图最大限度地激活一个不活跃的神经元，把它推到阈值 t 以上。
  - λ2 超参数平衡了这两个目标的相对重要性。

  完整算法流程：

  ![7](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/7.png?raw=true)

  

- 实验与测试：DeepXplore在所有测试的DNN中发现了很多很多错误，这些错误样本揭示了当前DNN模型的问题。与此同时，后续用这些样本继续训练时，可以提高原模型的正确率。

  研究者在3个DNN上进行了测试，分别是用了MNIST、Imagenet、Driving、Contagio/Virustotal、Drebin5个数据集，如下：

  ![1](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/1.png?raw=true)在基于图像的问题领域，探索了三种不同的约束：

  - 第一个约束模拟不同照明条件下的效果：DeepXplore可以使图像变暗或变亮，但不能改变内容。 在下图中，上面一行显示的是原始种子输入，下面一行显示的是 DeepXplore发现的差异诱导测试输入。箭头指向表示自动驾驶汽车决定转向的方式。

    ![8](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/8.png?raw=true)

- 第二个约束模拟意外或故意用单个小矩形遮挡住的镜头。

  ![9](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/9.png?raw=true)

- 第三个约束通过允许使用多个微小的黑色矩形进行遮挡来模拟透镜上多处被污垢覆盖后的影响。

  ![10](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/10.png?raw=true)

  DeepXplore在所有测试的DNN中发现了数千的错误行为。 下表总结了在用对应测试集中随机选取的 2000个种子输入对 DNN进行测试时，DeepXplore在每个测试的 DNN中发现的错误行为的数量。

  ![11](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/11.png?raw=true)

  实验测试表明，DeepXplore的神经元覆盖率比随机测试高34.4%，比对抗测试高33.2%，并且针对大多数模型，DeepXplore都可以在短时间内找到第一个能引起异常的输入。

  ![3](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/3.png?raw=true)

  此外，如果我们将 DNNs集合看作一个集成，并使用多数投票法，那么我们就为所生成的测试案例建立了一个自动标记系统。 通过使用这些新标记的训练样本就可以将神经网络的准确性提高 1-3％。

## 4. 实验

- [源码](https://github.com/peikexin9/deepxplore)

- 源码共有5个数据集的数据，及其生成脚本。

- 我们选ImageNet进行研究

  ```bash
  ubuntu@ubuntu-virtual-machine:~/AOS/deepxplore/ImageNet$ python gen_diff.py  -h
  Using TensorFlow backend.
  usage: gen_diff.py [-h] [-t {0,1,2}] [-sp START_POINT]
                     [-occl_size OCCLUSION_SIZE]
                     {light,occl,blackout} weight_diff weight_nc step seeds
                     grad_iterations threshold
  
  Main function for difference-inducing input generation in ImageNet dataset
  
  positional arguments:
    {light,occl,blackout}
                          realistic transformation type
    weight_diff           weight hyperparm to control differential behavior
    weight_nc             weight hyperparm to control neuron coverage
    step                  step size of gradient descent
    seeds                 number of seeds of input
    grad_iterations       number of iterations of gradient descent
    threshold             threshold for determining neuron activated
  
  optional arguments:
    -h, --help            show this help message and exit
    -t {0,1,2}, --target_model {0,1,2}
                          target model that we want it predicts differently
    -sp START_POINT, --start_point START_POINT
                          occlusion upper left corner coordinate
    -occl_size OCCLUSION_SIZE, --occlusion_size OCCLUSION_SIZE
                          occlusion size
  ```

  主要参数如上。

  下图作者给出了不同模式下的照片修改对比图，分别使用光照、遮挡矩形、多个遮挡物模式运行。

  ![12](https://github.com/m0xiaoxi/AOS_Paper_reading/blob/master/DeepXplore/pic/12.png?raw=true)

  图片模式运行：

  ```bash
  ubuntu@ubuntu-virtual-machine:~/AOS/deepxplore/ImageNet$ python gen_diff.py -t 0  light 0.3 0.2 1 2 3 0.1
  /usr/local/lib/python2.7/dist-packages/h5py/__init__.py:36: FutureWarning: Conversion of the second argument of issubdtype from `float` to `np.floating` is deprecated. In future, it will be treated as `np.float64 == np.dtype(float).type`.
    from ._conv import register_converters as _register_converters
  Using TensorFlow backend.
  2018-06-08 13:02:10.617459: I tensorflow/core/platform/cpu_feature_guard.cc:140] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 FMA
  2018-06-08 13:02:10.721914: W tensorflow/core/framework/allocator.cc:101] Allocation of 411041792 exceeds 10% of system memory.
  2018-06-08 13:02:11.041253: W tensorflow/core/framework/allocator.cc:101] Allocation of 411041792 exceeds 10% of system memory.
  2018-06-08 13:02:11.348274: W tensorflow/core/framework/allocator.cc:101] Allocation of 411041792 exceeds 10% of system memory.
  2018-06-08 13:02:11.697844: W tensorflow/core/framework/allocator.cc:101] Allocation of 411041792 exceeds 10% of system memory.
  Downloading data from https://github.com/fchollet/deep-learning-models/releases/download/v0.1/vgg19_weights_tf_dim_ordering_tf_kernels.h5
  574717952/574710816 [==============================] - 4353s 8us/step
  574726144/574710816 [==============================] - 4353s 8us/step
  2018-06-08 14:14:48.895939: W tensorflow/core/framework/allocator.cc:101] Allocation of 411041792 exceeds 10% of system memory.
  2018-06-08 14:14:49.212408: W tensorflow/core/framework/allocator.cc:101] Allocation of 411041792 exceeds 10% of system memory.
  2018-06-08 14:14:49.656251: W tensorflow/core/framework/allocator.cc:101] Allocation of 411041792 exceeds 10% of system memory.
  2018-06-08 14:14:49.988666: W tensorflow/core/framework/allocator.cc:101] Allocation of 411041792 exceeds 10% of system memory.
  Downloading data from https://github.com/fchollet/deep-learning-models/releases/download/v0.2/resnet50_weights_tf_dim_ordering_tf_kernels.h5
  102858752/102853048 [==============================] - 782s 8us/step
  102866944/102853048 [==============================] - 782s 8us/step
  Downloading data from https://s3.amazonaws.com/deep-learning-models/image-models/imagenet_class_index.json
  40960/35363 [==================================] - 1s 24us/step
  49152/35363 [=========================================] - 1s 20us/step
  input already causes different outputs: beer_glass, cocktail_shaker, packet
  covered neurons percentage 14888 neurons 0.075, 16168 neurons 0.085, 94059 neurons 0.740
  averaged covered neurons 0.576
  covered neurons percentage 14888 neurons 0.151, 16168 neurons 0.156, 94059 neurons 0.751
  averaged covered neurons 0.603
  ```

  

## 5. 相关思考

1. 该文章虽然在DL测试领域做出了一个很好的工作，但是距离真正完全实现DL系统测试还有很远的路要走。因为文章的测试模型只是在尝试从近似覆盖模型从训练数据中对比获得整个输入空间。而这种输入空间的获取，一个是相对较难，二是无法保证整个数据覆盖了所有输入，仍然可能存在一些未覆盖的点。

2. 这篇文章像我们展示了在深度学习领域的一些安全问题与考量，便于我们了解这个领域的前沿研究与学习思路

3. 文章讲述了DL系统下的一些测试挑战，那么我们能否测试一下机器学习方面的安全问题

4. 另外，针对DL领域的黑样本攻击是否也可以针对性地进行一些研究，如何特异性地自动构造黑样本攻击DL系统

     

## 5. 参考

1. [http://www.cs.columbia.edu/~junfeng/papers/deepxplore-sosp17.pdf](http://www.cs.columbia.edu/~junfeng/papers/deepxplore-sosp17.pdf)
2. [https://www.sigops.org/sosp/sosp17/slides/deepxplore-sosp17-slides.pptx](<https://www.sigops.org/sosp/sosp17/slides/deepxplore-sosp17-slides.pptx>)
3. [https://github.com/peikexin9/deepxplore](https://github.com/peikexin9/deepxplore)
4. [https://zhuanlan.zhihu.com/p/30457361](https://zhuanlan.zhihu.com/p/30457361)
5. https://zhuanlan.zhihu.com/p/32102821