---
layout: article
title:  'Kalman Filter!'
tags: SLAM KF
aside:
  toc: true
mathjax: true
mathjax_autoNumber: true
cover: /image/posts/2020-01-12/0.kalman.jpg
---

![kalman](/image/posts/2020-01-12/0.kalman.jpg)

卡尔曼滤波器又被称作传感器融合算法（Sensor Fusion Algorithm），它的原理就是将多个传感器的预测结果或者测量结果进行融合，得到最佳的结果，从而减少误差。

<!--more-->

## 1. 概率分布融合

卡尔曼滤波的推算要从概率分布的融合开始：

为了推算方便，我们先用一维状态进行推算，假设我们有两个传感器（里程计和激光）同时对一辆小车的前进距离$x$进行估计。两个传感器肯定都是有误差的，我们假设预测的结果分别服从高斯分布$\mathcal{N}(u_1, \sigma_1^2)$和$\mathcal{N}(u_2, \sigma_2^2)$ ：

$$f(x)=\frac{1}{\sqrt{2π}\sigma_1}e^{\frac{-(x-u_1)^2}{2\sigma_1^2}} $$

$$g(x)=\frac{1}{\sqrt{2π}\sigma_2}e^{\frac{-(x-u_2)^2}{2\sigma_2^2}} $$

即：
- 里程计预测的结果是小车前进的距离期望是$u_1$，方差为$\sigma_1^2$
- 激光预测的结果是小车前进的距离期望是$u_2$，方差为$\sigma_2^2$

这两个传感器的预测是相互独立的，根据$P(AB)=P(A)P(B)$（两个独立事件同时发生的概率为两个事件分别发生的概率的乘积）两个事件的最优估计的概率分布为：

$$f(x)*g(x)=\frac{1}{\sqrt{2π}\sigma_1}e^{\frac{-(x-u_1)^2}{2\sigma_1^2}} * \frac{1}{\sqrt{2π}\sigma_2}e^{\frac{-(x-u_2)^2}{2\sigma_2^2}} $$

计算后可以得到：

$$f(x)*g(x)=\frac{S_f}{\sqrt{2π}\sigma_f}e^{\frac{-(x-u_f)^2}{2\sigma_f^2}} $$

其中：

$$\sigma_f^2=\frac{\sigma_1^2\sigma_2^2}{\sigma_1^2+\sigma_2^2} $$

$$u_f=\frac{u_1\sigma_2^2+u_2\sigma_1^2}{\sigma_1^2+\sigma_2^2} $$

$$S_f=\frac{1}{\sqrt{2π(\sigma_1^2+\sigma_2^2)}}e^{\frac{-(u_1-u_2)^2}{2(\sigma_1^2+\sigma_2^2)}} $$

由此可以看出，两个高斯分布相乘后的结果还是一个高斯分布。由于$u_1, u_2, \sigma_1, \sigma_2 $都是固定值，因此$S_f$只是一个常数。

该过程即为两个传感器的估计结果融合的过程，融合后的结果为$N(u_f, \sigma_f)$，即两个传感器的结果融合后得到的最佳估计为$u_f$，最佳估计的方差为$\sigma_f^2$。

现在将$u_f$和$\sigma_f$进行变形，如下所示：

$$u_f=u_1 + \frac{\sigma_1^2(u_2-u_1)}{\sigma_1^2+\sigma_2^2}= u_1 + K(u_2-u_1)$$

$$\sigma_f^2=\sigma_1^2-\frac{\sigma_1^4}{\sigma_1^2+\sigma_2^2}=\sigma_1^2-K\sigma_1^2 $$

其中：

$$K=\frac{\sigma_1^2}{\sigma_1^2+\sigma_2^2}$$

公式的物理意义：$u_f$的值等于传感器1的预测结果加上$K(u_2-u_1)$，如果传感器1（里程计）的估计误差越大，即$\sigma_1^2$越大，那么$K$值就越大，最终$u_f$的结果就越偏向于$u_2$的值；假设$\sigma_1^2$趋近于无穷大，那么K就无限趋近于1，那么$u_f=u_2$，由此可见，如果$u_1$误差越大，$u_f$的结果中$u_1$所占的权重就越小；反之，如果$\sigma_2$越大，则$K$越趋近于0，$u_f$越趋近于$u_1$。这就是卡尔曼滤波器被称为传感器融合算法的原因，其中 <font color="#FF0000">K</font> 就是最重要的用于调节两个传感器结果的系数。

## 2.矩阵表示

上面我们得到了一维状态两种传感器预测结果的融合结果，实际运用中，状态值往往是多维的，我们需要将其转化为矩阵的表达形式。

我们已经得到了以下三个公式：

$$u_f = u_1 + K(u_2-u_1)$$

$$\sigma_f^2 = \sigma_1^2-K\sigma_1^2$$

$$ K = \frac{\sigma_1^2}{\sigma_1^2+\sigma_2^2} $$


在使用一维状态进行计算时，状态使用高斯分布表示为：$\mathcal{N}(\mu, \sigma^2)$，物理意义为机器人前进的距离x的期望为$\mu$，方差为$\sigma^2$，在多维状态中，表示为矩阵形式的高斯分布为：$\mathcal{N}(\vec{\mu}, \Sigma)$，假设是三维状态，那么$\vec{\mu} = \left[\begin{matrix} x_1 \\\ x_2 \\\ x_3 \end{matrix}\right]$，如果各个状态是不相关的，那么$\Sigma=\left[\begin{matrix} \sigma_1^2 & 0 & 0 \\\ 0 & \sigma_2^2 & 0 \\\ 0 & 0& \sigma_3^2\end{matrix}\right]$，但是现实中往往状态之间是有一定的相关性的，比如状态里面如果包含前进的距离和前进时的速度，那么速度的误差越大，往往距离的误差也就越大，两个状态就是正相关的，这种相关性可以使用协方差矩阵来表示，即$\Sigma_{ij}$表示状态$x_i$和状态$x_j$之间的关系，那么重新用矩阵表示方差为：$\Sigma=\left[\begin{matrix} \Sigma_{1,1} & \Sigma_{1,2} & \Sigma_{1,3} \\\ \Sigma_{2,1} & \Sigma_{2,2} & \Sigma_{2,3} \\\ \Sigma_{3,1} & \Sigma_{3,2}& \Sigma_{3,3}\end{matrix}\right]$。

将这三个公式切换为矩阵的形式为：

$$\vec{\mu'} = \vec{\mu_1} + K(\vec{\mu_2} - \vec{\mu_1})$$

$$\Sigma' = \Sigma_1-K\Sigma_1$$

$$\color{#ff0000}{K} =\Sigma_1(\Sigma_1+\Sigma_2)^{-1}$$

其中，使用$\mathcal{N}(\vec{\mu_1}, \Sigma_1)$替换$\mathcal{N}(u_1, \sigma_1^2)$，使用$\mathcal{N}(\vec{\mu_2}, \Sigma_2)$替换$\mathcal{N}(u_2, \sigma_2^2)$，得到的结果使用$\mathcal{N}(\vec{\mu'}, \Sigma')$替换$\mathcal{N}(u_f, \sigma_f^2)$。$(\Sigma_1+\Sigma_2)^{-1}$为矩阵$(\Sigma_1+\Sigma_2)$的逆矩阵，两者相乘后为单位矩阵。



## 3.物理量表示

上面我们已经推算了两种传感器数据融合的过程以及结果，那么实际使用的时候该怎么将机器人的状态和传感器数据统一起来呢？

### 1) $\mu$的意义
上述的$\mu_1$和$\mu_2$是同一个物理量的期望，可以是机器人实际前进的距离，也可以是传感器的精确值，因为机器人状态和传感器实际值必然存在确定的关系，所以无论$\mu$是指机器人状态还是传感器的精确值，最后都可以通过转化关系得到另一个值。通常使用机器人的状态值去计算传感器的结果是比较简单的，所以这里的$\mu$我们指的是传感器的精确值。

### 2) 关系表示
机器人k时刻的实际状态我们用$x_k$表示，传感器k时刻的精确值我们用$y_k$表示，机器人的状态与传感器值之间必然存在一定的转化关系，这种转化关系我们定义为$H$，k时刻的转化关系我们就定义为$H_k$，那么我们可以得到$y_k = H_k x_k$。

### 3) $\mu$的转化
假设我们上述的$\mu$的物理意义就是传感器的精确值，也就是$y_k$，我们用$\hat{x}_k$表示k时刻的机器人状态（位姿）最佳估计值，也是后验状态；用$\hat{x}_k^-$表示k时刻预测的机器人的状态值，也是先验状态。那么：$\vec{\mu'} = H_k\hat{x}_k$，$\vec{\mu_1} = H_k\hat{x}_k^-$。

那么，上面三个公式的第一个公式，我们就可以转化为下面的形式：

$$H_k\color{blue}{\hat{x}_k} = H_k\color{orange}{\hat{x}_k^-} + K(\vec{\mu_2} - H_k\color{orange}{\hat{x}_k^-})$$

假设$\mu_2$是第二个传感器直接测得的结果的期望$z_k$，协方差矩阵为$R_k$，那么进一步将上式转化为：

$$H_k\color{blue}{\hat{x}_k} = H_k\color{orange}{\hat{x}_k^-} + K(z_k - H_k\color{orange}{\hat{x}_k^-})$$

### 4) $\Sigma$的转化

上面我们将$\mu$进行了转化，$\mu_1$由机器人的状态转化而来，$\mu_2$由传感器直接测得，$\mu_2$对应的协方差矩阵为$R$，那么，$\mu_1$对应的协方差矩阵$\Sigma_1$如何转化呢？

根据协方差矩阵的性质：$Cov(Ax)=ACov(x)A^T$，我们可以得到，当状态左乘$H_k$时，对应的协方差矩阵为原协方差矩阵左乘$H_k$并且右乘$H_k$的转置。由此，我们可以得到$\Sigma_k = H_k\Sigma_{k-1}H_k^T$。将$x_k$对应的协方差矩阵用$P_k$表示，我们可以将原来的高斯分布 $\mathcal{N}(\vec{\mu_1}, \Sigma_1)$ 表示为 $\mathcal{N}(H_k\hat{x}_k^-, H_kP_k^-H_k^T)$，原来的$\mathcal{N}(\vec{\mu'}, \Sigma')$转化为：$\mathcal{N}(H_k\hat{x}_k, H_kP_kH_k^T)$ 。

那么，剩下的两个公式我们转化为：

$$H_k\color{blue}{P_k}H_k^T = H_kP_k^-H_k^T -\color{red}{K}H_kP_k^-H_k^T$$

$$\color{red}{K}=H_kP_k^-H_k^T(H_kP_k^-H_k^T+R_k)^{-1}$$

### 5) 公式简化

将上面的三个公式综合一下：

$$H_k\color{blue}{\hat{x}_k} = H_k\color{orange}{\hat{x}_k^-} + \color{red}{K}(z_k - H_k\color{orange}{\hat{x}_k^-})$$

$$H_k\color{blue}{P_k}H_k^T = H_k\color{orange}{P_k^-}H_k^T -\color{red}{K}H_k\color{orange}{P_k^-}H_k^T$$

$$\color{red}{K}=H_k\color{orange}{P_k^-}H_k^T(H_k\color{orange}{P_k^-}H_k^T+R_k)^{-1}$$

将公式1左乘$H_k^-$，公式2左乘$H_k^-$，然后右乘$H_k^T$的逆，我们可以得到：

$$\color{blue}{\hat{x}_k} = \hat{x}_k^- + \color{red}{K'}(z_k - H_k\hat{x}_k^-)$$

$$\color{blue}{P_k} = \color{orange}{P_k^-} -\color{red}{K'}H_k\color{orange}{P_k^-}$$

$$\color{red}{K'}=\color{orange}{P_k^-}H_k^T(H_k\color{orange}{P_k^-}H_k^T+R_k)^{-1}$$

### 6) 物理意义

上述三个公式的物理意义为：
- 公式1: 我们可以由机器人k时刻的先验状态估计$\hat{x}_k^-$以及k时刻的测量值$z_k$得到k时刻的后验状态估计$\hat{x}_k$。
- 公式2: 我们可以由k时刻机器人的先验状态估计的协方差矩阵得到后验估计协方差矩阵。
- 公式3: 计算新的卡尔曼增益(K)

上面我们已经将两个传感器的结果进行了融合，虽然我们使用的$\mu$的物理意义是传感器的精确值，但是实际我们最后得到的结果是机器人的状态值。

### 7) 预测公式

我们是将当前状态的先验状态估计作为已知变量进行计算的，但是，实际情况是，我们只知道上一时刻的最佳状态估计值，也就是$\hat{x}_{k-1}$，怎么由上一时刻的结果，得到这个先验状态估计值呢？这个需要根据实际情况进行计算：

比如，我们机器人的上一时刻位置为$x_{k-1}$，速度为$v$，那么当前时刻的位置应当为
$x_k = x_{k-1} + vt$，总之，我们可以通过一定的线性变化$A$以及系统中的控制量$u_k$得到先验估计值，那么，先验状态估计可以由以下公式得到：

$$\color{orange}{\hat{x}_k^-} = A \color{blue}{\hat{x}_{k-1}} + Bu_k$$

先验状态估计对应的协方差矩阵类似得可以表示为：

$$\color{orange}{P_k^-} = A\color{blue}{P_{k-1}}A^T + Q$$

其中，$Q$是系统控制过程中的噪声。

由此，我们已经从上一时刻的后验状态估计得到了下一时刻的先验状态估计。

## 4. 综述

经过以上过程的推算，我们已经可以由上一时刻的系统状态值得到当前时刻的系统状态值，可以分为两个过程，预测和更新的过程。假设我们已经得到了上一时刻的状态值$$\color{blue}{\hat{x}_{k-1}}$$以及$\color{blue}{P_{k-1}}$
。

首先根据系统的控制量进行预测的过程，可以得到当前状态的预估值：

$$\color{orange}{\hat{x}_k^-} = A \color{blue}{\hat{x}_{k-1}} + Bu_k$$

$$\color{orange}{P_k^-} = A\color{blue}{P_{k-1}}A^T + Q$$

然后，我们进行传感器融合的过程，得到当前时刻状态的最优估计：

首先计算卡尔曼增益：

$$\color{red}{K'}=\color{orange}{P_k^-}H_k^T(H_k\color{orange}{P_k^-}H_k^T+R_k)^{-1}$$

随后计算出最终估计值：

$$\color{blue}{\hat{x}_k} = \hat{x}_k^- + \color{red}{K'}(z_k - H_k\hat{x}_k^-)$$

$$\color{blue}{P_k} = \color{orange}{P_k^-} -\color{red}{K'}H_k\color{orange}{P_k^-}$$

这样就完成了一次迭代的过程，之后可以根据$$\hat{x}_k$$和$$P_k$$再计算$$\hat{x}_{k+1}$$和$$P_{k+1}$$。


