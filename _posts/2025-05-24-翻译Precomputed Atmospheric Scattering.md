---
layout: post
title:  "[论文翻译] 预计算大气散射 Precomputed Atmospheric Scattering"
date:   2025-05-24 15:57:00 +0800
categories: post
math: true
image: /assets/images/Precomputed Atmospheric Scattering/image0.png
---

[原文]是Eric Bruneton and Fabrice Neyret发表于2008年的论文， 
原作者2017年还更新了代码 [更新的源码]
作为实时渲染大气的一个重要文章，感觉有必要看下原文 ，顺便翻译下。大部分GPT翻译，简单校对：


![alt text](/assets/images/Precomputed Atmospheric Scattering/image.png)

## 摘要

我们提出了一种新的高精度方法，
可以实时从地面到外太空的任意视角渲染大气，
同时考虑瑞利散射和米氏散射中的多重散射效应。
该方法能够真实再现多种光散射现象，
例如白天和黄昏时天空的颜色、
各种视角和光照方向下的空气透视效果，
以及地球和山体在大气中投下的阴影（光束）。
我们的方法基于一种光传输方程的表达形式，
该表达形式可以对所有视点、视角方向和太阳方向进行预计算。
我们展示了如何紧凑地存储这些数据，
并提出了一种兼容GPU的算法， 可在几秒内完成预计算。
这些预计算数据使我们能够在运行时以恒定时间（constant Time）计算光传输方程，
无需采样，同时还能考虑地面造成的阴影和光束（light Shafts）。

## 1.引言

跳过介绍部分--大概是重要性和要达到什么效果
接下来的章节安排如下：第2节介绍物理模型和渲染方程，
并回顾相关工作。
第3节展示我们为实现可预计算表达形式所采用的求解方法。
第4节和第5节分别介绍我们的预计算算法和渲染算法。
第6节提供实现细节并展示我们的结果。

## 2.大气模型

大气光照渲染依赖于两个方面：一是局部介质属性的物理模型，
二是到达观察者眼睛为止的全局光照交换的模拟。
这其中包括与地面的光照交换（原文exchanges），
与地面的交换可以被建模为一个Lambertian surface，带有
reflectance α(x,λ) 和法线 n(x) 等属性的高度场。

从\[NSTN93]开始，大多数的CG（computer Graphics）论文，基于一个包括空气分子和气溶胶（aerosol）粒子的物理模型，
在2.1中概述。然而，传统的参与介质（participating media）
渲染方程在大气CG模型中很少被完整考虑，尤其是在交互式渲染中。我们将在第2.2节重新阐述这一通用模型，
并在第2.3节介绍以往CG模型中对此的近似处理。



![alt text](/assets/images/Precomputed Atmospheric Scattering/image-1.png)

**图1：**我们的方法。左侧：左图：参考方案 包括从点 x 到  $$ x_0 $$ 积分的单次散射 (a) 和多次散射 (b)，两者均考虑了遮挡。右图：我们的近似方法。积分是从点
x 到  $$ x_s $$  ，忽略了遮挡（遮挡通过使用  $$ x_s $$  隐式处理）。(a) 保持不变；(b) 由于忽略了次级散射的遮挡而受到影响（这会带来正向或负向的偏差，但影响很小）



### 2.1 物理模型

计算机图形中常用的物理模型是晴空模型（clear sky
model），该模型基于两种成分：空气分子和气溶胶粒子。这些成分分布在一个密度递减的薄球形层中，半径范围从  $$ R_g = 6360 $$ 
公里到  $$ R_t = 6420 $$ 公里（见图1）。
在每个点，光线从入射方向散射到偏离入射方向的角度为  $$ \theta $$ 的方向上的比例由散射系数  $$ \beta^s $$ 和相位函数  $$ P $$ 的乘积给出。
 $$ \beta^s $$ 取决于粒子密度，而  $$ P $$ 则描述了散射的角度依赖性。对于空气分子， $$ \beta^s $$ 和  $$ P $$ 可由瑞利散射理论 \[TS99] 给出：

![alt text](/assets/images/Precomputed Atmospheric Scattering/image-3.png)

其中， $$ h = r - R_g $$  表示海拔高度， $$ \lambda $$  是波长， $$ n $$ 是空气的折射率， $$ N $$ 是海平面 $$ R_g $$ 处的分子密度，而  $$ H_R = 8 $$ 
公里是如果大气密度为均匀时的大气厚度。按照 \[REK∗04] 的方法，我们使用
 $$ \beta^s_R = (5.8, 13.5, 33.1) \times 10^{-6} \, \text{m}^{-1} $$ 对应于波长  $$ \lambda = (680, 550, 440) \, \text{nm} $$ 。
气溶胶的密度同样呈指数衰减，其高度尺度更小，约为  $$ H_M ≃ 1.2 $$ 公里。它们的相位函数由米氏散射理论给出，使用 Cornette-Shanks
相位函数进行近似 \[TS99]：

![alt text](/assets/images/Precomputed Atmospheric Scattering/image-4.png)

不像空气分子，气凝胶会吸收一部分入射的光，入射光的吸收由吸收系数  $$ \beta^a_M $$ 
描述，进而得到消光（extinction）系数  $$ \beta^e_M = \beta^s_M + \beta^a_M $$ （见图6中的典型数值 ——
对于空气分子， $$ \beta^e_R = \beta^s_R $$  ）。
需要注意的是，折射率随高度的变化会导致光线发生轻微弯曲（小于2度 [HMS05]），为简化处理，我们在此忽略这一影响。

### 2.2 渲染方程

我们在此回顾适用于参与介质的渲染方程，应用于大气。记  $$ L(x, v, s) $$ 为当太阳在方向  $$ s $$ 上时，从方向  $$ v $$ 到达点  $$ x $$ 
的辐射亮度（radiance）；记  $$ x_o(x, v) $$ 为射线  $$ x + tv $$ 的终点（见图1）。需要注意的是， $$ x_o $$ 
要么位于地面上，要么位于大气顶层边界  $$ r = R_t $$ 
处。
从  $$ x_o $$ 到  $$ x $$ 的透射率(transmittance)  $$ T $$ 、在  $$ x_o $$ 处反射光的辐射亮度(radiance)  $$ I $$ ，以及在点  $$ y $$ 向方向  $$ -v $$ 
散射光的辐射亮度  $$ J $$ ，定义如下（见图2）：



![alt text](/assets/images/Precomputed Atmospheric Scattering/image-2.png)

**图2：**定义说明。
(a) 大气透射率  $$ T $$ 由吸收和向外散射的光共同得出。
(b)  $$ I[L] $$ 表示在  $$ x_o $$ 处反射的光  $$ L $$ 。在大气顶层边界上，该值为零。
(c)  $$ J[L] $$ 表示在点  $$ y $$ 处向方向  $$ -v $$ 散射的光  $$ L $$ 。
(d)  $$ S[L] $$ 表示从任意方向在  $$ x_o $$ 到  $$ x $$ 之间散射向点  $$ x $$ 的光  $$ L $$ 。



![alt text](/assets/images/Precomputed Atmospheric Scattering/image-5.png)

请注意， $$ I $$ 在大气顶层边界处为零。
使用上述符号，渲染方程为 [TS99]：

![alt text](/assets/images/Precomputed Atmospheric Scattering/image-6.png)

其中， $$ L_0 $$ 表示在到达x之前被透射率  $$ T(x, x_o) $$ 衰减  $$ x $$ 的直接太阳光 $$ L_{\text{sun}} $$ 。当  $$ v \neq s $$ ，
或太阳被地形遮挡（即  $$ x_o $$ 位于地面上）时， $$ L_0 $$ 为零。
 $$ R[L] $$ 表示在  $$ x_o $$ 处反射的光，并在到达  $$ x $$ 前被 $$ T(x, x_o) $$  衰减；而  $$ S[L] $$ 从  $$ x $$ 到  $$ x_o $$ 之间散射到  $$ x $$ 
的光（inscattered）（见图2）。

### 2.3 之前的渲染方法

方程（8）非常复杂，难以求解。因此，为了获得计算更简单的近似解，CG中引入了许多简化假设（参见 \[Slo02]
的综述）。大多数实时渲染方法会忽略多重散射。在这种情况下，方程（8）简化为：
 $$ L = L_0 + R[L_0] + S[L_0] $$ 
然而，即使是  $$ S[L_0] $$ 也相当复杂，难以求解。一些作者通过理想化的假设提出了解析解，例如：假设地球是平的，且大气密度恒定 \[HP02]
，或忽略米氏散射 \[REK∗04]。
平地球的假设限制了这些方法只能应用于地面观察者。除此之外， $$ S[L_0] $$ 通常通过数值积分计算 \[NSTN93]，而这可以通过低采样率实现实时计算
\[O'N05]。
一个值得注意的例外是 \[SFE07]，他们依赖于对该积分的预计算。然而，为了减少参数数量，他们只考虑了视天顶角和太阳天顶角，忽略了视线方向与太阳方向之间的夹角。因此，他们无法模拟如地球在大气中投下的阴影等现象。

如上所述，在白天忽略多重散射是可以接受的，但在黄昏时就不行了 \[HMS05]。这是因为白天阳光穿过的大气层远少于日出或日落时。
因此，一些作者提出了考虑多重散射的方法。\[PSS99]
使用双重散射蒙特卡洛模拟的结果，并拟合出一个解析模型，但他们的模型仅适用于地面观察者。\[NDKY96] 和 \[HMS05]
使用体积光能算法来计算多重散射，但这些方法远不能实时处理（每帧需数分钟至数小时）。

在本文中，我们提出了一种新的方法，可以在实时条件下渲染天空和空气透视效果，适用于从地面到太空的所有观察视角，同时考虑多重散射。该方法受
\[SFE07]
启发，并在以下方面进行了扩展：
引入了多重散射的处理，
加入了先前被忽略的“视线-太阳方向夹角”参数，
对预计算表进行了更优的参数化设计，
提出了用于模拟光束的新方法。

## 3. 我们的方法

为了兼顾效率和真实感，我们的目标是尽可能多地预计算光照  $$ L $$ 
，只做最小的近似处理。我们的方法在零次和单次散射时采用精确计算，而多次散射则通过近似遮蔽效应来实现。实际上，在处理零次和单次散射时，我们会考虑地面形状的细节，以获得正确的地面颜色、阴影和光束效果。但在计算多次散射时，我们将地面近似为一个具有恒定反射率的完美球体，以便实现预计算。

**符号说明**
在介绍我们的方法之前，我们需要先引入一些符号和辅助函数。
我们记  $$ \bar{L} = \bar{L}_0 + (\bar{R} + \bar{S})[\bar{L}] $$ 为方程 （8）在地面为具有恒定反射率  $$ \bar{\alpha} $$ 
的完美球体情况下的解。 $$ \bar{L}_0、\bar{R}、\bar{S}、\bar{x}_0、\bar{I} $$ 
等变量的定义与之前相同，但它们现在是针对这个球形地面的。需要注意的是，由于地面具有球面对称性，位置  $$ x $$ 和视角  $$ v $$ 
可以简化为海拔和视线天顶角。因此，像  $$ \bar{L} $$ 或  $$ \bar{S}[\bar{L}] $$ 这样的  $$ x, v, s $$ 的函数可以简化为只依赖4个参数（ $$ x, v $$ 
两个参数， $$ s $$ 两个参数）的函数。
还需注意， $$ L $$ （或  $$ \bar{L} $$ ）可以用线性算子  $$ R $$ 和  $$ S $$ （或  $$ \bar{R} $$ 和  $$ \bar{S} $$ ）的级数表达式来表示，其中第  $$ i $$ 
项对应的是光线正好经过  $$ i $$ 次反射和/或散射后的贡献：

![image-7.png](/assets/images/Precomputed Atmospheric Scattering/image-7.png)

**零次和一次散射**
我们在渲染过程中精确地计算  $$ L_0 $$ 和  $$ R[L_0] $$ 。为此，我们使用一个遮蔽算法来计算阳光遮挡（见公式
9），并使用一个预计算的透射率表  $$ T $$ ，它仅依赖于两个参数（见第4节）。
 $$ S[L_0] $$ 更复杂。它是一个从  $$ x $$ 到  $$ x_o $$ 的积分，但由于  $$ L_0 $$ 中包含遮蔽项，因此在阴影区域中的所有点  $$ y $$ 
处，被积函数为零（这就是产生光束的原因）。我们在此假设这些点位于  $$ x_s $$ 和  $$ x_o $$ 之间（见图1 —— 更一般的情形将在第5节中讨论）。
因此，该积分可以简化为在明亮段  $$ [x, x_s] $$ 上进行。此外，遮蔽效应可以忽略，因为它已经通过  $$ x_s $$ 考虑进来了，也就是说， $$ L_0 $$ 
可以被替换为  $$ \bar{L}_0 $$ 。这表明： $$ S[L_0] = \int_x^{x_s} T J[\bar{L}_0] $$ 
通过将其重写为： $$ \int_x^{\bar{x}_o} T J[\bar{L}_0] - \int_{x_s}^{\bar{x}_o} T J[\bar{L}_0] $$ 
，拓展了 \[O'N05] 中提出并在 \[SFE07] 中复用的一个思想，最终得出一个仅依赖于两个和四个参数（即  $$ T $$ 和  $$ \bar{S}[\bar{L}_0] $$ 
）的可预计算函数形式。

![alt text](/assets/images/Precomputed Atmospheric Scattering/image-8.png)

**多次散射**
如上所示，尽管存在遮蔽， $$ L_0 $$ 和  $$ L_1 $$ 仍然可以被精确计算。不幸的是，在其他项中考虑遮蔽（即  $$ L_2 + \ldots = R[L_*] + S[L_*] $$ 
）要困难得多。幸运的是，在这种情况下遮蔽效应可以近似处理。事实上，在白天时，多次散射效应相较于一次散射非常小，地面没有被阳光直接照射时对光照的贡献很小。
因此，我们在计算  $$ S[L_*] $$ 时，采用不考虑遮蔽情况下的多次散射贡献，并在  $$ x $$ 到  $$ x_s $$ 
之间对其进行积分。这个近似可能会引入正偏差或负偏差（见图1）。数学上，这种近似表示为：
 $$ S[L_*] \approx \int_x^{x_s} T J[\bar{L}_*] $$ 

我们还通过水平半球环境遮蔽，来近似  $$ R[L_*] $$ 中地面切平面导致的遮蔽效应，即： $$ \frac {1 + n\cdot\bar{n}}{2} $$ 
从而得到近似关系：
 $$ 
R[L_*] \approx \hat{R}[\bar{L}_*]
 $$ 

![alt text](/assets/images/Precomputed Atmospheric Scattering/image-9.png)  
  
通过使用与公式(13)相同的重写规则，  
并记$$ \bar{S}[\bar{L}]_|x = \bar{S}[\bar{L}](x, v, s) $$，我们最终得到:

![alt text](/assets/images/Precomputed Atmospheric Scattering/image-10.png)

借助为  $$ T $$ 和  $$ \bar{\varepsilon}[\bar{L}_*] $$ 预计算的2D表， 前三项可以快速的计算出来，以及  $$ \bar{S}[\bar{L}] $$ 可以预计算为4D
表的。我们现在将展示如何在合理大小的查找表中对它们进行预计算。

## 4.预计算

### 算法 4.1

![alt text](/assets/images/Precomputed Atmospheric Scattering/image-11.png)

我们预计算二维表格  $$ \mathbb{T} (x,v) $$ 中所有 x 和 v 情况下的  $$ T(x, \bar xo(x,v)) $$ 。由于具有球面对称性， $$ \mathbb T $$ 
仅依赖于变量  $$ r = ∥x∥ $$ 和  $$ µ = v·x / r $$ \[O'N05]。与 \[O'N05] 一样，我们接着使用恒等式：
 $$ T(x, y) = \mathbb T(x, v) / \mathbb T(y, v) $$ ，其中  $$ v = (y − x) / ∥y − x∥ $$ 
我们使用一个算法逐阶计算每一个散射阶数  $$ \bar{L}_i $$ ，从而将  $$ \bar{\varepsilon}[\bar{L}_*] $$ 和  $$ \bar{S}[\bar{L}_*] $$ 
预计算并存储在两个表  $$ \mathbb E $$ 和  $$ \mathbb S $$ 中。该算法使用三个中间表  $$ \Delta \mathbb E $$ 、 $$ \Delta \mathbb S $$ 和
 $$ \Delta \mathbb J $$ ，在每次迭代  $$ i $$ 
后分别包含：
 $$ \bar{\varepsilon}[\bar{L}_i] $$ 
 $$ \bar{S}[\bar{L}_i] $$ 
 $$ \bar{J}[\bar{L}_i] $$ 
在每次迭代结束时，将  $$ \Delta \mathbb E $$ 和  $$ \Delta \mathbb S $$ 加到结果表  $$ \mathbb E $$ 和  $$ \mathbb S $$ 中。
 $$ \bar{R} $$ 是根据以下恒等式计算的：
 $$ 
\bar{R}[L](x, v, s) = T(x, \bar{x}_o) \cdot \frac{\bar{\alpha}}{\pi} \cdot \bar{\varepsilon}[L](\bar{x}_o, s)
 $$ 
详见算法 4.1。

**角度精度** 由于  $$ \mathbb S $$ 是一个四维表格，其大小会随着分辨率迅速增加。因此我们只能对  $$ v $$ 
使用有限的角度分辨率。这带来了精度问题，但该问题主要局限于强烈的前向米氏散射。为了解决这个问题，我们将单次米氏散射项从  $$ \mathbb S $$ 
中的其他项中分离出来，以便在运行时应用相位函数。为此，我们将  $$ \bar{S}[\bar{L}] $$ 重写为：
 $$ 
P_M \bar{S}_M[\bar{L}_0] + P_R \bar{S}_R[\bar{L}_0] + \bar{S}[\bar{L}_*]
 $$ 
然后我们分别存储：
 $$ C_M = \bar{S}_M[\bar{L}_0] $$ 
和 $$ C_* = \bar{S}_R[\bar{L}_0] + \bar{S}[\bar{L}_*] / P_R $$ 
这对  $$ \mathbb S $$ 中的每个条目需要存储 6 个数值。如果需要提高效率，可以只存储  $$ C_M $$ 的红色分量  $$ C_{M,r} $$ ，将每个条目的存储量减少到
4
个数值。在这种情况下，其他分量可以通过  $$ \bar{S}_R[\bar{L}_0] $$ 与  $$ \bar{S}_M[\bar{L}_0] $$ 之间的比例关系来近似，这得出

![img.png](/assets/images/Precomputed Atmospheric Scattering/img.png)



![img_1.png](/assets/images/Precomputed Atmospheric Scattering/img_1.png)
**图3： 视线角度参数**。 左侧：使用 $$ \mu $$ 带来伪影。 右侧：使用  $$ u_\mu = d_o/d_h $$ 或者  $$ d_o / d_H $$ 解决这个问题（ $$ \mu $$ 
和  $$ u_\mu $$ 在预计算天空radiance表  $$ \mathbb S $$ 中使用了128个值）

![img_2.png](/assets/images/Precomputed Atmospheric Scattering/img_2.png)
**图4： 参数化**。  $$ u_r $$ 、 $$ u_\mu $$ 、 $$ u_{\mu_s} $$ 作为  $$ r $$ 、 $$ \mu $$ 、 $$ \mu_s $$ 的函数。



**参数化** 为了将  $$ \bar{S}[\bar{L}] $$ 存储到  $$ \mathbb S $$ 中，我们需要一个从  $$ (x, v, s) $$ 映射到表索引  $$ [0,1]^4 $$ 
的映射。一个简单的解决方案是使用  $$ r = \|x\| $$ 
以及视角天顶角、太阳天顶角和视-太阳角的余弦，分别为  $$ \mu = {v \cdot x} / {r} $$ 、 $$ \mu_s = {s \cdot x} / {r} $$ 
和  $$ \nu = v \cdot s $$ ，将其从  $$ [R_g, R_t] \times [-1, 1]^3 $$ 线性映射到  $$ [0,1]^4 $$ 。

该参数化的问题在于，需要非常高的  $$ \mu $$ 
分辨率才能对空中透视进行良好采样。例如，考虑一位靠近地面的观察者水平观看远处一座山，距离为  $$ d $$ （见图3）。空中透视由公式(16)给出：

 $$ 
S(x, v, s) - T(x, x_s) S(x_s, v, s)
 $$ 

此时，对于  $$ x $$ ，有  $$ \mu = 0 $$ ，而对于  $$ x_s $$ ，有  $$ \mu = {d} / {\sqrt{r^2 + d^2}} $$ ，当  $$ d = 100\,\text{km} $$ 
时，得到  $$ \Delta \mu = 0.016 \ll 1 $$ ，这个太小的值会产生明显的伪影（见图3）。

为了解决这个问题，我们采用了更优的参数化方式。我们用  $$ u_\mu $$ 替代  $$ \mu $$ ，其定义为从  $$ x $$ 到  $$ \bar{x}_o $$ 
的距离  $$ d_o = \|\bar{x}_o - x\| $$ 与从  $$ x $$ 到地平线（或“大气边界”）的距离  $$ d_h $$ （或  $$ d_H $$ 
）的比值（见图3）。在前述例子中，对于  $$ x $$ 和  $$ x_s $$ ，有  $$ d_H \simeq \sqrt{R_t^2 - R_g^2} $$ ，而  $$ d_o \simeq d_H $$ 
对于  $$ x $$ ， $$ d_o \simeq d_H - d $$ 对于  $$ x_s $$ ，因此当  $$ d = 100\,\text{km} $$ 时，得到  $$ \Delta u_\mu = 0.11 \gg 0.016 $$ 。
使用此映射时，128 个  $$ u_\mu $$ 采样即可避免上述伪影。

另一个问题是，由于视线在地平线处长度的不连续性， $$ \mathbb S $$ 
在地平线处也是不连续的。因此，连续映射会在该不连续处进行线性插值，从而导致伪影。我们通过确保  $$ u_\mu $$ 
本身在地平线处不连续来解决此问题（见图4）。最后，我们对  $$ r $$ 和  $$ \mu_s $$ 使用一个特定的非线性映射，以在靠近地面和太阳天顶角接近
90° 时获得更高的精度。因此，我们从  $$ (x, v, s) $$ 到  $$ [0,1]^4 $$ 的映射最终定义如下：

![img_3.png](/assets/images/Precomputed Atmospheric Scattering/img_3.png)

## 5. 渲染

为了渲染天空和空中透视效果，我们在每个像素处计算公式(16)。
 $$ \bar{L}_0 $$ 可以通过  $$ \mathbb  T $$ 高效计算得到。计算  $$ R[\bar{L}_0] $$ 涉及  $$ \mathbb T $$ 、 $$ \alpha(x_o) $$ 、 $$ n(x_o) $$ 
以及一个用于判断  $$ x_o $$ 是否被照亮的阴影测试。
最后，使用  $$ \mathbb E $$ 和  $$ \mathbb S $$ 来计算  $$ \hat{R}[\bar{L}_*] $$ 和  $$ \bar{S}[\bar{L}] $$ 。如同 \[SFE07] 所述， $$ x $$ 
是相机位置，或在相机位于太空中时，是视线与大气边界的最近交点。
唯一剩下的非平凡参数是  $$ x_s $$ ，它依赖于地形阴影并生成光束效应。

大多数光束算法使用采样或切片的方式，在视线方向上执行数值积分，并利用阴影贴图判断哪些采样点被照亮。为了消除由于离散采样引起的伪影，
每条视线最多需要使用多达 100 个采样点 \[IJTN07]。
我们在此提出一种受阴影体积技术启发的新方法 \[HHLH05]。该方法不依赖数值积分，因此不会出现这些伪影。
我们首先展示一种精确计算的方法，但它不适合在 GPU 上实现。随后我们提出一种更适合 GPU
的近似解法。我们的核心思想是利用预计算的积分  $$ \mathbb S $$ 来计算视线方向上每一个被照亮的线段  $$ [x_i, x_{i+1}] $$ 
所产生的散射光，该光照量由以下表达式给出： $$ T(x, x_i) \mathbb S_|{x_i} - T(x, x_{i+1}) \mathbb S_|{x_{i+1}} $$ 
根据定义，点  $$ x_i $$ 位于地形阴影体积的边界上，因此它们可以通过诸如 \[HHLH05]
所述的阴影体积算法找到。该算法将物体的轮廓边缘（从光源方向看）进行外扩，还会将这些物体投影到近平面上，即使在发生裁剪的情况下也能获得正确结果。

然而，这些算法也会生成许多并不代表光影边界的表面（见图5）。在计算散射光时必须忽略这些错误边界，否则将得到错误的结果。不幸的是，
检测这些错误边界是一个非局部操作，不适合在
GPU 上执行（例如需要多pass或使用列表结构）。


![img_4.png](/assets/images/Precomputed Atmospheric Scattering/img_4.png)

**图5:** 长度  $$ l $$ 的计算。
左图：由于错误的边界  $$ b $$ 和  $$ c $$ ，计算得到的长度  $$ \Delta z - \Delta_n \cdot z_g = z_g - z_a + z_c - z_b $$ 大于实际的  $$ l $$ 
。将该值限制（clamp）为  $$ z_g - z_{\min} $$ 可修复此问题。
右图：视点处于阴影中。若仅使用外扩边缘， $$ x_o $$ 会被误判为受光，且  $$ l $$ 将等于 0，而不是  $$ z_g - z_{\text{near}} $$ 
。将背面（虚线）投影到近裁剪面上可以解决该问题 \[HHLH05]。



我们的方法是使用阴影体算法来计算阴影段的总长度  $$ l $$ 
，并用该长度的单一段替代这些阴影段，在视线射线靠近“地面”的一端（见图5）。虚假边界仍然会带来问题，即对  $$ l $$ 
的高估。然而，在这里  $$ l $$ 可以被限制在阴影体的最近面和最远面之间的距离。这样在大多数情况下能得到正确结果，其余情况下则为近似值。我们的详细算法如下：

我们为每个像素关联四个值： $$ \Delta n $$ 、 $$ \Delta z $$ 、 $$ z_{min} $$ 、 $$ z_{max} $$ ，初始值分别为 0、0、 $$ \infty $$ 
、0。在第一步中，对于每个阴影体表面的前面（反之亦然为后面），我们将  $$ \Delta n $$ 减 1（或加 1），将  $$ \Delta z $$ 
减去（或加上）该片元的深度  $$ z $$ ，同时使用  $$ z $$ 更新  $$ z_{min} $$ 和  $$ z_{max} $$ 。在第二步中我们使用（见图5）：
![img_5.png](/assets/images/Precomputed Atmospheric Scattering/img_5.png)
当看向地面或者天空时
![img_6.png](/assets/images/Precomputed Atmospheric Scattering/img_6.png)

## 6.实现，结果和讨论


![img_7.png](/assets/images/Precomputed Atmospheric Scattering/img_7.png)
**图 7**：验证。
不同太阳天顶角和视角天顶角（视线与太阳方向之间的方位角为零）下，相对于天顶亮度的天空亮度。对比我们的模型（参数为：
 $$ \bar{\alpha} = 0.1 $$ 、
 $$ \beta^s_M = 2.2 \times 10^{-5} \, \text{m}^{-1} $$ 、 $$ \beta^s_M / \beta^e_M = 0.9 $$ 、 $$ g = 0.73 $$ 、 $$ H_M = 1.2 \, \text{km} $$ ）
与 CIE 天空模型 12（基于实际测量数据）。我们注意到在靠近地平线的位置（视角接近 90° 和 -90°）存在亮度高估的现象，
这一点在图 6 中也可以观察到。正如 \[ZWP07] 所示，Preetham 模型 \[PSS99] 同样存在这个问题，这可能源于当前计算机图形中使用的物理模型本身的限制。


![img_8.png](/assets/images/Precomputed Atmospheric Scattering/img_8.png)
**图 8**：结果。
(a) 从上到下分别为：[SFE07]、单次散射、多次散射和照片。使用 [SFE07] 方法时，由于缺少参数 ν，阴影未能显示；仅使用单次散射时画面过暗。
(b) 从太空中观看的日落。
(c) 我们用于性能测试的视角。

![img_9.png](/assets/images/Precomputed Atmospheric Scattering/img_9.png)
**图 9**：结果。
我们的渲染结果（无边框）与网上找到的真实照片（红色边框）进行对比。
色调映射可能解释了某些图像中与未校准照片相比天空色调的差异。



**预计算**
我们在 GPU 上实现了预计算算法，使用片元着色器进行数值积分处理。虽然这并非必须，但它使我们能够快速更改大气参数，并节省磁盘空间（实际上，在
NVidia 8800 GTS 上，5 阶散射可在 5 秒内完成计算）。我们将  $$ T(r, \mu) $$ 和  $$ E(r, \mu) $$ 分别存储在 64×256 和 16×64
的纹理中。我们将  $$ S(u_r, u_\mu, u_{\mu_s}, u_\nu) = [C_*, C_{M,r}] $$ 存储在一个 32×128×32×8 的表中，该表被视为 8 个三维表打包成一个
32×128×256 的 RGBA 纹理（对第 4 个坐标手动进行线性插值）。得益于我们优化的参数化方法，这个 4D 表相比 \[SFE07] 的 3D
表具有更高的精度且占用更少空间（使用 16 位浮点数时，S 表占用 8 MB，而他们的 128³ 纹理则占用 12 MB）。

**渲染**
渲染在4个pass里面完成

* 我们仅将地形绘制到深度缓冲中；
* 我们将地形的阴影体绘制到一个包含  $$ \Delta n $$ 、 $$ \Delta z $$ 、 $$ z_{min} $$ 、 $$ z_{max} $$ 的纹理中。为此，我们使用 **ADD** 和 **MAX**
  混合函数，禁用深度写入，并使用一个 **几何着色器** 来沿着太阳方向拉伸轮廓边缘。该着色器还会将位于该近平面与太阳之间的背面（从太阳视角看）投影到近平面上
  \[HHLH05]；
* 我们使用公式 (17) 和 (18)
  绘制地形和其他物体的空气透视效果，以及天空。如果存在透明物体（如云），则必须在混合之前为每个物体单独计算空气透视。我们使用  $$ \Delta n $$ 
  来计算  $$ R[L_0] $$ 中的遮挡，并使用如上所述计算出的  $$ \tilde{l} $$ 来获得  $$ x_s $$ （见第5节）；
* 最后应用一个全局的色调映射函数。

**结果**
我们使用 NASA Earth Observatory 提供的高程图和反射率纹理进行了多项测试 \[SVS∗05]。测试结果如图 8 和图 9 所示。如图 6 和图
7 所示，我们的模型能够较好地复现 CIE 晴空模型，该模型是根据地面实测数据拟合而成的 \[DK02]
。由于天空颜色和空气透视仅需每像素进行少量纹理采样（<10 次），因此我们的算法运行速度非常快。例如，在图 8 所示的右侧视角下，分辨率为
1024×768 的情况下，在 NVidia 8800 GTS 上可达到 125 fps（不含光柱效果）。这包括 5 ms 用于无阴影地形的渲染，0.4 ms 用于公式 (17)
和 (18) 的前三项，以及 2.6 ms 用于其余项（其中 1 ms 用于评估非线性参数化）。启用光柱后帧率为 25 fps（即前两个渲染通道代价较高，约
32 ms）。相比之下，我们使用每条光线 10 个采样点重新实现 \[O'N05] 方法（达到相同的单次散射质量但不含光柱）时，可达 50 fps。

**局限性**
我们方法的一个局限是气溶胶性质被假设为恒定的，仅依赖于高度，而实际上它们会随着大气条件发生显著变化 \[Slo02]
。尽管我们的预计算非常快速，可以迅速更改这些性质，但这些性质仍然是均匀的。

## 7. 结论

我们提出了首个可实现从任意视角实时渲染天空与空气透视的方法，具备多次散射、地形阴影与光柱效果，并能随视角和太阳角度的变化呈现出正确的视觉结果。
该方法基于最小的简化假设，使我们能够对渲染方程进行近似求解，并预计算其中大多数项。该方法可以轻松扩展到更复杂的物理模型，包括更多的大气成分或更广的波长范围。

未来的工作中，我们希望能对云层对地面照度和空气透视的影响进行建模，从而移除晴空的假设。实际上，在云层较多的情况下，地面与云层之间的相互反射也应被考虑 \[BNL06]
，并且云层对空气透视的影响也不容忽视。据我们所知，目前尚未有相关的研究实现这一点。

[原文源码]（2017年 [更新的源码]）

## 参考文献

[BNL06] BOUTHORS A., NEYRET F., LEFEBVRE S.: Real-time
realistic illumination and shading of stratiform clouds. In Euro
graphics Workshop on Natural Phenomena (sep 2006).

[DK02] DARULA S., KITTLER R.: CIE general sky standard
defining luminance distributions. eSim (2002).

[HHLH05] HORNUS S., HOBEROCK J., LEFEBVRE S., HART
J. C.: ZP+: correct Z-pass stencil shadows. In ACM Sympo
sium on Interactive 3D Graphics and Games (I3D) (April 2005),
ACM, ACMPress.

[HMS05] HABER J., MAGNOR M., SEIDEL H.-P.: Physically
based simulation of twilight phenomena. ACM Trans. Graph. 24,
4 (2005), 1353–1373.

[HP02] HOFFMAN N., PREETHAM A. J.: Rendering outdoor
light scattering in real time. Proceedings of Game Developer
Conference (2002).

[IJTN07] IMAGIRE T., JOHAN H., TAMURA N., NISHITA T.:
Anti-aliased and real-time rendering of scenes with light scat
tering effects. Vis. Comput. 23, 9 (2007), 935–944.

[NDKY96] NISHITA T., DOBASHI Y., KANEDA K., YA
MASHITA H.: Display method of the sky color taking into ac
count multiple scattering. In Proceedings of Pacific Graphics
(1996), pp. 117–132.

[NSTN93] NISHITA T., SIRAI T., TADAMURA K., NAKAMAE
E.: Display of the Earth taking into account atmospheric scatter
ing. In SIGGRAPH 93 (1993), ACM, pp. 175–182.

[O'N05] O'NEIL S.: Accurate atmospheric scattering. In GPU
Gems2:ProgrammingTechniques for High-Performance Graph
ics and General-Purpose Computation (2005), Addison-Wesley
Professional.

[PSS99] PREETHAM A. J., SHIRLEY P., SMITS. B. E.: A prac
tical analytic model for daylight. In SIGGRAPH 99 (1999).

[REK∗04] RILEY K., EBERT D. S., KRAUS M., TESSENDORF
J., HANSEN C. D.: Efficient rendering of atmospheric phenom
ena. In Rendering Techniques (2004), pp. 374–386.

[SFE07] SCHAFHITZEL T., FALK M., ERTL T.: Real-time ren
dering of planets with atmospheres. In WSCG International Con
ference in Central Europe on Computer Graphics, Visualization
and Computer Vision (2007).

[Slo02] SLOUP J.: Asurvey ofthe modelling and rendering of the
Earth's atmosphere. In SCCG '02: Proceedings of the 18th spring
conference on Computer graphics (2002), ACM, pp. 141–150.

[SVS∗05] STOCKLI R., VERMOTE E., SALEOUS N., SIMMON
R., HERRING D.: The Blue Marble Next Generation– a true
color Earth dataset including seasonal dynamics from MODIS.
NASA Earth Observatory (2005).

[TS99] THOMAS G. E., STAMNES K.: Radiative transfer in the
atmosphere and ocean. Cambridge Univ. Press, 1999.

[ZWP07] ZOTTI G., WILKIE A., PURGATHOFER W.: A critical
review of the Preetham skylight model. In WSCG 2007 Short
Communications Proceedings I (Jan. 2007), pp. 23–30

[原文源码]: http://evasion.inrialpes.fr/~Eric.Bruneton/
[更新的源码]:   https://ebruneton.github.io/precomputed_atmospheric_scattering/
[原文]: https://hal.science/inria-00288758/en/