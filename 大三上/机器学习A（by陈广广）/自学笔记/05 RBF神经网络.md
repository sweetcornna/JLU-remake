# RBF 神经网络

整理自 `../机器学习B/笔记/机器学习笔记.pdf` 第 58–70 页（第 5 章径向基神经网络），并参考 `../机器学习B/笔记/机器学习期末复习.pdf` 第 25 页。A 的手写笔记没有单独整理这一章，但 A、B 的考试知识点都列了它。自组织映射（SOM）网络在 B 笔记里放在聚类一章，见 [07 聚类与EM算法](<07 聚类与EM算法.md>)。

## 本章要点

- 考试知识点原文：（正则化）RBF 神经网络的结构及插值描述、广义 RBF 神经网络结构、学习方法。
- RBF 网络固定三层：输入层只传递信号；隐层用**径向基函数**（通常是高斯函数）做非线性变换，具有局部特性；输出层对隐层输出**线性加权**。
- 插值描述：$P$ 个样本各放一个基函数，解线性方程组 $\Phi W=d$ 得到权值。这种隐节点数等于样本数的网络就是正则化 RBF 网络。
- 广义 RBF 网络：隐节点数 $M$ 远小于样本数 $P$，中心不必落在样本上，各基函数宽度可以不同，输出带阈值。
- 学习方法要记三类参数怎么定：**中心**（从样本选、k-means/SOM 聚类、监督学习）、**扩展常数**（$\delta=d_{max}/\sqrt{2M}$ 或按最近中心距离取）、**输出权值**（LMS 或伪逆）。
- 和多层感知器比：RBF 用超球体划分、局部逼近、学得快、一般只有一个隐层；MLP 用超平面划分、全局逼近。

## 概念与解释

### 为什么要有 RBF 网络

多层前馈网络（BP 网络）用在信号处理、模式识别等场合，学习算法基于反向传播。它的缺点是计算量大、学习速度慢，而且依赖非线性优化技术。

径向基函数网络给多层前馈网络的学习提供了另一种思路，特点是推广能力好、计算量少、学习速度快。

神经网络的三类典型应用：

| 任务 | 形式 |
| --- | --- |
| 分类 | $l=f(\mathbf x)$，$\mathbf x\in X\subset\mathbb R^m$，$l\in C\subset\mathbb N$ |
| 函数逼近（回归） | $\mathbf y=f(\mathbf x)$，$\mathbf x\in\mathbb R^n$，$\mathbf y\in\mathbb R^m$ |
| 时间序列分析 | $\mathbf x(t)=f(\mathbf x_{t-1},\mathbf x_{t-2},\mathbf x_{t-3},\dots)$ |

用于函数逼近时，网络输出是基函数的线性组合 $f(\mathbf x)=\sum_{i=1}^m w_i\phi_i(\mathbf x)$，隐单元起分解、特征提取、变换的作用。

### 径向基函数

**定义**：取值只依赖于输入到原点的距离的函数，$\Phi(\mathbf x)=\Phi(\lVert\mathbf x\rVert)$；更一般地，依赖于到某个中心点 $\mathbf c$ 的距离，$\Phi(\mathbf x,\mathbf c)=\Phi(\lVert\mathbf x-\mathbf c\rVert)$。距离通常用欧氏距离（标准形式叫欧氏径向基函数），也可以用别的距离。

径向基函数 $\Phi(\lVert\mathbf x-\mathbf c\rVert)$ 有三个要素：中心 $\mathbf c$、距离度量 $r=\lVert\mathbf x-\mathbf c\rVert$、形状 $\phi$。

**高斯函数**最常用：

$$
\phi(\lVert\mathbf x-\mathbf x_c\rVert)=\exp\Big(-\frac{\lVert\mathbf x-\mathbf x_c\rVert^2}{2\sigma^2}\Big)
$$

$\mathbf x_c$ 是中心，$\sigma$ 是宽度参数，控制函数的径向作用范围：$\sigma$ 越小，函数越"尖"，只对离中心很近的输入有明显响应。

常见的径向基函数（$r=\lVert\mathbf x-\mathbf c\rVert$）。课件有两种写法，一种用形状参数 $\varepsilon$，一种用宽度 $\delta$：

| 名称 | 用 $\varepsilon$ 写 | 用 $\delta$ 写 |
| --- | --- | --- |
| 高斯函数 | $\phi(r)=e^{-(\varepsilon r)^2}$ | $\phi(r)=\exp\big(-\frac{r^2}{2\delta^2}\big)$ |
| 多二次函数（multiquadric） | $\phi(r)=\sqrt{1+(\varepsilon r)^2}$ | |
| 逆二次函数（inverse quadratic） | $\phi(r)=\frac{1}{1+(\varepsilon r)^2}$ | |
| 逆多二次函数（inverse multiquadric） | $\phi(r)=\frac{1}{\sqrt{1+(\varepsilon r)^2}}$ | $\phi(r)=\frac{1}{(r^2+\delta^2)^{1/2}}$ |
| 反演 S 型函数（reflected sigmoidal） | | $\phi(r)=\frac{1}{1+\exp(r^2/\delta^2)}$ |

（更正：原文反演 S 型函数写作 $\frac{1}{1+\exp(-r^2/\delta^2)}$，这样函数随 $r$ 增大而增大，没有"反演"的意思；应为 $\exp(+r^2/\delta^2)$，在 $r=0$ 处取最大值 1/2，随距离增大衰减到 0。）

$\varepsilon$ 越大，高斯函数衰减越快（课件图中 $\varepsilon=0.1$ 几乎是水平线，$\varepsilon=10$ 是一根尖峰）。

### RBF 网络的结构

RBF 网络是以径向基函数作为激活函数的人工神经网络，输出是输入经径向基函数变换后的线性组合，能以任意精度逼近任意连续函数。应用于非线性函数逼近、时间序列分析、分类。

拓扑结构只有**一个隐层**：

1. **输入层**：由信号源节点组成，只传递数据，不做变换。
2. **隐含层**：节点数按需要设定，激活函数是径向基函数（如高斯函数），对输入做空间映射变换。
3. **输出层**：激活函数是线性函数，对隐层输出线性加权后作为网络输出。

隐层的**局部特性**：神经元的激活程度随输入离中心的距离增大而降低，所以 RBF 网络属于局部逼近网络。

同一结构的两种用法：

- 函数逼近：$f(\mathbf x)=\sum_{m}w_m\phi(\lVert\mathbf x-\mathbf x_m\rVert)$，输出单元线性，做插值；隐单元做投影。
- 分类：$f(\mathbf x)=\sum_m w_m\phi(\lVert\mathbf x-\mathbf c_m\rVert)$，输出单元可以接 S 型函数做分类，隐单元对应"子类"。

### RBF 网络和经典神经网络的区别

作业题考过隐藏层的区别：

| | 经典神经网络（MLP） | RBF 网络 |
| --- | --- | --- |
| 隐层计算 | 内积 + tanh（inner-product + tanh） | 距离 + 高斯（distance + Gaussian） |
| 隐层性质 | 全局特性 | 局部特性 |
| 第一层参数的含义 | 权值 $w_{ij}^{(1)}$ | 中心（centers） |
| 输出层 | 线性聚合 | 线性聚合（相同） |

### 插值问题与正则化 RBF 网络

**插值问题**：在 $N$ 维空间到一维空间的映射中，给定 $P$ 个输入向量 $\mathbf x_p$ 和目标值 $d_p$，找一个非线性映射 $F(\mathbf x)$ 满足插值条件

$$
F(\mathbf x_p)=d_p,\quad p=1,2,\dots,P
$$

插值通俗说就是：已知一组数据点，要能预测这些点之间未知位置的值。

**用径向基函数解插值问题**：

1. 选 $P$ 个基函数，每个对应一个训练样本，形式为 $\phi(\lVert\mathbf x-\mathbf x_p\rVert)$。自变量是 $\mathbf x$ 到中心 $\mathbf x_p$ 的距离，距离径向对称，所以叫径向基函数。
2. 插值函数取基函数的线性组合：$F(\mathbf x)=\sum_{p=1}^P w_p\,\phi(\lVert\mathbf x-\mathbf x_p\rVert)$。
3. 代入插值条件，得到关于 $w_p$ 的 $P$ 阶线性方程组：$\sum_{p=1}^P w_p\,\phi(\lVert\mathbf x_i-\mathbf x_p\rVert)=d_i$，$i=1,\dots,P$。
4. 记 $\phi_{ip}=\phi(\lVert\mathbf x_i-\mathbf x_p\rVert)$，写成矩阵形式：

$$
\begin{pmatrix}\phi_{11}&\phi_{12}&\cdots&\phi_{1P}\\ \phi_{21}&\phi_{22}&\cdots&\phi_{2P}\\ \vdots&\vdots&&\vdots\\ \phi_{P1}&\phi_{P2}&\cdots&\phi_{PP}\end{pmatrix}\begin{pmatrix}w_1\\w_2\\ \vdots\\w_P\end{pmatrix}=\begin{pmatrix}d_1\\d_2\\ \vdots\\d_P\end{pmatrix},\qquad \Phi W=\mathbf d
$$

$\Phi$ 叫插值矩阵，可逆时 $W=\Phi^{-1}\mathbf d$。

**Micchelli 定理**：对一大类径向基函数（包括高斯函数、逆多二次函数），只要输入点 $\mathbf x_1,\mathbf x_2,\dots,\mathbf x_P$ 各不相同，插值矩阵 $\Phi$ 就可逆。（更正：原文写作"如果 $\phi_1,\dots,\phi_P$ 各不相同"，定理的条件是样本点互不相同。）

把上面的插值过程画成网络，就是隐节点数等于样本数 $P$、每个隐节点的中心就是一个样本的 RBF 网络，即**正则化 RBF 网络**。作业题"RBF 神经网络中隐藏层节点数"答案是"等于样本数"，说的就是它。

**完全内插的问题**：

1. 泛化能力差：插值曲面必须穿过所有训练点，数据有噪声时会把噪声也拟合进去。
2. 超定问题：训练样本数远大于系统固有自由度时，问题是超定的，插值矩阵求逆容易不稳定。

解决办法是加正则化项限制模型复杂度（或用交叉验证选复杂度），或者改用隐节点更少的广义 RBF 网络。（更正：原文说正则化 RBF 网络和广义 RBF 网络都"通过减少隐神经元的数量"避免完全内插的问题。正则化 RBF 网络的隐节点数仍等于样本数，它靠正则化项改善泛化；减少隐节点数的是广义 RBF 网络。）

整理补充：按正则化理论，在误差平方和上加光滑性惩罚 $\lambda$ 后，解的形式不变，权值改由 $(\Phi+\lambda I)W=\mathbf d$ 求得；$\lambda=0$ 就退回严格插值。

### 广义 RBF 网络

和正则化 RBF 网络比，广义 RBF 网络有四点不同：

1. **基函数数目**：$M$ 与样本数 $P$ 不同，通常 $M\ll P$。作业题"广义 RBF 神经网络中隐藏层节点数"答案是"远远小于样本数"。
2. **中心位置**：不要求落在训练样本上，在训练中确定。
3. **扩展常数**（宽度）：各基函数的扩展常数不再统一，可以分别确定。
4. **输出函数**：仍是线性的，但加了阈值参数（图中的 $\varphi_0$ 节点，权值 $T$），用来补偿基函数在样本集上的平均值与目标值平均值之间的差。

网络输出：$f(\mathbf x)=\sum_{m=1}^M w_m\,\phi(\lVert\mathbf x-\mathbf c_m\rVert)$（加阈值项），$\phi(r)=\exp\big(-\frac{r^2}{2\delta^2}\big)$，中心 $\mathbf c_m$ 和宽度 $\delta$ 都是要学的参数。

### RBF 网络的非线性映射

- 设一个隐层，隐节点激活函数 $\phi(\mathbf x)$，隐节点数 $M$ 大于输入节点数 $N$，相当于把输入映射到更高维的隐空间。
- 若 $M$ 足够大，样本在隐空间里就是线性可分的。
- 隐层到输出层可以用和感知器类似的线性可分算法来学。

**例：用两个隐节点解决异或问题。** 取

$$
\phi_1(X)=\exp(-\lVert X-C_1\rVert^2),\ C_1=(1,1);\qquad \phi_2(X)=\exp(-\lVert X-C_2\rVert^2),\ C_2=(0,0)
$$

| 输入 | 类别 | $(\phi_1,\phi_2)$ | $\phi_1+\phi_2$ |
| --- | --- | --- | --- |
| (0,0) | 0 | (0.1353, 1.0000) | 1.1353 |
| (1,1) | 0 | (1.0000, 0.1353) | 1.1353 |
| (0,1) | 1 | (0.3679, 0.3679) | 0.7358 |
| (1,0) | 1 | (0.3679, 0.3679) | 0.7358 |

原空间里异或线性不可分；映射后 (0,1) 和 (1,0) 落到同一点，一条直线（比如 $\phi_1+\phi_2=0.9$）就能把两类分开。这个例子只有 2 个隐节点，映射后仍是二维，但已经可分；隐节点更多时就是映射到更高维空间。

### 广义 RBF 网络的训练

**结构设计**：主要靠经验，定隐层节点数和输出层节点数，隐节点数一般小于样本数（$M<P$）。

**参数设计**要定三组参数：

1. 各基函数的中心：决定网络的映射特性。
2. 扩展常数：控制基函数的宽度，也就是非线性的程度。
3. 输出节点的权值：决定隐层到输出层的线性映射，一般用有监督算法（最小二乘或梯度下降）确定。

#### 中心和扩展常数的三种确定方法

1. **从样本中选取中心**：数据密集处多选、稀疏处少选；数据均匀分布时中心也均匀分布。扩展常数按中心间最大距离 $d_{max}$ 和中心数目 $M$ 取，例如 $\delta=\dfrac{d_{max}}{\sqrt{2M}}$。例：$d_{max}=4$，$M=8$ 时 $\delta=4/\sqrt{16}=1$。
2. **自组织选择中心**（k-means 聚类、SOM）：用聚类自动确定中心位置。
3. **监督学习选中心**：结合输出目标，用梯度下降同时学中心、宽度和权值。

#### 方法 2：k-means 聚类确定中心

先估计中心数目 $M$，记 $c_j(k)$ 为第 $k$ 次迭代时第 $j$ 个中心（$k$ 是迭代次数，$j$ 才是类别编号）。

1. 初始化：随机取 $c_1(0),c_2(0),\dots,c_M(0)$。
2. 对每个样本 $X_p$ 计算到各中心的欧氏距离 $\lVert X_p-c_j(k)\rVert$，$p=1,\dots,P$，$j=1,\dots,M$。
3. 相似匹配：找最近的中心 $j^*$，$\lVert X_p-c_{j^*}(k)\rVert=\min_j\lVert X_p-c_j(k)\rVert$，把 $X_p$ 归入第 $j^*$ 类。
4. 更新中心，两种做法：
   - 均值法（k-means）：$c_j(k+1)=\dfrac{1}{N_j}\sum_{X\in U_j(k)}X$，$U_j(k)$ 是第 $j$ 类的样本集合，$N_j$ 是其样本数。
   - 竞争学习（SOM 的做法）：只移动获胜中心，

$$
c_j(k+1)=\begin{cases}c_j(k)+\eta\,[X_p-c_j(k)], & j=j^*\\ c_j(k), & j\ne j^*\end{cases}
$$

   获胜中心向当前样本移动一步，步长由学习率 $\eta$ 和样本到中心的距离决定；其他中心不动。
5. $k$ 加 1；若中心的变化量没有小于阈值，回到第 2 步。

扩展常数：记 $d_j=\min_i\lVert c_j-c_i\rVert$ 为中心 $c_j$ 到最近的其他中心的距离，取 $\delta_j=\lambda d_j$，$\lambda$ 是缩放因子。

#### 输出层权值

1. **最小均方（LMS）算法**：和感知器类似，按均方误差迭代调整权值。
2. **伪逆法**：令 $\varphi_{pj}=\varphi(\lVert X_p-c_j\rVert)$，隐层输出矩阵 $\hat\Phi=(\varphi_{pj})_{P\times M}$，权向量 $W=(w_1,\dots,w_M)^T$。由 $\hat\Phi W=\mathbf d$（方程数多于未知数）得最小二乘解

$$
W=\hat\Phi^{+}\mathbf d,\qquad \hat\Phi^{+}=(\hat\Phi^T\hat\Phi)^{-1}\hat\Phi^T
$$

$\hat\Phi^+$ 是 $\hat\Phi$ 的伪逆。推导和线性回归的正规方程一样，见 [02 线性回归](<02 线性回归.md>)。

#### 方法 3：监督学习（梯度下降）同时学中心、宽度、权值

单输出，基函数 $G(\lVert X-c_j\rVert)=\exp\big(-\frac{\lVert X-c_j\rVert^2}{2\delta_j^2}\big)$，误差函数

$$
E=\frac12\sum_{i=1}^P e_i^2=\frac12\sum_{i=1}^P\big(d_i-F(X_i)\big)^2=\frac12\sum_{i=1}^P\Big(d_i-\sum_{j=1}^M w_jG(\lVert X_i-c_j\rVert)\Big)^2
$$

（更正：原文最后一项写作 $G(\lVert X_i-c_i\rVert)$，中心的下标应为 $j$。）

$e_i=d_i-F(X_i)$ 是第 $i$ 个样本的误差，$d_i$ 是目标值，$w_j$、$c_j$、$\delta_j$ 是第 $j$ 个基函数的权值、中心、宽度，$P$ 是样本数，$M$ 是基函数数。

推导要点：$\dfrac{\partial E}{\partial w_j}=-\sum_i e_iG_{ij}$；由 $\dfrac{\partial G}{\partial c_j}=G\cdot\dfrac{X-c_j}{\delta_j^2}$、$\dfrac{\partial G}{\partial \delta_j}=G\cdot\dfrac{\lVert X-c_j\rVert^2}{\delta_j^3}$，再乘上 $\dfrac{\partial E}{\partial G_{ij}}=-e_iw_j$。沿负梯度更新：

$$
\Delta c_j=-\eta\frac{\partial E}{\partial c_j}=\eta\frac{w_j}{\delta_j^2}\sum_{i=1}^P e_i\,G(\lVert X_i-c_j\rVert)\,(X_i-c_j)
$$

$$
\Delta\delta_j=-\eta\frac{\partial E}{\partial \delta_j}=\eta\frac{w_j}{\delta_j^3}\sum_{i=1}^P e_i\,G(\lVert X_i-c_j\rVert)\,\lVert X_i-c_j\rVert^2
$$

$$
\Delta w_j=-\eta\frac{\partial E}{\partial w_j}=\eta\sum_{i=1}^P e_i\,G(\lVert X_i-c_j\rVert)
$$

三个偏导数都用 sympy 核对过。**每个数据修正一次**的版本（相当于 SGD）把误差函数换成单个样本的 $E=\frac12e^2$，公式去掉 $\sum_i$ 即可：

$$
\Delta c_j=\eta\frac{w_j}{\delta_j^2}\,e\,G(\lVert X-c_j\rVert)(X-c_j),\quad \Delta\delta_j=\eta\frac{w_j}{\delta_j^3}\,e\,G(\lVert X-c_j\rVert)\lVert X-c_j\rVert^2,\quad \Delta w_j=\eta\,e\,G(\lVert X-c_j\rVert)
$$

### RBF 网络用于分类

- 每个 RBF 神经元存一个"代表"向量（可以就是训练集里的一个样本）。
- 对新输入，每个神经元算输入到自己代表的欧氏距离，输出一个 0 到 1 之间的相似度。选相似度函数就是选激活函数。
- 粗略地说，输入离 A 类代表比离 B 类代表更近，就判为 A 类。

### 应用实例：用 RBF 网络拟合 Hermite 多项式

B 笔记给了一段 numpy 实现：拟合 $y=1.1(1-x+2x^2)e^{-x^2/2}$，$x$ 在 $[-5,5]$ 上取 500 个点，50 个隐节点，权值、中心、宽度的学习率分别是 0.1、0.2、0.1，批量梯度下降训练 1000 轮。第 0 轮输出曲线和目标差得很远，第 100 轮大致形状出来但有毛刺，第 500 轮基本重合。

下面是按原代码思路整理的版本。原代码有一行在 PDF 里被截断，按上下文补全；另外原代码求中心梯度时先用 `np.dot(hi_output[j], X[j]-c)` 把所有中心的项加成一个向量，再和 $w_j/\sigma_j^2$ 做外积，结果每个中心拿到的是所有中心混在一起的梯度，和上面的 $\Delta c_j$ 公式不符。（更正：中心梯度应逐个中心计算 $\sum_i e_i\,G_{ij}\,(X_i-c_j)$。）两种写法都能收敛，是因为权值和宽度也在同时调整；用同一随机种子各跑 1000 轮，误差平方和的一半分别降到约 0.115（原写法）和 0.059（改正后）。

```python
import numpy as np

class RBFNetwork:
    def __init__(self, hidden_nums, r_w, r_c, r_sigma, seed=0):
        self.h = hidden_nums
        self.r = {"w": r_w, "c": r_c, "sigma": r_sigma}   # 三组参数各自的学习率
        self.rng = np.random.default_rng(seed)

    def train(self, X, y, iters):
        y = y.reshape(-1, 1)
        m, n = X.shape
        sigma = self.rng.random((self.h, 1))     # 宽度 (h,1)
        c = self.rng.random((self.h, n))         # 中心 (h,n)
        w = self.rng.random((self.h + 1, 1))     # 权值，最后一个是阈值 (h+1,1)
        errs = []
        for _ in range(iters):
            # 正向计算
            diff = X[:, None, :] - c[None, :, :]                 # (m,h,n)
            dist2 = (diff ** 2).sum(axis=2)                      # (m,h)
            hi = np.exp(-dist2 / (2 * sigma.T ** 2))             # 隐层输出 (m,h)
            yi_in = np.hstack([hi, np.ones((m, 1))])             # 加截距列 (m,h+1)
            y_out = yi_in @ w                                    # (m,1)
            errs.append(0.5 * np.linalg.norm(y_out - y) ** 2)
            # 反向：g = y_out - y = -e
            g = y_out - y
            dw = yi_in.T @ g
            dsigma = ((hi * dist2).T @ g) * w[:-1] / sigma ** 3
            dc = (w[:-1] / sigma ** 2) * np.einsum('j,ji,jin->in', g[:, 0], hi, diff)
            w -= self.r["w"] * dw / m
            sigma -= self.r["sigma"] * dsigma / m
            c -= self.r["c"] * dc / m
        self.w, self.c, self.sigma = w, c, sigma
        return errs

if __name__ == "__main__":
    X = np.linspace(-5, 5, 500)[:, None]
    y = 1.1 * (1 - X[:, 0] + 2 * X[:, 0] ** 2) * np.exp(-0.5 * X[:, 0] ** 2)
    errs = RBFNetwork(50, 0.1, 0.2, 0.1, seed=1).train(X, y, 1000)
    print(errs[0], errs[100], errs[500], errs[-1])
```

## 易混对比

### 正则化 RBF 网络与广义 RBF 网络

| | 正则化 RBF 网络 | 广义 RBF 网络 |
| --- | --- | --- |
| 隐节点数 | 等于样本数 $P$ | $M\ll P$ |
| 中心 | 就是全部训练样本 | 需要确定（样本中选、聚类、监督学习） |
| 扩展常数 | 统一 | 各基函数可以不同 |
| 输出阈值 | 无 | 有 |
| 权值求法 | 解 $\Phi W=\mathbf d$（或加正则项） | LMS 或伪逆 $W=\hat\Phi^+\mathbf d$ |
| 问题 | 样本多时矩阵大、易过拟合噪声 | 中心和宽度要额外确定 |

### RBF 网络与多层感知器（MLP）

| 方面 | 多层感知器 | RBF 网络 |
| --- | --- | --- |
| 分类方式 | 超平面划分输入空间 | 超球体划分，中心是基函数中心，半径由宽度决定 |
| 学习方式 | 全局学习，所有权值一起调，训练较慢（深层网络更明显） | 局部学习，每个基函数只管附近的数据，训练快，对新数据响应快 |
| 网络结构 | 可以有多个隐层 | 通常只有一个隐层，但往往需要更多隐节点来覆盖输入空间，输入维数高时容易遇到维数灾难 |
| 隐层和输出层 | 分类时隐层、输出层都非线性；回归时输出层常用线性 | 隐层非线性，输出层通常线性（也可以接非线性） |
| 求解 | 难有解析解，靠梯度下降、Adam 等迭代 | 中心和宽度定了之后输出层是线性问题，可直接用线性代数求解 |

### RBF 网络与 SVM（整理补充，供论述题参考）

B 期末复习第 25 页有一道论述题"分别用 SVM 和 RBF 网络完成函数回归或模式分类之一，结合实践阐述二者在原理、效果上的异同"，原稿没有答案。下面是可以用的角度：

| 方面 | RBF 网络 | 高斯核 SVM |
| --- | --- | --- |
| 相同点 | 决策函数都是 $\sum_i w_i\exp(-\lVert\mathbf x-\mathbf c_i\rVert^2/2\sigma^2)+b$ 的形式，都靠高斯函数做非线性映射 | 同左 |
| 中心怎么来 | 人为指定数目，再用聚类或梯度下降确定 | 由二次规划自动选出，中心就是支持向量，个数由数据和 $C$ 决定 |
| 优化目标 | 经验风险（误差平方和） | 结构风险（间隔最大化加松弛惩罚） |
| 最优性 | 同时学中心和宽度时非凸，可能陷入局部极小 | 凸二次规划，全局最优 |
| 需要调的超参数 | 隐节点数、宽度、学习率 | 核宽度、惩罚参数 $C$（回归时还有 $\varepsilon$） |
| 小样本时 | 容易过拟合 | 泛化通常更稳 |

SVM 的内容见 [06 支持向量机](<06 支持向量机.md>)。

## 典型题

### 例 1：严格插值（整理补充练习）

一维样本 $x=0,1,2$，目标 $d=1,3,2$，每个样本放一个高斯基函数，$\sigma=1$。求权值。

思路：写插值矩阵 $\phi_{ip}=\exp(-(x_i-x_p)^2/2)$，解 $\Phi W=\mathbf d$。

$$
\Phi=\begin{pmatrix}1&0.6065&0.1353\\0.6065&1&0.6065\\0.1353&0.6065&1\end{pmatrix}
$$

解得 $W\approx(-1.3781,\ 3.9702,\ -0.2216)^T$，代回 $\Phi W=(1,3,2)^T$。插值函数 $F(x)=\sum_p w_p e^{-(x-x_p)^2/2}$ 在样本点之间给出预测，比如 $F(0.5)\approx2.2156$，$F(1.5)\approx2.8608$。

### 例 2：伪逆法求广义 RBF 网络输出权值（整理补充练习）

样本 $x=0,1,2,3,4$，$d=0,1,1.5,1,0$；取 2 个中心 $c_1=1$、$c_2=3$，$\sigma=1$，输出带阈值。

思路：隐层输出矩阵 $\hat\Phi$ 是 $5\times3$（两列高斯输出加一列 1），用 $W=(\hat\Phi^T\hat\Phi)^{-1}\hat\Phi^T\mathbf d$。

结果 $W\approx(2.1943,\ 2.1943,\ -1.3710)^T$（两个中心对称，权值相同），拟合值约为 $(-0.016,\ 1.120,\ 1.291,\ 1.120,\ -0.016)$。

### 例 3：概念选择题

作业中的两道单选（隐节点数等于样本数、远小于样本数）和"RBF 网络与常用神经网络的隐藏层有什么区别"简答，见 [10 简答题精编](<10 简答题精编.md>)。
