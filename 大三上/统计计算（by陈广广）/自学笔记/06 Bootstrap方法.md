# Bootstrap 方法

整理自 `统计计算笔记（压缩）.pdf` 第 51–58 页（3.7 节）。课件的定义和例题原文在 `../重制版PPT/第3章 随机模拟（下）（重制版）.pptx` 里，这份笔记写手写批注那部分：四步流程怎么背、最小二乘那两条性质的完整证明、几道例题的数怎么算出来、百分位法和 bootstrap-t 法的下标到底怎么取。

p52 的最小二乘无偏性与协方差阵证明是整节唯一一处从头写到尾的手写推导，旁边打了两个五角星。p58 的血型例题是一道典型的极大似然大题。

## 3.7.1 Bootstrap 方法引入（原笔记 p51）

### Jackknife 与 Bootstrap 的对比

课件那张图把两种重抽样方法画在一起：

- **Jackknife**：从总体里取一个样本，每次剔掉一个观测，得到一个估计量 $\hat\theta$。
- **Bootstrap**：从样本里反复抽出子样本，每个子样本给出一个估计量 $\hat\theta_1^*,\hat\theta_2^*,\dots$。

图上有一条虚线从「总体」指向「样本」，意思就是 Bootstrap 把**样本当成总体**来抽。这是整个方法的出发点。

### Bootstrap 的四个特点（右上角手写，考概念就答这四条）

1. **样本要能够反映总体**。
2. **有放回的重采样，采样个数和原始样本一致**。
3. 前面的抽样不影响后面的。
4. 可用于人们对总体知之甚少的情况。

第 2 条是做题时最容易漏的：每个 bootstrap 样本的容量必须还是 $n$，不能抽少了。第 3 条说的是有放回抽样的独立性。

### 课件的三段话

- 在对观测值重复抽样，每次用重抽样数据生成一个**经验分布函数**。
- 对每个重抽样数据集或者等价地经验分布函数，可以计算统计量的一个新的取值，收集这些值可以给出受关注的统计量的**抽样分布**的一个估计。
- Bootstrap 用在**独立同分布**的数据上，也可用在不独立的数据上，比如回归残差或者时间序列数据（需要先处理数据）。手写补了一句：**使数据变得相对独立同分布**。

### 记号约定

设总体 $X$ 服从某个未知分布 $F(x)$，$\boldsymbol X=(x_1,\dots,x_n)$ 是 $X$ 的一个样本，$\phi$ 是 $F$ 的一个参数（可以是标准误差、置信区间或者 $p$ 值）。把 $\phi$ 看成 $F$ 的一个**泛函** $\phi(F)$，用统计量 $\hat\phi=g(\boldsymbol X)$ 估计 $\phi$。

再设 $\psi=\psi(g,F,n)$ 是统计量 $\hat\phi$ 的某种分布特征（$\hat\phi$ 的抽样分布的数字特征）。例如

$$\psi=\sqrt{\operatorname{Var}(\bar X)}\ \ (\text{统计量 }\bar X\text{ 的标准误差}),\qquad \psi=E\hat\phi-\phi\ \ (\text{统计量 }\hat\phi\text{ 的偏差})$$

手写在 $\phi$ 旁边举例「$\phi$ 是期望」，在 $\hat\phi=g(\boldsymbol X)$ 旁边举例「$\hat\phi$ 是样本均值」，在 $\psi$ 旁边写「要估计样本均值的标准误差」。三层套娃很容易绕晕，按这个对照看：$\phi$ 是要估的参数，$\hat\phi$ 是估计量，$\psi$ 是这个估计量本身的精度指标。

最后一句「可以用随机模拟的方法估计 $\psi$」被标了两句话：**把统计量看成随机变量**，**相当于在估计统计量的统计量**。这句话是整节的纲。

### 四步流程（手写编号，背这个）

1. **获取原始样本**：从样本估计总体分布 $F$ 为 $\hat F$。
2. **有放回的重抽样，获取 Bootstrap 样本**：从 $\hat F$ 抽取 $B$ 个独立样本 $Y^{(b)}$，$b=1,\dots,B$，每个 $Y^{(b)}$ 样本容量为 $n$，$Y^{(b)}$ 为 Bootstrap 样本。
3. **获取每个 Bootstrap 样本特征的估计值**：从每个 bootstrap 样本 $Y^{(b)}$ 估计得到 $\hat\phi^{(b)}=g(Y^{(b)})$，$b=1,\dots,B$。
4. **估计原始样本的分布特征**：$\hat\phi^{(b)}$（$b=1,\dots,B$）是 $g(Y)$ 在 $\hat F$ 下的**独立同分布样本**，可以用标准的估计方法估计关于 $g(Y)$ 在 $\hat F$ 下的分布特征 $\hat\psi=\psi(g,\hat F,n)$，估计结果记作 $\hat\psi$，并以 $\hat\psi$ 作为统计量 $\hat\phi$ 的抽样分布的数字特征 $\psi(g,F,n)$ 的估计值。

课件在旁边补了一条选择说明：从样本 $\boldsymbol X$ 估计 $\hat F$ 时，可以采用参数模型，也可以采用经验分布函数 $F_n$。**参数模型在模型正确时效率较高；经验分布法使用简单，基本不依赖于模型。** 从经验分布 $F_n$ 抽样，相当于从 $\boldsymbol X=(x_1,\dots,x_n)$ 独立有放回抽样。

这两条路就是 3.7.2 非参数 Bootstrap 和 3.7.3 参数 Bootstrap 的分界。

## 3.7.2 非参数 Bootstrap：估计标准误差（原笔记 p52）

### 问题描述（右上角手写，答题开头照抄）

设总体 $X\sim F(x,\phi)$，$\phi\in\Theta$，$\phi$ 是一个总体的参数；$\hat\phi=g(\boldsymbol X)$ 是总体参数 $\phi$ 的估计量；分布 $F$ 未知，$x_1,\dots,x_n$ 是来自 $F$ 的样本。求 $\hat\phi$ 的标准误差 $SE$ 的 Bootstrap 估计。

标准误差的定义：

$$SE=\sqrt{\operatorname{Var}(\hat\phi)}$$

红笔在旁边标了「**是估计量的**」，提醒 $SE$ 描述的是估计量的波动，不是总体的波动。

### 为什么可以拿 $\hat F_n$ 代替 $F$

手写把这条逻辑写全了：

- 设 $\hat F_n$ 是相应的经验分布函数，当 $n$ 很大时，$\hat F_n$ 接近 $F$。
- 用 $\hat F_n$ 代替 $F$，在 $\hat F_n$ 中抽样。
- 在 $\hat F_n$ 中抽样，就是在原始样本 $x_1,x_2,\dots,x_n$ 中每次随机地抽取一个个体作有放回抽样。
- 如此抽取 $B$ 个独立样本 $Y^{(b)}$，$b=1,\dots,B$，每一个 $Y^{(b)}$ 样本容量为 $n$，称 $Y^{(b)}$ 为 Bootstrap 样本。
- 从每个 bootstrap 样本 $Y^{(b)}$ 估计得到 $\hat\phi^{(b)}=g(Y^{(b)})$，这些 $\hat\phi^{(b)}$ 是 $g(Y)$ 在 $\hat F$ 下的独立同分布样本。

### 例：点参数估计的 SE

总体 $X\sim F(x)$，来自总体的样本 $X_j$（$j=1,\dots,n$）。用样本平均值

$$\hat\phi=\bar X=\frac1n\sum_{i=1}^n X_i$$

作为期望 $\phi=EX$ 的点估计，用它估计标准误差（$S^2$ 为样本方差）：

$$SE(\bar X)=\sqrt{\frac{\operatorname{Var}(X)}{n}}$$

课件接着说：根据中心极限定理和强大数律，当样本量 $n$ 较大时可以取 $EX$ 的近似 95% 置信区间为 $\bar X\pm 2\,SE(\bar X)$。这里的 2 是 1.96 取整，要知道它是哪来的。

### 例：线性模型参数估计的 SE

模型

$$Y=X\beta+\varepsilon,\qquad \varepsilon\sim N(0,\sigma^2 I_n),\ \sigma^2\ \text{未知}$$

$X$ 是已知的 $n\times p$ 数值矩阵，$n>p$；$\beta$ 是未知的系数向量。手写在 $\varepsilon$ 旁边标了「**不含任何 $X$ 的相关信息**」，这句话是后面两条证明里 $E\varepsilon=0$ 能提出来的依据。

$X$ 列满秩时，$\beta$ 的**最小二乘估计**为

$$\hat\beta=(X^{\mathsf T}X)^{-1}X^{\mathsf T}Y$$

红箭头注解：最小化预测值与实际值之间的平方和。

### 证明一：$\hat\beta$ 是唯一一个最优线性无偏估计（手写，打了五角星）

$$E\hat\beta=E\big[(X^{\mathsf T}X)^{-1}X^{\mathsf T}Y\big]$$

$$=E\big[(X^{\mathsf T}X)^{-1}X^{\mathsf T}(X\beta+\varepsilon)\big]$$

$$=E\big[(X^{\mathsf T}X)^{-1}X^{\mathsf T}X\beta\big]+E\big[(X^{\mathsf T}X)^{-1}X^{\mathsf T}\varepsilon\big]$$

$$=E\beta+(X^{\mathsf T}X)^{-1}X^{\mathsf T}E\varepsilon$$

$$=\beta$$

倒数第二步下面用蓝笔画波浪线标了 $E\varepsilon=\mathbf 0$，旁边写着理由：**$X$ 和 $\varepsilon$ 无关，可提取系数**，以及 $\varepsilon\sim N(0,\sigma^2 I_n)$，$E\varepsilon=0$。

三个关键动作：把 $Y$ 换成 $X\beta+\varepsilon$；$(X^{\mathsf T}X)^{-1}X^{\mathsf T}X=I$ 约掉；$X$ 是常数矩阵可以提到期望外面。

### 证明二：$\hat\beta$ 的协方差阵为 $\operatorname{Var}(\hat\beta)=\sigma^2(X^{\mathsf T}X)^{-1}$（手写，打了五角星）

先算 $\hat\beta-\beta$：

$$\hat\beta-\beta=(X^{\mathsf T}X)^{-1}X^{\mathsf T}Y-\beta$$

$$=(X^{\mathsf T}X)^{-1}X^{\mathsf T}(X\beta+\varepsilon)-\beta$$

$$=\beta+(X^{\mathsf T}X)^{-1}X^{\mathsf T}\varepsilon-\beta$$

$$=(X^{\mathsf T}X)^{-1}X^{\mathsf T}\varepsilon$$

再代进方差定义（右侧那一栏）：

$$\operatorname{Var}(\hat\beta)=E\big[(\hat\beta-\beta)(\hat\beta-\beta)^{\mathsf T}\big]$$

$$=E\big[(X^{\mathsf T}X)^{-1}X^{\mathsf T}\varepsilon\varepsilon^{\mathsf T}X(X^{\mathsf T}X)^{-1}\big]$$

$$=(X^{\mathsf T}X)^{-1}X^{\mathsf T}E[\varepsilon\varepsilon^{\mathsf T}]X(X^{\mathsf T}X)^{-1}$$

$$=(X^{\mathsf T}X)^{-1}X^{\mathsf T}\,\sigma^2 I_n\,X(X^{\mathsf T}X)^{-1}$$

$$=\sigma^2(X^{\mathsf T}X)^{-1}$$

最后一步是 $(X^{\mathsf T}X)^{-1}(X^{\mathsf T}X)(X^{\mathsf T}X)^{-1}=(X^{\mathsf T}X)^{-1}$。（更正：原文第三行写成了 $(X^{\mathsf T}X^{-1})X^{\mathsf T}$，括号位置手滑，应为 $(X^{\mathsf T}X)^{-1}X^{\mathsf T}$。）

关键是 $E[\varepsilon\varepsilon^{\mathsf T}]=\sigma^2I_n$，它是「误差同方差且互不相关」这个假设的矩阵写法。

### 由协方差阵得到单个系数的 SE

第 $j$ 个系数 $\beta_j$ 的标准误差可估计为

$$SE(\hat\beta_j)=\hat\sigma\sqrt{a^{(jj)}}$$

其中 $\hat\sigma$ 是 $\sigma$ 的估计，$a^{(jj)}$ 为 $(X^{\mathsf T}X)^{-1}$ 的 $(j,j)$ 元素。也就是取协方差阵的对角元开根号。

这一段的意义在于：$\sigma$ 未知，$SE$ 只能估计；Bootstrap 给了另一条不依赖正态假设的路。

### 求 SE 的 Bootstrap 估计的步骤（绿色标注那一段，要背）

1. 自原始数据样本 $x=(x_1,x_2,\dots,x_n)$ 按**放回抽样**的方法，抽得容量为 $n$ 的样本 $Y=(Y_1,Y_2,\dots,Y_n)$。
2. 相继地、独立地求出 $B$ 个（$B\ge 1000$）容量为 $n$ 的 bootstrap 样本 $Y^{(i)}=(Y_1^{(i)},\dots,Y_n^{(i)})$，$i=1,2,\dots,B$。对于第 $i$ 个 bootstrap 样本，计算 $\hat\phi^{(i)}$。
3. 计算

$$SE=\sqrt{\frac{1}{B-1}\sum_{i=1}^{B}\big(\hat\phi^{(i)}-\bar\phi\big)^2},\qquad \bar\phi=\frac1B\sum_{i=1}^{B}\hat\phi^{(i)}$$

分母是 $B-1$ 不是 $B$，这就是样本标准差。右侧手写补了一句：每个 $Y^{(i)}$ 通过在原始样本中有放回地抽样得到。

## 两道例题（原笔记 p53）

### 例：身高体重的相关系数

设 $(H,W)$ 为某地小学五年级学生的身高和体重的总体，$(H,W)\sim F(\cdot,\cdot)$，求估计 $H$ 和 $W$ 的相关系数 $\phi$ 估计量的标准误差估计。调查了 $n=10$ 个学生：

| $h_i$ | 144 | 166 | 163 | 143 | 152 | 169 | 130 | 159 | 160 | 175 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| $w_i$ | 38 | 44 | 41 | 35 | 38 | 51 | 23 | 51 | 46 | 51 |

四个步骤（手写用圈码 ①②③④ 把它们和上一页的四步一一对应上了）：

1. 从 $F_n$ 中作 $n=10$ 次独立抽样，即从 $\{(h_1,w_1),\dots,(h_n,w_n)\}$ 中**有放回**独立抽取 $n$ 次，得到 $\hat F=F_n$ 的一组样本 $Y^{(1)}$。红笔批注：**每个样本的数据量和原始样本一样**。
2. 重复第 1 步，直到获取了 $B$ 组 bootstrap 样本 $Y^{(b)}$，$b=1,\dots,B$。
3. 对每一样本 $Y^{(b)}$ 计算样本相关系数 $\hat\phi^{(b)}=g(Y^{(b)})$。红笔批注：**计算的是 bootstrap 样本对应的统计量**。
4. 把 $\hat\phi^{(b)}$ 作为 $\hat F$ 下 $n=10$ 的样本相关系数的简单随机样本，估计其**样本标准差 $S$**，以 $S$ 作为 $\psi(g,\hat F,n)$ 的估计，进而用 $S$ 估计 $\hat\phi$ 在真实的总体分布 $F$ 下的标准误差 $SE(\hat\phi)$。

注意抽样抽的是**成对的** $(h_i,w_i)$，不能把身高和体重分开各自抽，否则相关系数就被破坏了。

### 例 1：中位数估计的标准误差（B=10 的手算）

某种基金的年回报率是具有分布函数 $F$ 的连续型随机变量，$F$ 未知，$F$ 的中位数 $\theta$ 是未知参数。数据（% 率）：

$$18.2,\ 9.5,\ 12.0,\ 21.1,\ 10.2\qquad (n=5)$$

以样本中位数作为总体中位数 $\theta$ 的估计。求中位数估计的标准误差的 bootstrap 估计。

**解**：原始样本自小到大排序为 $9.5,10.2,12.0,18.2,21.1$，中间一个数为 $12.0$，所以样本中位数 $\hat\theta=12.0$。

相继地、独立地在上述 5 个数据中按放回抽样取样，取 $B=10$ 得到 10 个 bootstrap 样本：

| 编号 | bootstrap 样本 | 中位数 $\hat\theta^{(i)}$ |
| --- | --- | --- |
| 1 | 9.5, 18.2, 12.0, 10.2, 18.2 | 12.0 |
| 2 | 21.1, 18.2, 12.0, 9.5, 10.2 | 12.0 |
| 3 | 21.1, 10.2, 10.2, 12.0, 10.2 | 10.2 |
| 4 | 18.2, 12.0, 9.5, 18.2, 10.2 | 12.0 |
| 5 | 21.1, 12.0, 18.2, 12.0, 18.2 | 18.2 |
| 6 | 10.2, 10.2, 9.5, 21.1, 10.2 | 10.2 |
| 7 | 9.5, 21.1, 12.0, 10.2, 12.0 | 12.0 |
| 8 | 10.2, 18.2, 10.2, 21.1, 21.1 | 18.2 |
| 9 | 10.2, 12.0, 18.2, 18.2, 18.2 | 18.2 |
| 10 | 18.2, 10.2, 18.2, 10.2, 10.2 | 10.2 |

十个中位数逐个已用 Python 复算，和课件给的一行数字完全一致。

页脚手写把均值算式写全了：

$$\bar\theta=\frac{1}{10}\sum_{i=1}^{10}\hat\theta^{(i)}=\frac{1}{10}(12.0+12.0+10.2+12.0+18.2+10.2+12.0+18.2+18.2+10.2)=13.32$$

标准误差的 bootstrap 估计（手写把公式抄在右边）：

$$SE=\sqrt{\frac{1}{B-1}\sum_{i=1}^{B}(\hat\theta^{(i)}-\bar\theta)^2}$$

代入 $B=10$：

$$\hat\psi=\sqrt{\frac19\sum_{i=1}^{10}(\hat\theta^{(i)}-\bar\theta)^2}=3.4579$$

Python 复核得 $3.4579$，和课件一致。

这里有个容易写错的地方：公式里减的是 **10 个 bootstrap 中位数的均值 $\bar\theta=13.32$**，不是原始样本的中位数 $12.0$。下一节估 MSE 时减的才是原始样本的估计值。

## 估计均方误差 MSE 的 Bootstrap 估计（原笔记 p54）

### 例 2：铂的升华热

设金属元素铂的升华热是具有分布函数 $F$ 的连续型随机变量，$F$ 的中位数是未知参数，现测得以下数据（以 kcal/mol 计，$n=26$，手写在末尾标了样本量）：

```text
136.3  136.6  135.8  135.4  134.7  135.0  134.1  143.3  147.8
148.8  134.8  135.2  134.9  149.5  141.2  135.4  134.8  135.8
135.0  133.7  134.4  134.9  134.8  134.5  134.3  135.2
```

求中位数 $\theta$ 的均方误差 $E(\hat\theta-\theta)^2$ 的 bootstrap 估计（以样本中位数作为总体中位数 $\theta$ 的估计）。

**解**：红笔在「解」字后面加了三个字：**要先排序**。这是手算这类题的第一步，也是最容易忘的一步。

$n=26$ 是偶数，排序后第 13、14 个数分别是 $135.0$ 和 $135.2$，所以原始样本中位数为

$$\hat\theta=\frac{135.0+135.2}{2}=135.1$$

Python 排序核对：第 13、14 位确实是 135.0 和 135.2，中位数 135.1。

相继地、独立地抽取 5 个 bootstrap 样本（每个容量仍是 26），求出各自的中位数。红笔画了一条长箭头从五个 Median 指到下面的求和式，批注「每个 bootstrap 样本的中位数」：

| 样本 | 中位数 |
| --- | --- |
| Sample 1 | 135.2 |
| Sample 2 | 135.1 |
| Sample 3 | 135.0 |
| Sample 4 | 134.9 |
| Sample 5 | 135.2 |

将上述 5 个数代入

$$\frac15\sum_{i=1}^{5}\big(\hat\theta^{(i)}-135.1\big)^2=0.014$$

即得中位数的均方误差 MSE 的 bootstrap 估计为 $0.014$。

逐项算一遍：$0.1^2+0^2+0.1^2+0.2^2+0.1^2=0.01+0+0.01+0.04+0.01=0.07$，$0.07/5=0.014$。Python 复核一致。

手写还在旁边记了 $\bar x=135.08$（五个中位数的平均，复核无误），以及一句对照：**相较于标准误差，带平方**。这句话点出了两个公式的差别：

| | 减去谁 | 分母 | 要不要开根号 |
| --- | --- | --- | --- |
| 标准误差 SE | bootstrap 估计的均值 $\bar\theta$ | $B-1$ | 要 |
| 均方误差 MSE | 原始样本的估计值 $\hat\theta$ | $B$ | 不要 |

注意 MSE 的定义里 $\theta$ 是真值，实际用 $\hat\theta=135.1$ 顶替。这就是「把样本当总体」在这道题里的具体做法。

## 估计偏差 bias 的 Bootstrap 估计（原笔记 p55）

### 例 3

接着例 2，求中位数 $\theta$ 的偏差的 bootstrap 估计。

设 $X=(X_1,\dots,X_n)$ 是来自总体 $F$ 的样本，$\hat\theta=\hat\theta(X_1,\dots,X_n)$ 是中位数 $\theta$ 的估计量。以样本中位数作为总体中位数 $\theta$ 的估计，由例 2 知原始样本的中位数为 $135.1$，以 $135.1$ 作为总体中位数 $\theta$ 的估计。

取 $\psi=\hat\theta-\theta$，需估计 $E(\hat\theta-\theta)$。红笔在 $135.1$ 上画箭头写了两个字：**作差**。和上一节唯一的区别就是不平方。

课件列了 5 个 bootstrap 样本及其中位数（$135.2,135.1,135,134.9,135.2$，和例 2 同一批），然后给出

$$b^{\star}=\frac{1}{10000}\sum_{i=1}^{10000}\big(\hat\theta^{(i)}-135.1\big)=\frac{1}{10000}\sum_{i=1}^{10000}\hat\theta^{(i)}-135.1=135.14-135.1=0.04$$

（更正：正文写的是「将上述 **1000** 个数取平均值」，而公式里的上下标都是 $10000$，例 4 用的也是 $B=10000$。这里的 1000 是课件笔误，应为 10000。）

$135.14-135.1=0.04$ 核对无误。

偏差估出来是正的 $0.04$，说明样本中位数在这批数据上略微偏大。三个公式放一起看：

$$\text{bias}=\frac1B\sum(\hat\theta^{(i)}-\hat\theta),\qquad \text{MSE}=\frac1B\sum(\hat\theta^{(i)}-\hat\theta)^2,\qquad SE=\sqrt{\frac{1}{B-1}\sum(\hat\theta^{(i)}-\bar\theta)^2}$$

bias 和 MSE 减的都是原始估计 $\hat\theta$，SE 减的是 bootstrap 均值 $\bar\theta$。

## Bootstrap 置信区间之百分位数法（原笔记 p56）

### 方法

1. 从样本 $x=(x_1,x_2,\dots,x_n)$ 中抽出 $B$ 个容量为 $n$ 的 **bootstrap 样本**。
2. 对于每个 bootstrap 样本求出 $\theta$ 的 **bootstrap 估计** $\hat\theta_1^\star,\hat\theta_2^\star,\dots,\hat\theta_B^\star$。
3. 将它们自小到大排序，得 $\hat\theta_{(1)}^\star,\hat\theta_{(2)}^\star,\dots,\hat\theta_{(B)}^\star$。

用对应的 $\hat\theta^\star$ 的分布作为 $\hat\theta$ 的分布的近似，求出 $\hat\theta^\star$ 的分布的近似分位数 $\hat\theta^\star_{\alpha/2}$ 和 $\hat\theta^\star_{1-\alpha/2}$，使

$$P\{\hat\theta^\star_{\alpha/2}<\hat\theta^\star<\hat\theta^\star_{1-\alpha/2}\}=1-\alpha$$

于是近似地有

$$P\{\hat\theta^\star_{\alpha/2}<\theta<\hat\theta^\star_{1-\alpha/2}\}=1-\alpha$$

红笔在这一步旁边写了一句话点破这个跳跃：**Bootstrap 的区间和总体一样**。这一步是整个百分位法唯一的近似，也是它的弱点所在（分布有偏时会失真）。

### 下标怎么取

$$k_1=\Big[B\times\frac{\alpha}{2}\Big],\qquad k_2=\Big[B\times\Big(1-\frac{\alpha}{2}\Big)\Big]$$

以 $\hat\theta^\star_{(k_1)}$ 和 $\hat\theta^\star_{(k_2)}$ 分别作为分位数 $\hat\theta^\star_{\alpha/2}$ 和 $\hat\theta^\star_{1-\alpha/2}$ 的估计，得到近似等式

$$P\{\hat\theta^\star_{(k_1)}<\theta<\hat\theta^\star_{(k_2)}\}=1-\alpha$$

于是得到 $\theta$ 的置信水平为 $1-\alpha$ 的近似置信区间 $(\hat\theta^\star_{(k_1)},\hat\theta^\star_{(k_2)})$，称为 $\theta$ 的置信水平为 $1-\alpha$ 的 **bootstrap 置信区间**。这种求置信区间的方法称为**分位数法**（百分位数法）。

黄笔在 $k_1,k_2$ 旁边打了问号，问的是这个方括号取整是**向上**还是**向下**。课件例题里 $B\alpha/2$ 恰好是整数，看不出来；p57 的 bootstrap-t 例题也是整数。实际实现一般取 $k_1=\lceil B\alpha/2\rceil$、$k_2=\lfloor B(1-\alpha/2)\rfloor$，也就是往区间内侧收，保证覆盖率不低于名义水平。（本整理稿补，原稿只打了问号没给答案；考试遇到不整除的数据先按题目给的记号写。）

页脚红笔画了一条数轴，两端各标 $\alpha/2$，旁边写「落在其间的概率为 $1-\alpha$」，就是双侧等尾的意思。

### 例 4

在例 2 中，以样本中位数作为总体中位数 $\theta$ 的估计，求 $\theta$ 的置信水平为 0.95 的 bootstrap 置信区间。

**解** $n=26$，$B=10000$，原始样本以及 10000 个模拟 bootstrap 样本见例 2。红笔批注「抽样本、初中位数」（先抽样本、再求中位数）。

对于每一个 bootstrap 样本算出中位数 $M_1^\star,M_2^\star,\dots,M_{10000}^\star$，自小到大排序得

$$M_{(1)}^\star\le M_{(2)}^\star\le\cdots\le M_{(250)}^\star\le M_{(251)}^\star\le\cdots\le M_{(9750)}^\star\le M_{(9751)}^\star\le\cdots\le M_{(10000)}^\star$$

由 $B=10000$，$1-\alpha=0.95$，$\alpha=0.05$，

$$k_1=\Big[10000\times\frac{0.05}{2}\Big]=250,\qquad k_2=\Big[10000\times\Big(1-\frac{0.05}{2}\Big)\Big]=9750$$

bootstrap 置信区间为

$$(M_{(250)}^\star,M_{(9750)}^\star)=(134.8,\ 135.8)$$

$250$ 和 $9750$ 已用 Python 核对。注意置信区间 $(134.8,135.8)$ 把点估计 $135.1$ 包在里面，位置偏右一点，和上一节算出的正偏差 $0.04$ 方向一致。

## Bootstrap 置信区间之 Bootstrap-t 法（原笔记 p57）

这一页以**求期望 $\phi$ 的 bootstrap 置信区间**为例。

### 枢轴量法的回顾

**枢轴量法**是构造置信区间的最基本的方法。设 $\phi$ 是总体 $F(\cdot)$ 的一个参数（比如期望），$\boldsymbol X=(x_1,\dots,x_n)$ 为来自总体的样本，容量为 $n$，均值和方差均为未知参数，要利用样本值来估计期望 $\phi$。

考虑函数 $g(\boldsymbol X)$ 为与 $\phi$ 有关系的一个统计量，经常是 $\phi$ 的估计量：

$$g(\boldsymbol X)=\frac{\bar X-\mu}{S/\sqrt n}$$

假设总体 $F$ 具有正态分布，此时 $g(\boldsymbol X)$ 的分布与参数 $\phi$ 无关，它是一个**枢轴量**，而且有 $g(\boldsymbol X)\sim t(n-1)$，利用枢轴量 $g(\boldsymbol X)$ 就能求得 $\phi$ 的置信区间。

红笔在下面写了一句限制条件：**总体不是正态分布的话，$g(\boldsymbol X)$ 就不是 $t$ 分布**。蓝笔在旁边标「传统方法」。这两句合起来就是 bootstrap-t 的动机：正态假设不成立时，改用重抽样去近似枢轴量的分布。

右上角手写了三条对枢轴量的说明，考概念就答这个：

- **枢轴量是一个函数，不是一个值。**
- **枢轴量包含待估计参数和样本，不包含其他未知参数。**
- 如果有变换 $W=h(g(\boldsymbol X),\phi)$ 使得 $W$ 的分布不依赖于任何未知参数，则设 $W$ 的左右两侧的分位数分别为 $W_{\alpha/2}$ 和 $W_{1-\alpha/2}$，有 $P\{W_{\alpha/2}<h(T,\phi)<W_{1-\alpha/2}\}=1-\alpha$，反解上面的不等式可以得到 $\phi$ 的置信区间。

课件补充：如果对枢轴量 $W$ 很难求分位数时，可以用 bootstrap 方法获得置信区间。设 $\hat F$ 为总体分布 $F$ 的估计，设 $Y=(y_1,\dots,y_n)$ 为总体 $\hat F$ 的样本，$\hat\phi=\phi(\hat F)$ 为与总体 $\hat F$ 对应的参数 $\phi$ 的值（实际是 $\phi$ 的估计值），则 $V=h(g(Y),\hat\phi)$ 与 $W$ 的分布相近，可以用 $V$ 的分位数近似 $W$ 的分位数。

### 用 Bootstrap 方法求 $\phi$ 的近似置信区间（手写完整推导）

原始样本 $\boldsymbol X=(x_1,x_2,\dots,x_n)$ 的样本均值 $\bar x=\frac1n\sum x_i$ 作为 $\phi$ 的估计。

考虑与 $g(\boldsymbol X)$ 相应的枢轴量

$$W^\star=\frac{\bar x^\star-\bar x}{s^\star/\sqrt n}$$

其中 $\bar x^\star$ 是 $\bar x$ 相应的 Bootstrap 样本均值，$s^\star$ 是 $s$ 相应的 Bootstrap 样本标准差。

注意分母里的**中心是 $\bar x$ 不是 $\mu$**。真值 $\mu$ 不知道，bootstrap 世界里的「真值」就是原始样本的 $\bar x$。这是这个方法最容易写错的一处。

用 $W^\star$ 的分布近似 $g(\boldsymbol X)$ 的分布，求出 $W^\star$ 的近似分位数为 $W^\star_{\alpha/2}$ 和 $W^\star_{1-\alpha/2}$：

$$P\Big\{W^\star_{\alpha/2}<\frac{\bar x^\star-\bar x}{s^\star/\sqrt n}<W^\star_{1-\alpha/2}\Big\}=1-\alpha$$

近似地有

$$P\Big\{W^\star_{\alpha/2}<\frac{\bar x-\phi}{s/\sqrt n}<W^\star_{1-\alpha/2}\Big\}=1-\alpha$$

反解不等式（注意两边乘的 $s/\sqrt n$ 为正，但移项后左右端点对调）：

$$\Longrightarrow\ P\Big\{\bar x-W^\star_{1-\alpha/2}\frac{s}{\sqrt n}<\phi<\bar x-W^\star_{\alpha/2}\frac{s}{\sqrt n}\Big\}=1-\alpha$$

将 $W^\star$ 的 $B$ 个 Bootstrap 值从小到大排序

$$W^\star_{(1)}\le W^\star_{(2)}\le\cdots\le W^\star_{(B)}$$

记

$$k_1=\Big[B\times\frac{\alpha}{2}\Big],\qquad k_2=\Big[B\times\Big(1-\frac{\alpha}{2}\Big)\Big]$$

作为分位数的估计，得

$$P\Big\{\bar x-W^\star_{(k_2)}\frac{s}{\sqrt n}<\phi<\bar x-W^\star_{(k_1)}\frac{s}{\sqrt n}\Big\}=1-\alpha$$

于是 $\phi$ 的置信水平为 $1-\alpha$ 的 Bootstrap 区间为

$$\Big(\bar x-W^\star_{(k_2)}\frac{s}{\sqrt n},\ \ \bar x-W^\star_{(k_1)}\frac{s}{\sqrt n}\Big)$$

**下标和端点是交叉的**：区间左端用大下标 $k_2$，右端用小下标 $k_1$，因为 $W^\star$ 前面带负号。百分位法不交叉，bootstrap-t 交叉，这两个别记混。

### 例 5

有 30 窝仔猪出生时各窝猪的存活只数为

```text
9  8  10  12  11  12   7   9  11   8   9   7   7   8   9   7
9  9  10   9   9   9  12  10  10   9  13  11  13   9
```

用 bootstrap-t 法求均值 $\mu$ 的置信水平为 0.90 的置信区间。手写标了 $1-\alpha=0.9$，「总体分布未知」。

右侧手写把解题步骤拆成四步：

1. 求原始样本的均值、方差、标准差。
2. 计算每个样本的 $\bar x_i^\star$、$s_i^\star$、$w_i^\star$。
3. 对 $w_i^\star$ 排序。
4. 区间估计（求置信区间）。

左下角手写把原始样本的三个数算了出来：

$$n=30,\quad \bar x=9.53,\quad s=\sqrt{\frac{\sum(x_i-\bar x)^2}{n-1}}=1.72,\quad s^2=\frac{\sum(x_i-\bar x)^2}{n-1}=2.95$$

Python 复核：30 个数之和 286，$\bar x=9.5333$，$s^2=2.9471$，$s=1.7167$，四舍五入后和笔记一致。

对于第 $i$ 个（$i=1,\dots,10000=B$）bootstrap 样本，求出它的均值 $\bar x_i^\star$ 和样本标准差 $s_i^\star$，从而得到 $w^\star$ 的第 $i$ 个值

$$w_i^\star=\frac{\bar x_i^\star-\bar x}{s_i^\star/\sqrt n},\quad i=1,2,\dots,10000$$

其中 $\bar x$ 是由原始样本确定的样本均值。

将 $w_i^\star$ 自小到大排序得到 $w^\star_{(1)}\le w^\star_{(2)}\le\cdots\le w^\star_{(10000)}$（红笔批注「找分位数」）。

取置信水平 $1-\alpha=0.90$，此时 $\alpha=0.10$，$\alpha/2=0.05$，$1-\alpha/2=0.95$，

$$k_1=\Big[B\times\frac{\alpha}{2}\Big]=500,\qquad k_2=\Big[B\times\Big(1-\frac{\alpha}{2}\Big)\Big]=9500$$

红笔在下面写了算式：$10000\times 0.05=500$，$10000\times(1-0.05)=9500$。得

$$w^\star_{(500)}=-1.7813,\qquad w^\star_{(9500)}=1.62999$$

于是得到 $\mu$ 的置信水平为 0.90 的 bootstrap-t 置信区间为

$$\Big(9.53-1.6299\times\frac{1.72}{\sqrt{30}},\ \ 9.53+1.7813\times\frac{1.72}{\sqrt{30}}\Big)=(9.0182,\ 10.0894)$$

红笔在 $9.53$ 下面标「$\bar x$」，在 $1.6299$ 下面标「$w_{(k_2)}$」，在 $1.72$ 上标「$s$」。

逐步核对（Python 复核）：$\dfrac{1.72}{\sqrt{30}}=0.3140$；左端 $9.53-1.6299\times 0.3140=9.0182$；右端 $9.53+1.7813\times 0.3140=10.0894$。

注意这个区间**不关于 $\bar x$ 对称**（左边差 0.512，右边差 0.559），因为 $w^\star$ 的两个分位数绝对值不相等。这正是 bootstrap-t 相对于正态区间的价值：它保留了枢轴量分布的偏斜。

另外，$9.53+1.7813\times\frac{s}{\sqrt n}$ 里的加号，展开就是 $\bar x-w^\star_{(500)}\frac{s}{\sqrt n}$，因为 $w^\star_{(500)}=-1.7813$ 是负的。别以为公式写错了。

## 3.7.3 参数 Bootstrap 方法（原笔记 p58）

### 方法

假设所研究的总体的分布函数 $F(x;\beta)$ 的**形式已知**，但其中包含未知参数 $\beta$（$\beta$ 可以是向量）。现在已知有一个来自 $F(x;\beta)$ 的样本 $X_1,X_2,\dots,X_n$，

1. 利用这一样本求出 $\beta$ 的**最大似然估计** $\hat\beta$；
2. 在 $F(x;\beta)$ 中以 $\hat\beta$ 代替 $\beta$ 得到 $F(x;\hat\beta)$；
3. 接着在 $F(x;\hat\beta)$ 中产生容量为 $n$ 的样本 $X_1^\star,X_2^\star,\dots,X_n^\star\sim F(x;\hat\beta)$；
4. 这种样本可以产生很多个，例如产生 $B$ 个（$B\ge 1000$），就可以利用这些样本对总体进行统计推断，其做法与**非参数 bootstrap 方法一样**。

这种方法称为**参数 bootstrap 法**。

和非参数法的唯一区别在第 3 步：非参数法从原始样本里有放回地抽，参数法从拟合好的分布里生成新数据。前提是分布族选对了。

### 例：血型的极大似然估计

据 Hardy-Weinberg 定律，若基因频率处于平衡状态，则在一总体中个体具有血型 M、MN、N 的概率分别是 $(1-\theta)^2$、$2\theta(1-\theta)$、$\theta^2$，其中 $0<\theta<1$。据 1937 年对某地区的调查有以下的数据：

| 血型 | M | MN | N | 合计 |
| --- | --- | --- | --- | --- |
| 人数 | 342 | 500 | 187 | 1029 |

红笔在三个人数下面依次标了 $x_1,x_2,x_3$，在表格左边写「来自总体的样本」、右边写「原始样本」。

问：(1) 求 $\theta$ 的最大似然估计；(2) 求 $\theta$ 的置信水平为 0.90 的 bootstrap 置信区间。

### (1) 极大似然估计（手写完整推导）

记 $x_1,x_2,x_3$ 为具有血型 M、MN、N 的人数，记 $x_1+x_2+x_3=n$。

**似然函数**（三种血型的概率各自取到对应的人数次方，相乘）：

$$L=\big[(1-\theta)^2\big]^{x_1}\cdot\big[2\theta(1-\theta)\big]^{x_2}\cdot\big[\theta^2\big]^{x_3}$$

合并同底数：

$$=2^{x_2}\,\theta^{x_2+2x_3}\,(1-\theta)^{2x_1+x_2}$$

$\theta$ 的指数 $x_2+2x_3$ 来自 MN 贡献一个 $\theta$、N 贡献两个 $\theta$；$(1-\theta)$ 的指数 $2x_1+x_2$ 来自 M 贡献两个、MN 贡献一个。

**对数似然**：

$$\ln L=x_2\ln 2+(x_2+2x_3)\ln\theta+(2x_1+x_2)\ln(1-\theta)$$

（更正：原文这一行把第一个加号写成了乘号，写作 $x_2\ln 2\cdot(x_2+2x_3)\ln\theta+\cdots$，取对数后三项应该是相加。）

**求导**：

$$\frac{\mathrm{d}\ln L}{\mathrm{d}\theta}=\frac{x_2+2x_3}{\theta}+\frac{-(2x_1+x_2)}{1-\theta}=0$$

$x_2\ln 2$ 是常数，求导没了。

**解方程**：通分得 $(x_2+2x_3)(1-\theta)=(2x_1+x_2)\theta$，展开整理

$$x_2+2x_3=\theta\big(2x_1+x_2+x_2+2x_3\big)=\theta\big(2x_1+2x_2+2x_3\big)$$

得

$$\hat\theta=\frac{x_2+2x_3}{2x_1+2x_2+2x_3}=\frac{x_2+2x_3}{2n}$$

（更正：原文最后一步写成了 $\dfrac{x_2+2x_3}{n}$，漏了分母上的 2。用后面的数据验算就能看出来：$\dfrac{874}{1029}=0.849$，和课件给的 $0.4247$ 对不上；$\dfrac{874}{2\times1029}=0.4247$ 才对。）

这个结果有个直观解释：$2n$ 是总的基因数（每人两条），$x_2+2x_3$ 是其中 N 型基因的个数，$\hat\theta$ 就是 **N 型基因的频率**。

代入数据 $x_1=342$，$x_2=500$，$x_3=187$，$n=1029$：

$$\hat\theta=\frac{500+2\times 187}{2\times 1029}=\frac{874}{2058}=0.4247$$

用 sympy 解同一个似然方程得到 $\hat\theta=\dfrac{x_2/2+x_3}{x_1+x_2+x_3}$，与 $\dfrac{x_2+2x_3}{2n}$ 恒等；代入数据得 $0.424684$，二阶导为负（约 $-8423$），确认是极大值点。

Python 复核脚本：

```python
import sympy as sp
th, x1, x2, x3 = sp.symbols('theta x1 x2 x3', positive=True)
lnL = x2*sp.log(2) + (x2+2*x3)*sp.log(th) + (2*x1+x2)*sp.log(1-th)
sol = sp.solve(sp.Eq(sp.diff(lnL, th), 0), th)[0]
print(sp.simplify(sol))                       # (x2/2 + x3)/(x1 + x2 + x3)
print(float(sol.subs({x1: 342, x2: 500, x3: 187})))   # 0.4246840...
```

### (2) 参数 bootstrap 置信区间

以数据 $x_1=342$，$x_2=500$，$x_3=187$，$n=1029$ 代入得到 $\hat\theta=0.4247$（红笔批注：**利用样本求极大似然估计**）。

以 $\hat\theta$ 代替 $\theta$，得到

$$(1-\hat\theta)^2=0.331,\qquad 2\hat\theta(1-\hat\theta)=0.489,\qquad \hat\theta^2=0.180$$

于是血型的近似分布律为

| 血型 | M | MN | N |
| --- | --- | --- | --- |
| 概率 | 0.331 | 0.489 | 0.180 |

三个概率加起来是 1.000（复核：0.3310+0.4887+0.1804，精确值合计为 1）。

以此分布律产生 10000 个 bootstrap 样本（每个样本还是 1029 个人），从而得到 $\theta$ 的 10000 个 bootstrap 估计 $\hat\theta_1^\star,\dots,\hat\theta_{10000}^\star$。红笔批注：**每个 bootstrap 样本都有一个对应的 $\hat\theta$**。黄笔在分布律表下面画了个问号，问的应该是「怎么按这个分布律生成样本」，答案是从三项分布 $\mathrm{Multinomial}(1029;0.331,0.489,0.180)$ 里抽一次，得到一组 $(x_1^\star,x_2^\star,x_3^\star)$，再套 $\hat\theta^\star=\frac{x_2^\star+2x_3^\star}{2n}$（本整理稿补）。

将这 10000 个数按自小到大的次序排序得到 $\hat\theta^\star_{(1)}\le\cdots\le\hat\theta^\star_{(10000)}$，取

$$\big(\hat\theta^\star_{(500)},\hat\theta^\star_{(9500)}\big)$$

为 $\theta$ 的置信水平为 0.90 的 bootstrap 置信区间。红笔批注：**和非参数方法一样，分位数法**。下标 500 和 9500 的来历和例 5 完全相同。

（更正：课件把这个区间的数值写成了 $(1.83,1.92)$，这不可能，题目已经限定 $0<\theta<1$。本整理稿按同样流程复算：用 $\mathrm{Multinomial}(1029;\ 0.3310,0.4887,0.1804)$ 生成 $B=10000$ 个样本，换 8 个随机种子重复，第 500 位稳定在 $0.4067\sim 0.4072$，第 9500 位稳定在 $0.4422\sim 0.4427$，即区间约为 $(0.407,\ 0.443)$。正态近似 $\hat\theta\pm 1.645\sqrt{\widehat{\operatorname{Var}}(\hat\theta)}=(0.4068,0.4426)$ 与之吻合。考试答这道题时把区间写成关于 $0.4247$ 大致对称、宽度约 0.036 的样子就对了。）

## 本章易错点

- 三个公式的中心和分母不一样。SE 减 bootstrap 均值 $\bar\theta$、除以 $B-1$、要开根号；MSE 和 bias 减原始估计 $\hat\theta$、除以 $B$，MSE 平方、bias 不平方。考试最常丢分的就是这里。
- 每个 bootstrap 样本的容量必须等于原始样本容量 $n$，重抽 $B$ 次是「样本个数」，不是「样本容量」。
- 多维数据（比如例题里的身高体重对）要成对抽，不能拆开分别抽。
- 百分位法的区间是 $(\hat\theta^\star_{(k_1)},\hat\theta^\star_{(k_2)})$，下标顺着走；bootstrap-t 的区间是 $\big(\bar x-W^\star_{(k_2)}\frac{s}{\sqrt n},\ \bar x-W^\star_{(k_1)}\frac{s}{\sqrt n}\big)$，下标是交叉的。
- bootstrap-t 的枢轴量 $W^\star=\dfrac{\bar x^\star-\bar x}{s^\star/\sqrt n}$，分子中心是原始样本均值 $\bar x$，不是未知的 $\mu$；分母用的是 bootstrap 样本自己的标准差 $s^\star$。
- 参数 bootstrap 和非参数 bootstrap 只差「新样本从哪来」：参数法从 $F(x;\hat\beta)$ 生成，非参数法从原始样本有放回抽。后续求 SE、求区间的做法完全一样。
- 血型那道题的 $\hat\theta=\dfrac{x_2+2x_3}{2n}$，分母上的 2 不能丢。写似然函数时先把 $2^{x_2}$ 提出来，它是常数，对求导没影响但漏写会让式子对不上。
- 枢轴量的定义要说清两点：它是函数不是数值；它只能含待估参数和样本，不能含别的未知参数。
