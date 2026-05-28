# Lecture 12 — 风险度量与 VaR（整理版）

**课程思路概览**：风险度量与投资组合风险管理。由单资产 VaR 出发，比较参数法（正态假设）与非参数法（历史法），检验假设，使用 Cornish–Fisher 修正高阶矩，补充 Expected Shortfall，并推广到多资产组合波动率计算。

## 目录（按 notebook 顺序）
1. 单只股票持仓价值（基准）
2. 单日 VaR（参数法 - 正态分布假设）
3. 单日 VaR（历史法 - 排序法）
4. 多日 VaR - 10 天收益（参数法）
5. 多日 VaR - 10 天收益（历史法 - 排序法）
6. 正态性检验
7. 偏度与峰度的计算
8. Cornish-Fisher 修正的 VaR（基于高阶矩）
9. Expected Shortfall（条件期望损失）
10. 两资产组合波动率
11. 实际数据：IBM 和 WMT 的两资产组合
12. N 资产组合波动率


---

## 1. 单只股票持仓价值（基准）

**目的**：计算当前持仓的总市值，是后续 VaR 计算的基础。

**关键代码**：
```python
wealth = nStocks * df['Close'].iloc[-1]
```

**说明**：用股票数量乘以最新收盘价得到当前持仓价值。

---

## 2. 单日 VaR（参数法 - 正态分布假设）

**原理**：假设收益率服从正态分布 $r \sim N(\mu, \sigma^2)$，在置信水平 $\alpha$ 下的左尾分位点为：

$$R_{\alpha} = \mu + z_{\alpha} \sigma$$

其中 $z_{\alpha} = \Phi^{-1}(1-\alpha)$（对于 99% VaR，$z \approx -2.33$）。

**持仓 VaR 计算**（按代码符号）：

$$\text{VaR} = P \times (\mu - z \sigma)$$

**关键代码**：
```python
z = norm.ppf(1 - confidence_level)  # 正态分位数
ret = df['Adj Close'].pct_change().dropna()  # 日收益率
mean = ret.mean()  # 均值 μ
stdev = ret.std()  # 标准差 σ
var = position * (mean - z * stdev)  # VaR
```

**注意**：由于 $z$ 为负数（约 -2.33），公式 $\mu - z\sigma$ 实际上是在加上一个正的风险溢价。

---

## 3. 单日 VaR（历史法 - 排序法）

**原理**：不假设任何分布，直接从历史数据中取第 1% 小的收益率作为分位点，再乘以持仓价值。

**计算步骤**：
1. 对历史收益率序列按升序排序
2. 找到第 $m = \lfloor n(1-\alpha) \rfloor$ 个位置的收益率
3. 持仓 VaR = $P \times r_{(m)}$

**优缺点**：
- 优点：无需分布假设，能捕捉极端值和厚尾现象
- 缺点：对样本敏感，样本量小时不稳定

**关键代码**：
```python
ret2 = np.sort(df['ret'])
m = int(n * (1 - confidence_level))
var_sort = position * ret2[m]
```

---

## 4. 多日 VaR - 10 天收益（参数法）

**原理**：多期收益采用复合收益计算：

$$R_{10日} = \prod_{t=1}^{10}(1+r_t) - 1$$

然后在多日收益序列上应用参数法 VaR 计算。

**关键步骤**：
1. 计算日收益率：`ret = pct_change()`
2. 转换为收益加 1：`retplus = ret + 1`
3. 按 10 天分组，计算每组的累积收益：`groupby.prod() - 1`
4. 在多日收益序列上计算均值、标准差、VaR

**说明**：时间越长，收益率方差越大，VaR 也越大。

---

## 5. 多日 VaR - 10 天收益（历史法 - 排序法）

**原理**：与单日历史法相同，但基于 10 天复合收益序列：

1. 对 10 日收益序列排序
2. 取左侧第 1% 的收益
3. 持仓 VaR = $P \times r_{10日,(m)}$

**代码示例**：
```python
ret3 = np.sort(ret2)  # ret2 为 10 日复合收益
lefttail = int(n * (1 - confidence_level))
var_sort = position * ret3[lefttail]
```

---

## 6. 正态性检验

**目的**：验证参数法 VaR 的关键假设——收益率是否服从正态分布。

**检验方法**：
- **Shapiro-Wilk 检验**：小样本首选
- **Anderson-Darling 检验**：大样本也适用

**检验结果解读**：
- p-value < 0.05：拒绝正态分布假设
- 实际股票收益往往呈现**厚尾**、**负偏**，说明参数法可能**低估风险**

---

## 7. 偏度与峰度的计算

**偏度（Skewness）**：衡量分布的不对称性。第 3 阶中心矩的标准化：

$$s = \frac{1}{n}\sum_{i=1}^{n}\left(\frac{r_i - \mu}{\sigma}\right)^3$$

- 负偏（s < 0）：左尾更厚，坏事发生概率更大
- 正偏（s > 0）：右尾更厚

**峰度（Kurtosis）**：衡量分布的尾部厚度。第 4 阶中心矩的标准化：

$$k = \frac{1}{n}\sum_{i=1}^{n}\left(\frac{r_i - \mu}{\sigma}\right)^4 - 3$$

- 正态分布的峰度 ≈ 0（多数库返回 Fisher 峰度）
- 峰度 > 0：厚尾现象，极端事件更频繁

**股票收益特征**：通常表现为**负偏 + 高峰度**，即极端亏损比预期更常见。

---

## 8. Cornish-Fisher 修正的 VaR（基于高阶矩）

**问题**：正态假设会低估风险，需要用偏度和峰度来修正。

**Cornish-Fisher 修正**：将正态分位数 $z$ 扩展为 $t$：

$$t = z + \frac{1}{6}(z^2-1)s + \frac{1}{24}(z^3-3z)k - \frac{1}{36}(2z^3-5z)s^2$$

其中 $s$ 为偏度，$k$ 为峰度。

**修正后的 VaR**：

$$\text{VaR}_{\text{CF}} = P \times (\mu - t \sigma)$$

**效果**：
- 若收益呈现负偏和高峰度，修正后的 VaR > 参数法 VaR（更保守）
- 若偏度和峰度接近正态分布，修正效果有限

**代码实现**：
```python
s = stats.skew(ret)
k = stats.kurtosis(ret)
t = z + (1/6)*(z**2-1)*s + (1/24)*(z**3-3*z)*k - (1/36)*(2*z**3-5*z)*s**2
var_cf = position * (mean - t * stdev)
```

---

## 9. Expected Shortfall（条件期望损失）

**定义**：在给定置信水平下，条件于损失超过 VaR 时的**平均损失**。

$$\text{ES}_{\alpha} = E[R | R < R_{\text{VaR}}]$$

**历史法计算**：取最坏的 $m$ 个收益的平均值：

$$\text{ES} = P \times \frac{1}{m} \sum_{i=1}^{m} r_{(i)}, \quad r_{(i)} \text{ 为按升序排序后的收益}$$

**意义**：
- VaR 只告诉你"最坏情况下损失多少"
- ES 回答"如果超过 VaR，平均会损失多少"
- ES > VaR，两者结合可更全面地衡量尾部风险

**代码示例**：
```python
ret_sorted = np.sort(ret)
m = int(n * (1 - confidence_level))
es = position * (ret_sorted[:m].mean())
```

---

## 10. 两资产组合波动率

**公式**：

$$\sigma_p = \sqrt{w_1^2 \sigma_1^2 + w_2^2 \sigma_2^2 + 2 w_1 w_2 \text{Cov}(r_1, r_2)}$$

其中：
- $w_1, w_2$：两个资产的权重（$w_1 + w_2 = 1$）
- $\sigma_1, \sigma_2$：两个资产的波动率
- $\text{Cov}(r_1, r_2)$：两个资产收益的协方差

**关键点**：
- 相关性 < 1 时，组合波动率 < 权重加权平均波动率
- 这就是**分散化收益**的来源

**代码实现**：
```python
def portfolio_volatility(ret1, ret2, w1):
    x1, x2 = w1, 1 - w1
    s1, s2 = ret1.std(), ret2.std()
    cov = np.cov(ret1, ret2)[1][1]
    return sqrt(x1**2 * s1**2 + x2**2 * s2**2 + 2 * x1 * x2 * cov)
```

---

## 11. 实际数据：IBM 和 WMT 的两资产组合

**流程**：
1. 分别载入 IBM 和 WMT 的历史数据
2. 计算各自的日收益率
3. 合并两个收益序列（按日期对齐）
4. 指定权重（例如 50%-50%）
5. 调用 `portfolio_volatility()` 计算组合波动率

**说明**：这是两资产公式在真实数据上的应用演示。

---

## 12. N 资产组合波动率

**通用公式**（矩阵形式）：

$$\sigma_p = \sqrt{\mathbf{w}^T \Sigma \mathbf{w}}$$

其中：
- $\mathbf{w} = (w_1, w_2, \ldots, w_n)^T$ 为权重向量
- $\Sigma$ 为 $n \times n$ 的协方差矩阵

**计算步骤**：
1. 组织收益率矩阵：$R \in \mathbb{R}^{T \times n}$（T 个时间点，n 个资产）
2. 计算协方差矩阵：$\Sigma = \text{Cov}(R)$
3. 计算 $\sigma_p = \sqrt{\mathbf{w}^T \Sigma \mathbf{w}}$

**代码实现**：
```python
def vol_portfolio(ret_matrix, w):
    cov = np.cov(ret_matrix.T)
    final = 0
    for i in range(len(w)):
        for j in range(len(w)):
            final += w[i] * w[j] * cov[i][j]
    return sqrt(final)
```

或使用矩阵运算简化：
```python
cov = np.cov(ret_matrix.T)
var_p = float(w.dot(cov).dot(w))
return sqrt(var_p)
```

---

## 重要关系与实践建议

### 参数法 vs 历史法
| 特性 | 参数法 | 历史法 |
|------|------|------|
| 计算速度 | 快 | 较慢 |
| 分布假设 | 需要（通常假设正态） | 无 |
| 极端值捕捉 | 可能低估 | 较好 |
| 样本依赖性 | 低（只需 μ, σ） | 高（需完整历史） |

### 正态性失效时的处理
1. **做正态性检验**：观察 p-value 和偏度/峰度
2. **如果拒绝正态**：
   - 优先考虑历史法或 Cornish-Fisher 修正
   - 不要单独依赖参数法
3. **补充 ES**：VaR + ES 结合，更全面衡量尾部风险

### 组合构建的核心
- **协方差矩阵**是关键：准确估计各资产间的相关性
- **权重优化**：在给定收益目标下最小化波动率（均值-方差优化）
- **多元化效应**：当相关性 < 1 时存在；相关性越低，效应越明显

---

## 复习路径建议

**第一阶段**：单资产 VaR 基础
1. 读取数据，计算日收益率
2. 做正态性检验，观察偏度/峰度
3. 用参数法计算 VaR
4. 用历史法计算 VaR，对比两种方法

**第二阶段**：VaR 方法改进
1. 计算 Cornish-Fisher 修正值 $t$，比较修正前后的 VaR
2. 计算 Expected Shortfall（ES），理解 VaR 的局限性

**第三阶段**：多期 VaR
1. 学习多期复合收益的计算
2. 在多期收益上应用参数法和历史法
3. 理解时间如何影响风险

**第四阶段**：投资组合风险
1. 学习二资产组合波动率公式及相关性的作用
2. 推广到 N 资产的矩阵形式
3. 在真实数据上实验，提高对分散化的理解

---

> 本文件对应 lecture12_2.ipynb 的内容顺序。建议对照 notebook 的代码进行学习和实践。
