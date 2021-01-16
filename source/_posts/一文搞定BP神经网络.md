---
title: 一文搞定BP神经网络
date: 2021-01-16 17:20:51
categories:
- 计算机/人工智能
tags:
- 人工智能
- 神经网络
mathjax: true
---

&emsp;&emsp;**本文着重讲述经典BP神经网络的数学推导过程，并辅助一个小例子。本文不会介绍机器学习库(比如sklearn, TensorFlow等)的使用。**

>&emsp;&emsp;**本文难免会有叙述不合理的地方，希望读者可以在发送邮件。我会及时吸纳大家的意见，并在之后的chat里进行说明。**
>
>本文参考了一些资料，在此一并列出。
>
>- http://neuralnetworksanddeeplearning.com/chap2.html
>- https://www.deeplearning.ai/ coursera对应课程视频讲义
>- coursera华盛顿大学机器学习专项
>- 周志华《机器学习》
>- 李航《统计学习方法》
>- 张明淳《工程矩阵理论》

### 0. 什么是人工神经网络？
&emsp;&emsp;首先给出一个**经典的定义**：“神经网络是由具有适应性的简单单元组成的广泛并行互连网络，它的组织能够模拟生物神经系统对真实世界物体所作出的交互反应”[Kohonen, 1988]。

&emsp;&emsp;这种说法虽然很经典，但是对于初学者并不是很友好。比如我在刚开始学习的时候就把人工神经网络想象地很高端，以至于很长一段时间都不能理解为什么神经网络能够起作用。类比最小二乘法线性回归问题，在求解数据拟合直线的时候，我们是采用某种方法让预测值和实际值的“偏差”尽可能小。同理，BP神经网络也做了类似的事情——即通过让“偏差”尽可能小，使得神经网络模型尽可能好地拟合数据集。

### 1. 神经网络初探
#### 1.1 神经元模型
&emsp;&emsp;神经元模型是模拟生物神经元结构而被设计出来的。典型的神经元结构如下图1所示：

![图1 典型神经元结构 （图片来自维基百科）](https://gallery.angryberry.tech/blog/bp-network/pics/1.png)

&emsp;&emsp;神经元大致可以分为树突、突触、细胞体和轴突。树突为神经元的输入通道，其功能是将其它神经元的动作电位传递至细胞体。其它神经元的动作电位借由位于树突分支上的多个突触传递至树突上。神经细胞可以视为有两种状态的机器，激活时为“是”，不激活时为“否”。神经细胞的状态取决于从其他神经细胞接收到的信号量，以及突触的性质（抑制或加强）。当信号量超过某个阈值时，细胞体就会被激活，产生电脉冲。电脉冲沿着轴突并通过突触传递到其它神经元。（内容来自维基百科“感知机”）

&emsp;&emsp;同理，我们的神经元模型就是为了模拟上述过程，典型的神经元模型如下：

![图2 典型神经元模型结构 （摘自周志华老师《机器学习》第97页）](https://gallery.angryberry.tech/blog/bp-network/pics/2.png)

&emsp;&emsp;这个模型中，每个神经元都接受来自其它神经元的输入信号，每个信号都通过一个带有权重的连接传递，神经元把这些信号加起来得到一个总输入值，然后将总输入值与神经元的阈值进行对比（模拟阈值电位），然后通过一个“激活函数”处理得到最终的输出（模拟细胞的激活），这个输出又会作为之后神经元的输入一层一层传递下去。
#### 1.2 神经元激活函数
&emsp;&emsp;本文主要介绍2种激活函数，分别是$sigmoid$和$relu$函数，函数公式如下：
$$sigmoid(z)=\frac{1}{1+e^{-z}}$$

$$relu(z)= \begin{cases}
z & z>0\\\\ 
0&z\leq0
\end{cases}
$$
&emsp;&emsp;做函数图如下：

![$sigmoid(z)$](https://gallery.angryberry.tech/blog/bp-network/pics/3.png)

![$relu(z)$](https://gallery.angryberry.tech/blog/bp-network/pics/4.png)


>**补充说明**
>*【补充说明的内容建议在看完后文的反向传播部分之后再回来阅读，我只是为了文章结构的统一把这部分内容添加在了这里】*
>
>&emsp;&emsp;引入激活函数的**目的**是在模型中引入非线性。如果没有激活函数，那么无论你的神经网络有多少层，最终都是一个线性映射，单纯的线性映射无法解决线性不可分问题。引入非线性可以让模型解决线性不可分问题。
>
>&emsp;&emsp;一般来说，在神经网络的中间层更加建议使用 $relu$ 函数，两个原因：
>
>- $relu$函数计算简单，可以加快模型速度；
>- 由于反向传播过程中需要计算偏导数，通过求导可以得到$sigmoid$函数导数的最大值为0.25，如果使用$sigmoid$函数的话，每一层的反向传播都会使梯度最少变为原来的四分之一，当层数比较多的时候可能会造成梯度消失，从而模型无法收敛。
#### 1.3 神经网络结构
&emsp;&emsp;我们使用如下神经网络结构来进行介绍，第0层是输入层（3个神经元）， 第1层是隐含层（2个神经元），第2层是输出层：

![【图4 神经网络结构（手绘）】](https://gallery.angryberry.tech/blog/bp-network/pics/5.png)


 &emsp;&emsp;我们使用以下**符号约定**，$w_{jk}^{[l]}$表示从网络第$(l-1)^{th}$中$k^{th}$个神经元指向第$l^{th}$中第$j^{th}$个神经元的连接权重，比如上图中$w^{[1]}_{21}$即从第0层第1个神经元指向第1层第2个神经元的权重。同理，我们使用$b^{[l]}_j$来表示第$l^{th}$层中第$j^{th}$神经元的偏差，用$z^{[l]}_j$来表示第$l^{th}$层中第$j^{th}$神经元的线性结果,用$a^{[l]}_j$来表示第$l^{th}$层中第$j^{th}$神经元的激活函数输出。

 &emsp;&emsp;激活函数使用符号$\sigma$表示，因此，第$l^{th}$层中第$j^{th}$神经元的激活为：
 $$a^{[l]}_j=\sigma(\sum_kw^{[l]}_{jk}a^{[l-1]}_k+b^{[l]}_j)$$

 &emsp;&emsp;现在，我们使用矩阵形式重写这个公式：

 &emsp;&emsp;定义$w^{[l]}$表示权重矩阵，它的每一个元素表示一个权重，即每一行都是连接第$l$层的权重，用上图举个例子就是：

 $$w^{[1]}=\left[ \begin{array}{cc}  w_{11}^{[1]} & w_{12}^{[1]} & w_{13}^{[1]} \\\\ w_{21}^{[1]}& w_{22}^{[1]} & w_{23}^{[1]}\end{array}\right]$$
&emsp;&emsp;同理，

$$b^{\[1\]}=\left[ \begin{array}{cc} b^{[1]}_1 \\\\ b^{[1]}_2 \end{array}\right]$$

$$ z^{[1]}=
\left[ \begin{array}{cc}  w_{11}^{[1]} & w_{12}^{[1]} & w_{13}^{[1]} \\\\ 
w_{21}^{[1]}& w_{22}^{[1]} & w_{23}^{[1]} \end{array} \right] \cdot \left[ \begin{array}{cc} a^{[0]}_1 \\\\ a^{[0]}_2 \\\\ a^{[0]}_3 \end{array} \right] + \left[ \begin{array}{cc} b^{[1]}_1 \\\\ b^{[1]}_2 \end{array} \right] \\\\ 
= \left[ \begin{array}{cc} w_{11}^{[1]} a_1^{[0]} + w_{12}^{[1]}a_2^{[0]} + w_{13}^{[1]}a^{[0]}_3+b_1^{[1]} \\\\ w^{[1]}_{21}a_1^{[0]} + w_{22}^{[1]}a_2^{[0]} + w_{23}^{[1]}a^{[0]}_3+b^{[1]}_2\end{array}\right]$$

&emsp;&emsp;更一般地，我们可以把前向传播过程表示：

**$$a^{[l]}=\sigma(w^{[l]}a^{[l-1]}+b^{[l]})$$**

&emsp;&emsp;到这里，我们已经讲完了前向传播的过程，**值得注意的是**，这里我们只有一个输入样本，对于多个样本同时输入的情况是一样的，只不过我们的输入向量不再是一列，而是m列，每一个都表示一个输入样本。

&emsp;&emsp;**多样本输入**情况下的表示为：

$$Z^{[l]}=w^{[l]}\cdot A^{[l-1]}+b^{[l]}$$

$$A^{[l]}=\sigma(Z^{[l]})$$

此时

$$A^{[l\-1]}=\left[ \begin{array}{cc} |& | & \ldots & | \\\\ a^{[l\-1]\(1\)} & a^{[l-1]\(2\)} & \ldots & a^{[l\-1]\(m\)} \\\\ | & | & \ldots &| \end{array} \right]$$

**每一列都表示一个样本，从样本1到m**

&emsp;&emsp;$w^{[l]}$的含义和原来完全一样，$Z^{[l]}$也会变成m列，每一列表示一个样本的计算结果。
>之后我们的叙述都是先讨论单个样本的情况，再扩展到多个样本同时计算。

### 2. 损失函数和代价函数
&emsp;&emsp;说实话，**损失函数（Loss Function）**和**代价函数（Cost Function）**并没有一个公认的区分标准，很多论文和教材似乎把二者当成了差不多的东西。

&emsp;&emsp;为了后面描述的方便，我们把二者稍微做一下区分（这里的区分仅仅对本文适用，对于其它的文章或教程需要根据上下文自行判断含义）：
>&emsp;&emsp;损失函数主要指的是对于**单个样本**的损失或误差；代价函数表示**多样本**同时输入模型的时候**总体**的误差——每个样本误差的和然后取平均值。
>
&emsp;&emsp;**举个例子**，如果我们把单个样本的损失函数定义为：
$$L(a,y)=-[y \cdot log(a)+(1-y)\cdot log(1-a)]$$
&emsp;&emsp;那么对于m个样本，代价函数则是：
$$C=-\frac{1}{m}\sum_{i=0}^m(y^{(i)}\cdot log(a^{(i)})+(1-y^{(i)})\cdot log(1-a^{(i)}))$$

### 3. 反向传播
&emsp;&emsp;反向传播的基本思想就是通过计算输出层与期望值之间的误差来调整网络参数，从而使得误差变小。

&emsp;&emsp;反向传播的思想很简单，然而人们认识到它的重要作用却经过了很长的时间。后向传播算法产生于1970年，但它的重要性一直到David Rumelhart，Geoffrey Hinton和Ronald Williams于1986年合著的论文发表才被重视。

&emsp;&emsp;事实上，人工神经网络的强大力量几乎就是建立在反向传播算法基础之上的。反向传播基于四个基础等式，数学是优美的，仅仅四个等式就可以概括神经网络的反向传播过程，然而理解这种优美可能需要付出一些脑力。事实上，**反向传播如此之难**，以至于相当一部分初学者很难进行独立推导。所以如果读者是初学者，希望读者可以耐心地研读本节。**对于初学者，我觉得拿出1-3个小时来学习本小节是比较合适的，当然，对于熟练掌握反向传播原理的读者，你可以在十几分钟甚至几分钟之内快速浏览本节的内容。**
#### 3.1 矩阵补充知识
&emsp;&emsp;对于大部分理工科的研究生，以及学习过矩阵论或者工程矩阵理论相关课程的读者来说，可以跳过本节。

&emsp;&emsp;本节主要面向只学习过本科线性代数课程或者已经忘记矩阵论有关知识的读者。

&emsp;&emsp;**总之，具备了本科线性代数知识的读者阅读这一小节应该不会有太大问题。本节主要在线性代数的基础上做一些扩展。**（不排除少数本科线性代数课程也涉及到这些内容，如果感觉讲的简单的话，勿喷）

##### **3.1.1 求梯度矩阵**

&emsp;&emsp;假设函数  $f:R^{m\times n}\rightarrow R$可以把输入矩阵（shape: $m\times n$）映射为一个实数。那么，函数$f$的梯度定义为：

$$\nabla_Af(A)=\left[ \begin{array}{cc} \frac{\partial f(A)}{\partial A_{11}}&\frac{\partial f(A)}{\partial A_{12}}&\ldots&\frac{\partial f(A)}{\partial A_{1n}} \\\\ \frac{\partial f(A)}{\partial A_{21}}&\frac{\partial f(A)}{\partial A_{22}}&\ldots&\frac{\partial f(A)}{\partial A_{2n}} \\\\\vdots &\vdots &\ddots&\vdots\\\\ \frac{\partial f(A)}{\partial A_{m1}}&\frac{\partial f(A)}{\partial A_{m2}}&\ldots&\frac{\partial f(A)}{\partial A_{mn}}\end{array}\right]$$
&emsp;&emsp;即$$(\nabla_Af(A))_{ij}=\frac{\partial f(A)}{\partial A_{ij}}$$

&emsp;&emsp;同理，一个输入是向量（**向量一般指列向量，本文在没有特殊声明的情况下默认指的是列向量**）的函数$f:R^{n\times 1}\rightarrow R$，则有：

$$\nabla_xf(x)=\left[ \begin{array}{cc}\frac{\partial f(x)}{\partial x_1}\\\\ \frac{\partial f(x)}{\partial x_2}\\\\ \vdots \\\\ \frac{\partial f(x)}{\partial x_n} \end{array}\right]$$

>&emsp;&emsp;**注意**：这里涉及到的梯度求解的前提是函数$f$ **返回的是一个实数**，**如果函数返回的是一个矩阵或者向量，那么我们是没有办法求梯度的**。比如，对函数$f(A)=\sum_{i=0}^m\sum_{j=0}^nA_{ij}^2$，由于返回一个实数,我们可以求解梯度矩阵。如果$f(x)=Ax (A\in R^{m\times n}, x\in R^{n\times 1})$，由于函数返回一个$m$行1列的向量，因此不能对$f$求梯度矩阵。

&emsp;&emsp;根据定义，很容易得到以下性质：
>&emsp;&emsp;$\nabla_x(f(x)+g(x))=\nabla_xf(x)+\nabla_xg(x)$
>&emsp;&emsp;$\nabla(tf(x))=t\nabla f(x), t\in R$

&emsp;&emsp;有了上述知识，我们来举个例子：
>&emsp;&emsp;定义函数$f:R^m\rightarrow R, f(z)=z^Tz$,那么很容易得到$\nabla_zf(z)=2z$，具体请读者自己证明。

##### **3.1.2 海塞矩阵**

&emsp;&emsp;定义一个输入为$n$维向量，输出为实数的函数$f:R^n\rightarrow R$，那么海塞矩阵（Hessian Matrix）定义为多元函数$f$的二阶偏导数构成的方阵：

$$\nabla^2_xf(x)=\left[ \begin{array}{cc} \frac{\partial^2f(x)}{\partial x_1^2}&\frac{\partial^2f(x)}{\partial x_1\partial x_2}&\ldots &\frac{\partial^2f(x)}{\partial x_1\partial x_n}\\\\ \frac{\partial^2f(x)}{\partial x_2\partial x_1}&\frac{\partial^2f(x)}{\partial x_2^2}&\ldots&\frac{\partial^2f(x)}{\partial x_2\partial x_n}\\\\ \vdots&\vdots&\ddots&\vdots\\\\\frac{\partial^2f(x)}{\partial x_n\partial x_1}&\frac{\partial^2f(x)}{\partial x_n\partial x_2}&\ldots&\frac{\partial^2f(x)}{\partial x_n^2}\end{array}\right]$$

&emsp;&emsp;由上式可以看出，海塞矩阵**总是对称阵**。
>&emsp;&emsp;**注意：**很多人把海塞矩阵看成$\nabla _xf(x)$的导数，这是不对的。只能说，海塞矩阵的**每个元素**都是函数$f$二阶偏导数。那么，有什么区别呢？
>
>&emsp;&emsp;首先，来看**正确的解释**。**海塞矩阵的每个元素是函数$f$的二阶偏导数。**拿$\frac{\partial^2f(x)}{\partial x_1\partial x_2}$举个例子，函数$f$对$x_1$求偏导得到的是一个实数，比如$\frac{\partial^2f(x)}{\partial x_1}=x_2^3x_1$，因此继续求偏导是有意义的,继续对$x_2$求偏导可以得到$3x_1x_2^2$。
>
>&emsp;&emsp;然后,来看一下**错误的理解**。把海塞矩阵看成$\nabla _xf(x)$的导数，也就是说**错误地以为**$\nabla^2_xf(x)=\nabla_x(\nabla_xf(x))$，要知道，$\nabla_xf(x)$是一个向量，而在上一小节我们已经重点强调过，**在我们的定义里对向量求偏导是没有定义的**。
>
>&emsp;&emsp;但是$\nabla_x\frac{\partial f(x)}{\partial x_i}$是有意义的，因为$\frac{\partial f(x)}{\partial x_i}$是一个实数，具体地：
>
>$$\nabla_x\frac{\partial f(x)}{\partial x_i}=\left[ \begin{array}{cc} \frac{\partial^2f(x)}{\partial x_i\partial x_1}\\\\\frac{\partial^2f(x)}{\partial x_i\partial x_2}\\\\\vdots\\\\\frac{\partial^2f(x)}{\partial x_i\partial x_n}\end{array}\right]$$
>
>&emsp;&emsp;即海塞矩阵的第i行（或列）。
>
>&emsp;&emsp;希望读者可以好好区分。

##### **3.1.3 总结**

&emsp;&emsp;根据3.1.1和3.1.2小节的内容很容易得到以下等式：
>&emsp;&emsp;$b\in R^{n}, x\in R^n, A\in R^{n\times n}并且A 是对称矩阵$
>&emsp;&emsp;$b,x$均为列向量
>&emsp;&emsp;那么，
>&emsp;&emsp;$\nabla_xb^Tx=b$
>&emsp;&emsp;$\nabla_xx^TAx=2Ax(A是对称阵)$
>&emsp;&emsp;$\nabla^2_xx^TAx=2A(A是对称阵)$
>
>&emsp;&emsp;这些公式可以根据前述定义自行推导，有兴趣的读者可以自己推导一下。
#### 3.2 矩阵乘积和对应元素相乘
&emsp;&emsp;在下一节讲解反向传播原理的时候，尤其是把公式以矩阵形式表示的时候，需要大家时刻区分什么时候需要矩阵相乘，什么时候需要对应元素相乘。

&emsp;&emsp;比如对于矩阵$A=\left[ \begin{array}{cc} 1 & 2 \\\\ 3 & 4\end{array}\right]，矩阵B=\left[ \begin{array}{cc} -1 & -2 \\\\ -3 & -4\end{array}\right]$
&emsp;&emsp;**矩阵相乘**

$$AB=\left[\begin{array}{cc}1\times -1+2\times -3 & 1\times -2+2\times -4 \\\\ 3\times -1+4\times -3 & 3\times -2+4\times -4\end{array}\right]=\left[\begin{array}{cc}-7&-10 \\\\ -15&-22\end{array}\right]$$

&emsp;&emsp;**对应元素相乘**使用符号$\odot$表示：

$$A\odot B=\left[\begin{array}{cc}1\times -1&2\times -2 \\\\ 3\times -3&4\times -4\end{array}\right]=\left[\begin{array}{cc}-1&-4 \\\\ -9&-16\end{array}\right]$$
#### 3.3 梯度下降法原理
&emsp;&emsp;通过之前的介绍，相信大家都可以自己求解梯度矩阵（向量）了。

&emsp;&emsp;那么梯度矩阵（向量）求出来的意义是什么？从几何意义讲，梯度矩阵代表了函数增加最快的方向，因此，沿着与之相反的方向就可以更快找到最小值。如图5所示：

![【图5 梯度下降法 图片来自百度】](https://gallery.angryberry.tech/blog/bp-network/pics/8.png)


&emsp;&emsp;反向传播的过程就是利用梯度下降法原理，慢慢的找到代价函数的最小值，从而得到最终的模型参数。梯度下降法在反向传播中的具体应用见下一小节。

#### 3.4 反向传播原理（四个基础等式）
&emsp;&emsp;反向传播能够知道如何更改网络中的权重$w$ 和偏差$b$ 来改变代价函数值。最终这意味着它能够计算偏导数

$$\frac{\partial L(a^{[l]},y)} {\partial w^{[l]}_{jk}}$$ 

和

$$\frac{\partial L(a^{[l]},y)}{\partial b^{[l]}_j}$$

&emsp;&emsp;为了计算这些偏导数，我们首先引入一个中间变量$\delta^{[l]}_j$，我们把它叫做网络中第$l^{th}$层第$j^{th}$个神经元的误差。后向传播能够计算出误差$\delta_j^{[l]}$，然后再将其对应回$\frac{\partial L(a^{[l]},y)}{\partial w^{[l]}_{jk}}$和$\frac{\partial L(a^{[l]},y)}{\partial b^{[l]}_j}$ 。

&emsp;&emsp;那么，如何定义每一层的误差呢？如果为第$l$ 层第$j$ 个神经元添加一个扰动$\Delta z^{[l]}_j$，使得损失函数或者代价函数变小，那么这就是一个好的扰动。通过选择 $\Delta z^{[l]}_j$与$\frac{\partial L(a^{[l]}, y)}{\partial z^{[l]}_j}$符号相反（梯度下降法原理），就可以每次都添加一个好的扰动最终达到最优。

&emsp;&emsp;受此启发，我们定义网络层第$l$ 层中第$j$ 个神经元的误差为$\delta^{[l]}_j$:

$$\delta^{[l]}_j=\frac{\partial L(a^{[L], y})}{\partial z^{[l]}_j}$$


&emsp;&emsp;于是，每一层的误差向量可以表示为：

$$\delta ^{[l]}=\left[\begin{array}{cc}\delta ^{[l]}_1\\\\\delta ^{[l]}_2\\\\ \vdots \\\\ \delta ^{[l]}_n\end{array} \right]$$

&emsp;&emsp;**下面开始正式介绍四个基础等式【确切的说是四组等式】**
>&emsp;&emsp;**注意：**这里我们的输入为单个样本(所以我们在下面的公式中使用的是损失函数而不是代价函数)。多个样本输入的公式会在介绍完单个样本后再介绍。

- **等式1 ：输出层误差**

$$\delta^{[L]}_j=\frac{\partial L}{\partial a^{[L]}_j}\sigma^{'}(z^{[L]}_j)$$
&emsp;&emsp;其中，$L$表示输出层层数。**以下用$\partial L$ 表示 $\partial L(a^{[L]}, y)$**

&emsp;&emsp;写成矩阵形式是：

$$\delta^{[L]}=\nabla _aL\odot \sigma^{'}(z^{[L]})$$
&emsp;&emsp;**【注意是对应元素相乘，想想为什么？】**

>&emsp;&emsp;**说明**
>
>&emsp;&emsp;根据本小节开始时的叙述，我们期望找到$\partial L \ /\partial z^{[l]}_j$，然后朝着方向相反的方向更新网络参数，并定义误差为：
>
>$$\delta^{[L]}_j=\frac{\partial  L}{\partial z^{[L]}_j}$$
>
>&emsp;&emsp;根据链式法则，
$$ \delta^{[L]}_j = \sum_k \frac{\partial L}{\partial a^{[L]}_k} \frac{\partial a^{[L]}_k}{\partial z^{[L]}_j}$$
&emsp;&emsp;当$k\neq j$时，$\partial a^{[L]}_k / \partial z^{[L]}_j$就为零。结果我们可以简化之前的等式为 
$$\delta^{[L]}_j = \frac{\partial L}{\partial a^{[L]}_j} \frac{\partial a^{[L]}_j}{\partial z^{[L]}_j}$$
&emsp;&emsp;重新拿出定义：$a^{[L]}_j = \sigma(z^{[L]}_j)$，就可以得到：
$$\delta^{[L]}_j = \frac{\partial L}{\partial a^{[L]}_j} \sigma'(z^{[L]}_j)$$
&emsp;&emsp;再"堆砌"成向量形式就得到了我们的矩阵表示式（这也是为什么使用矩阵形式表示需要 *对应元素相乘* 的原因）。

- **等式2： 隐含层误差**
$$\delta^{[l]}_j = \sum_k w^{[l+1]}_{kj} \delta^{[l+1]}_k \sigma'(z^{[l]}_j)$$

&emsp;&emsp;写成矩阵形式：

$$\delta^{[l]}=[w^{[l+1]T}\delta^{[l+1]}]\odot \sigma ^{'}(z^{[l]})$$


>&emsp;&emsp;**说明：**
> 
>$$z_k^{[l+1]}=\sum_j w_{kj}^{[l+1]} a_j^{[l]}+b_k^{[l+1]}=\sum_j w_{kj}^{[l+1]}\sigma\(z^{[l]}_j\)+b^{[l+1]}_k$$
>
>&emsp;&emsp;进行偏导可以获得：$$\frac {\partial z^{[l+1]}_k} {\partial z_j^{[l]}} = w^{[l+1]}_{kj} \sigma'(z^{[l]}_j)$$
>
>&emsp;&emsp;代入得到：$$\delta^{[l]}_j = \sum_k w^{[l+1]}_{kj} \delta^{[l+1]}_k \sigma'(z^{[l]}_j)$$
>


- **等式3：参数变化率**

$$\frac{\partial L}{\partial b^{[l]}_j}=\delta^{[l]}_j$$

$$\frac{\partial L}{\partial w^{[l]}_{jk}}=a^{[l-1]}_k\delta^{[l]}_j$$

&emsp;&emsp;写成矩阵形式：
$$\frac{\partial L}{\partial b^{[l]}}=\delta^{[l]}$$$$\frac{\partial L}{\partial w^{[l]}}=\delta^{[l]}a^{[l-1]T}$$


>&emsp;&emsp;**说明：**
>
>&emsp;&emsp;根据链式法则推导。
>&emsp;&emsp;由于
>$$z^{[l]}_j=\sum_kw^{[l]}_{jk}a^{[l]}_k+b^{[l]}_k$$
>
>&emsp;&emsp;对$b^{[l]}_j$求偏导得到：
>
>$$\frac{\partial L}{\partial b^{[l]}_j}=\frac{\partial L}{\partial z^{[l]}_j}\frac{\partial z^{[l]}_j}{b^{[l]}_j}=\delta^{[l]}_j$$
>
>&emsp;&emsp;对 $w^{[l]}_{jk}$求偏导得到：
>
>$$\frac{\partial L}{\partial w_{jk}^{[l]}}=\frac{\partial L}{\partial z_j^{[l]}} \frac{\partial z_j^{[l]}} {w_{jk}^{[l]}}=a_k^{[l-1]}\delta_j^{[l]}$$
>
>&emsp;&emsp;最后再变成矩阵形式就好了。
>>&emsp;&emsp;对矩阵形式来说，需要特别注意维度的匹配。强烈建议读者在自己编写程序之前，先列出这些等式，然后仔细检查维度是否匹配。
>
>>&emsp;&emsp;很容易看出$\frac{\partial L}{\partial w^{[l]}}$是一个$dim(\delta^{[l]})$行$dim(a^{[l-1]})$列的矩阵，和$w^{[l]}$的维度一致；$\frac{\partial L}{\partial b^{[l]}}$是一个维度为$dim(\delta^{[l]})$的列向量

- **等式4：参数更新规则**

&emsp;&emsp;这应该是这四组公式里最简单的一组了，根据梯度下降法原理，朝着梯度的反方向更新参数：

$$b^{[l]}_j\leftarrow b^{[l]}_j-\alpha \frac{\partial L}{\partial b^{[l]}_j}$$ 


$$w_{jk}^{[l]}\leftarrow w_{jk}^{[l]} -\alpha\frac{\partial L}{\partial w^{[l]}_{jk}}$$ 

&emsp;&emsp;写成矩阵形式：

$$b^{[l]}\leftarrow b^{[l]}-\alpha\frac{\partial L}{\partial b^{[l]}}$$

$$w^{[l]}\leftarrow w^{[l]}-\alpha\frac{\partial L}{\partial w^{[l]}}$$
>&emsp;&emsp;这里的$\alpha$指的是学习率。学习率指定了反向传播过程中梯度下降的步长。

#### 3.5 反向传播总结

&emsp;&emsp;我们可以得到如下最终公式：

##### **3.5.1 单样本输入公式表**

|说明|公式|备注|
|----|---|---|
|输出层误差|$$\delta^{[L]}=\nabla _aL\odot \sigma^{'}(z^{[L]})$$||
|隐含层误差|$$\delta^{[l]}=[w^{[l+1]T}\delta^{[l+1]}]\odot \sigma ^{'}(z^{[l]})$$||
|参数变化率|$$\frac{\partial L}{\partial b^{[l]}}=\delta^{[l]}$$$$\frac{\partial L}{\partial w^{[l]}}=\delta^{[l]}a^{[l-1]T}$$|注意维度匹配|
|参数更新|$$b^{[l]}\leftarrow b^{[l]}-\alpha\frac{\partial L}{\partial b^{[l]}}$$$$w^{[l]}\leftarrow w^{[l]}-\alpha\frac{\partial L}{\partial w^{[l]}}$$|$\alpha$是学习率|

##### **3.5.2 多样本输入公式表**

&emsp;&emsp;**多样本**：需要使用代价函数，如果有m个样本，那么由于代价函数有一个$\frac{1}{m}$的常数项，因此所有的参数更新规则都需要有一个$\frac{1}{m}$的前缀。

&emsp;&emsp;**多样本同时输入的时候需要格外注意维度匹配，一开始可能觉得有点混乱，但是不断加深理解就会豁然开朗。**

|说明|公式|备注|
|----|---|---|
|输出层误差|$dZ^{[L]}=\nabla _AC\odot \sigma^{'}(Z^{[L]})$|此时$dZ^{[l]}$不再是一个列向量，变成了一个$m$列的矩阵，每一列都对应一个样本的向量|
|隐含层误差|$dZ^{[l]}=[w^{[l+1]T}dZ^{[l+1]}]\odot \sigma ^{'}(Z^{[l]})$|此时$dZ^{[l]}$的维度是$n\times m$，$n$表示第l层神经元的个数，m表示样本数|
|参数变化率|$db^{[l]}=\frac{\partial C}{\partial b^{[l]}}=\frac{1}{m}meanOfEachRow(dZ^{[l]})\\\\dw^{[l]}=\frac{\partial C}{\partial w^{[l]}}=\frac{1}{m}dZ^{[l]}A^{[l-1]T}$|更新$b^{[l]}$的时候需要对每行求均值；   注意维度匹配;   $m$是样本个数|
|参数更新|$$b^{[l]}\leftarrow b^{[l]}-\alpha\frac{\partial C}{\partial b^{[l]}}$$$$w^{[l]}\leftarrow w^{[l]}-\alpha\frac{\partial C}{\partial w^{[l]}}$$|$\alpha$是学习率|

##### **3.5.3 关于超参数**

&emsp;&emsp;通过前面的介绍，相信读者可以发现BP神经网络模型有一些参数是需要设计者给出的，也有一些参数是模型自己求解的。

&emsp;&emsp;那么，哪些参数是需要模型设计者确定的呢？

&emsp;&emsp;比如，学习率$\alpha$，隐含层的层数，每个隐含层的神经元个数，激活函数的选取，损失函数（代价函数）的选取等等，这些参数被称之为**超参数**。

&emsp;&emsp;其它的参数，比如权重矩阵$w$和偏置系数$b$在确定了超参数之后是可以通过模型的计算来得到的，这些参数称之为普通参数，简称**参数**。

&emsp;&emsp;超参数的确定其实是很困难的。因为你很难知道什么样的超参数会让模型表现得更好。比如，学习率太小可能造成模型收敛速度过慢，学习率太大又可能造成模型不收敛；再比如，损失函数的设计，如果损失函数设计不好的话，可能会造成模型无法收敛；再比如，层数过多的时候，如何设计网络结构以避免梯度消失和梯度爆炸……

&emsp;&emsp;神经网络的程序比一般程序的调试难度大得多，因为它并不会显式报错，它只是无法得到你期望的结果，作为新手也很难确定到底哪里出了问题（对于自己设计的网络，这种现象尤甚，我目前也基本是新手，所以这些问题也在困扰着我）。当然，使用别人训练好的模型来微调看起来是一个捷径……

&emsp;&emsp;总之，神经网络至少在目前来看感觉还是黑箱的成分居多，希望通过大家的努力慢慢探索吧。

### 4. 是不是猫？
&emsp;&emsp;本小节主要使用上述公式来完成一个小例子，这个小小的神经网络可以告诉我们一张图片是不是猫。本例程参考了coursera的作业，有改动。

&emsp;&emsp;在实现代码之前，先把用到的公式列一个表格吧，这样对照着看大家更清晰一点(如果你没有2个显示器建议先把这些公式抄写到纸上，以便和代码对照)：

|编号|公式|备注|
|:--:|:-------------------------------------------------|--|
|1|$Z^{[l]}=w^{[l]}A^{[l-1]}+b^{[l]}$||
|2|$A^{[l]}=\sigma(Z^{[l]})$||
|3|$dZ^{[L]}=\nabla_AC\odot\sigma^{'}(Z^{[L]})$||
|4|$dZ^{[l]}=[w^{[l+1]T}dZ^{[l+1]}]\odot \sigma ^{'}(Z^{[l]})$|
|5|$db^{[l]}=\frac{\partial C}{\partial b^{[l]}}=\frac{1}{m}meanOfEachRow(dZ^{[l]})$|
|6|$dw^{[l]}=\frac{\partial C}{\partial w^{[l]}}=\frac{1}{m}dZ^{[l]}A^{[l-1]T}$|
|7|$b^{[l]}\leftarrow b^{[l]}-\alpha \cdot db^{[l]}$|
|8|$w^{[l]}\leftarrow w^{[l]}-\alpha\cdot dw^{[l]}$||
|9|$dA^{[l]}=w^{[l]T}\odot dZ^{[l]}$|

&emsp;&emsp;准备工作做的差不多了，让我们开始吧？等等，好像我们还没有定义代价函数是什么？OMG！好吧，看来我们得先把这个做好再继续了。

&emsp;&emsp;那先看结果吧，我们的代价函数是：

$$C = -\frac{1}{m} \sum^{m}_{i=1}(y^{\(i\)}log(a^{[L]\(i\)})+(1-y^{(i)})log(1-a^{[L]\(i\)}))$$

&emsp;&emsp;其中，$m$是样本数量；

&emsp;&emsp;下面简单介绍一下这个代价函数是怎么来的（作者非数学专业，不严谨的地方望海涵）。
.
&emsp;&emsp;代价函数的确定用到了统计学中的**“极大似然法”**，既然这样，那就不可避免地要介绍一下“极大似然法”了。极大似然法简单来说就是“**在模型已定，参数未知的情况下，根据结果估计模型中参数的一种方法**"，换句话说，极大似然法提供了一种给定观察数据来评估模型参数的方法。

&emsp;&emsp;举个例子（本例参考了知乎相关回答），一个不透明的罐子里有黑白两种球（球仅仅颜色不同，大小重量等参数都一样）。有放回地随机拿出一个小球，记录颜色。重复10次之后发现7次是黑球，3次是白球。问你罐子里白球的比例？

&emsp;&emsp;相信很多人可以一口回答“30%”，那么，为什么呢？背后的原理是什么呢？

&emsp;&emsp;这里我们把每次取出一个球叫做一次抽样，把“抽样10次，7次黑球，3次白球”这个事件发生的概率记为$P(事件结果|Model)$，我们的Model需要一个参数$p$表示白球的比例。那么$P(事件结果|Model)=p^3(1-p)^7$。

&emsp;&emsp;好了，现在我们已经有事件结果的概率公式了，接下来求解模型参数$p$，根据极大似然法的思想，既然这个事件发生了，那么为什么不让这个事件（抽样10次，7次黑球，3次白球）发生的概率最大呢？因为显然概率大的事件发生才是合理的。于是就变成了求解$p^3(1-p)^7$取最大值的$p$，即导数为0，经过求导：
$$d(p^3(1-p)^7)=3p^2(1-p)^7-7p^3(1-p)^6=p^2(1-p)^6(3-10p)=0$$
&emsp;&emsp;求解可得$p=0.3$

&emsp;&emsp;**极大似然法**有一个重要的假设：
>假设所有样本独立同分布！！！

&emsp;&emsp;好了，现在来看看我们的神经网络模型。

&emsp;&emsp;最后一层我们用sigmoid函数求出一个激活输出a，如果a大于0.5，就表示这个图片是猫（$y=1$），否则就不是猫（$y=0$）。因此:
$$P(y=1|x;\theta)=a$$
$$P(y=0|x;\theta)=1-a$$
>公式解释：
>上述第一个公式表示，给定模型参数$\theta$和输入$x$，是猫的概率是$P(y=1|x;\theta)=a$

&emsp;&emsp;把两个公式合并成一个公式，即
$$p(y|x;\theta)=a^y(1-a)^{(1-y)}$$
>这里的$\theta$指的就是我们神经网络的权值参数和偏置参数。

&emsp;&emsp;那么似然函数
$$L(\theta)=p(Y|X;\theta)=\prod^m_{i=1}p(y^{\(i\)}|x^{\(i\)};\theta)=\prod^m_{i=1}(a^{[L]\(i\)})^{y^{\(i\)}}(1-a^{[L]\(i\)})^{(1-y^{\(i\)})}$$
&emsp;&emsp;变成对数形式：
$$log(L(\theta))=\sum^m_{i=1}(y^{\(i\)}log(a^{[L]\(i\)})+(1-y^{\(i\)})log(1-a^{[L]\(i\)}))$$
&emsp;&emsp;所以我们的目标就是最大化这个对数似然函数，也就是最小化我们的代价函数：
$$C =-\frac{1}{m} \sum^{m}_{i=1}(y^{\(i\)}log(a^{[L]\(i\)})+(1-y^{\(i\)})log(1-a^{[L]\(i\)}))$$
&emsp;&emsp;其中，$m$是样本数量；

&emsp;&emsp;好了，终于可以开始写代码了，码字手都有点酸了，不得不说公式真的好难打。
>由于代码比较简单就没有上传github。本文代码和数据文件可以在[这里下载https://pan.baidu.com/s/1qYNYA8O](https://pan.baidu.com/s/1qYNYA8O)，密码：zxrb  
>
>其他下载源：
>https://drive.google.com/file/d/0B6exrzrSxlh3TmhSV0ZNeHhYUmM/view?usp=sharing


#### 4.1 辅助函数
&emsp;&emsp;辅助函数主要包括激活函数以及激活函数的反向传播过程函数：
其中，激活函数反向传播代码对应公式4和9.

```python
def sigmoid(z):
    """
    使用numpy实现sigmoid函数
    
    参数：
    Z numpy array
    输出：
    A 激活值（维数和Z完全相同）
    """
    return 1/(1 + np.exp(-z))

def relu(z):
    """
    线性修正函数relu
    
    参数：
    z numpy array
    输出：
    A 激活值（维数和Z完全相同）
    
    """
    return np.array(z>0)*z

def sigmoidBackward(dA, cacheA):
    """
    sigmoid的反向传播
    
    参数：
    dA 同层激活值
    cacheA 同层线性输出
    输出：
    dZ 梯度
    
    """
    s = sigmoid(cacheA)
    diff = s*(1 - s)
    dZ = dA * diff
    return dZ

def reluBackward(dA, cacheA):
    """
    relu的反向传播
    
    参数：
    dA 同层激活值
    cacheA 同层线性输出
    输出：
    dZ 梯度
    
    """
    Z = cacheA
    dZ = np.array(dA, copy=True) 
    dZ[Z <= 0] = 0
    return dZ
```
&emsp;&emsp;另外一个重要的辅助函数是数据读取函数和参数初始化函数：
```python
def loadData(dataDir):
    """
    导入数据
    
    参数：
    dataDir 数据集路径
    输出：
    训练集，测试集以及标签
    """
    train_dataset = h5py.File(dataDir+'/train.h5', "r")
    train_set_x_orig = np.array(train_dataset["train_set_x"][:]) # your train set features
    train_set_y_orig = np.array(train_dataset["train_set_y"][:]) # your train set labels

    test_dataset = h5py.File(dataDir+'/test.h5', "r")
    test_set_x_orig = np.array(test_dataset["test_set_x"][:]) # your test set features
    test_set_y_orig = np.array(test_dataset["test_set_y"][:]) # your test set labels

    classes = np.array(test_dataset["list_classes"][:]) # the list of classes
    
    train_set_y_orig = train_set_y_orig.reshape((1, train_set_y_orig.shape[0]))
    test_set_y_orig = test_set_y_orig.reshape((1, test_set_y_orig.shape[0]))
    
    return train_set_x_orig, train_set_y_orig, test_set_x_orig, test_set_y_orig, classes

def iniPara(laydims):
    """
    随机初始化网络参数
    
    参数：
    laydims 一个python list
    输出：
    parameters 随机初始化的参数字典（”W1“，”b1“，”W2“，”b2“, ...）
    """
    np.random.seed(1)
    parameters = {}
    for i in range(1, len(laydims)):
        parameters['W'+str(i)] = np.random.randn(laydims[i], laydims[i-1])/ np.sqrt(laydims[i-1])
        parameters['b'+str(i)] = np.zeros((laydims[i], 1))
    return parameters
```

#### 4.2 前向传播过程

>对应公式1和2.

```python
def forwardLinear(W, b, A_prev):
    """
    前向传播
    """
    Z = np.dot(W, A_prev) + b
    cache = (W, A_prev, b)
    return Z, cache

def forwardLinearActivation(W, b, A_prev, activation):
    """
    带激活函数的前向传播
    """
    Z, cacheL = forwardLinear(W, b, A_prev)
    cacheA = Z
    if activation == 'sigmoid':
        A = sigmoid(Z)
    if activation == 'relu':
        A = relu(Z)
    cache = (cacheL, cacheA)
    return A, cache

def forwardModel(X, parameters):
    """
    完整的前向传播过程
    """
    layerdim = len(parameters)//2
    caches = []
    A_prev = X
    for i in range(1, layerdim):
        A_prev, cache = forwardLinearActivation(parameters['W'+str(i)], parameters['b'+str(i)], A_prev, 'relu')
        caches.append(cache)
        
    AL, cache = forwardLinearActivation(parameters['W'+str(layerdim)], parameters['b'+str(layerdim)], A_prev, 'sigmoid')
    caches.append(cache)
    
    return AL, caches
```

#### 4.3 反向传播过程

>线性部分反向传播对应公式5和6。
```python
def linearBackward(dZ, cache):
    """
    线性部分的反向传播
    
    参数：
    dZ 当前层误差
    cache （W, A_prev, b）元组
    输出：
    dA_prev 上一层激活的梯度
    dW 当前层W的梯度
    db 当前层b的梯度
    """
    W, A_prev, b = cache
    m = A_prev.shape[1]
    
    dW = 1/m*np.dot(dZ, A_prev.T)
    db = 1/m*np.sum(dZ, axis = 1, keepdims=True)
    dA_prev = np.dot(W.T, dZ)
    
    return dA_prev, dW, db
```
>非线性部分对应公式3、4、5和6 。
```python
def linearActivationBackward(dA, cache, activation):
    """
    非线性部分的反向传播
    
    参数：
    dA 当前层激活输出的梯度
    cache （W, A_prev, b）元组
    activation 激活函数类型
    输出：
    dA_prev 上一层激活的梯度
    dW 当前层W的梯度
    db 当前层b的梯度
    """
    cacheL, cacheA = cache
    
    if activation == 'relu':
        dZ = reluBackward(dA, cacheA)
        dA_prev, dW, db = linearBackward(dZ, cacheL)
    elif activation == 'sigmoid':
        dZ = sigmoidBackward(dA, cacheA)
        dA_prev, dW, db = linearBackward(dZ, cacheL)
    
    return dA_prev, dW, db
```
>完整反向传播模型：
```python
def backwardModel(AL, Y, caches):
    """
    完整的反向传播过程
    
    参数：
    AL 输出层结果
    Y 标签值
    caches 【cacheL, cacheA】
    输出：
    diffs 梯度字典
    """
    layerdim = len(caches)
    Y = Y.reshape(AL.shape)
    L = layerdim
    
    diffs = {}
    
    dAL = - (np.divide(Y, AL) - np.divide(1 - Y, 1 - AL))
    
    currentCache = caches[L-1]
    dA_prev, dW, db =  linearActivationBackward(dAL, currentCache, 'sigmoid')
    diffs['dA' + str(L)], diffs['dW'+str(L)], diffs['db'+str(L)] = dA_prev, dW, db
    
    for l in reversed(range(L-1)):
        currentCache = caches[l]
        dA_prev, dW, db =  linearActivationBackward(dA_prev, currentCache, 'relu')
        diffs['dA' + str(l+1)], diffs['dW'+str(l+1)], diffs['db'+str(l+1)] = dA_prev, dW, db
        
    return diffs
```
#### 4.4 测试结果
&emsp;&emsp;打开你的jupyter notebook，运行我们的BP.ipynb文件，首先导入依赖库和数据集，然后使用一个循环来确定最佳的迭代次数大约为2000：

![【图6】](https://gallery.angryberry.tech/blog/bp-network/pics/6.png)



&emsp;&emsp;最后用一个例子来看一下模型的效果——判断一张图片是不是猫：

![【图7】](https://gallery.angryberry.tech/blog/bp-network/pics/7.png)


好了，测试到此结束。你也可以自己尝试其它的神经网络结构和测试其它图片。

### 5. 本文小结

&emsp;&emsp;本文主要叙述了经典的全连接神经网络结构以及前向传播和反向传播的过程。通过本文的学习，读者应该可以独立推导全连接神经网络的传播过程，对算法的细节烂熟于心。另外，由于本文里的公式大部分是我自己推导的，瑕疵之处，希望读者不吝赐教。

&emsp;&emsp;虽然这篇文章实现的例子并没有什么实际应用场景，但是自己推导一下这些数学公式并用代码实现对理解神经网络内部的原理很有帮助，继这篇博客之后，我还计划写一个如何自己推导并实现卷积神经网络的教程，如果有人感兴趣，请继续关注我！

&emsp;&emsp;本次内容就到这里，谢谢大家。



