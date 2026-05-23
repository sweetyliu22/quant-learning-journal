# 期权与 BS 模型笔记

## 1. 期权的到期收益（Payoff）

设：

- 标的资产到期价格：`S_t`
- 行权价：`X`
- 期权费：`C`

### 看涨期权（Call）

到期收益：

```math
\max(S_t - X, 0) - C
```

含义：

- 若 `S_t > X`，执行期权获利
- 若 `S_t \le X`，放弃执行

---

### 看跌期权（Put）

到期收益：

```math
\max(X - S_t, 0) - C
```

含义：

- 若 `S_t < X`，执行期权获利
- 若 `S_t \ge X`，放弃执行

---

# 2. BS（Black-Scholes）期权定价模型

## 参数说明

- `S`：当前股价
- `X`：行权价
- `r`：无风险利率
- `\sigma`：波动率
- `t`：到期时间

---

## d1

```math
d_1=\frac{\ln(S/X)+(r+\sigma^2/2)t}{\sigma\sqrt{t}}
```

---

## d2

```math
d_2=d_1-\sigma\sqrt{t}
```

---

## 看涨期权价格（Call）

```math
C=S\cdot N(d_1)-Xe^{-rt}N(d_2)
```

对应 Python：

```python
c = s * stats.norm.cdf(d1) - x * exp(-r*t) * stats.norm.cdf(d2)
```

---

## 看跌期权价格（Put）

```math
P=Xe^{-rt}N(-d_2)-SN(-d_1)
```

对应 Python：

```python
p = x * exp(-r*t) * stats.norm.cdf(-d2) - s * stats.norm.cdf(-d1)
```

---

# 3. 隐含波动率（Implied Volatility）

核心思想：

不断调整波动率 `\sigma`，使：

```text
BS 模型计算价格 ≈ 市场真实价格
```

即：

```text
理论价格 = 市场价格
```

常用方法：

- 二分法
- 牛顿迭代法

---

# 4. 利率平价与外汇远期套利

## 已知条件

- 三个月后需要支付：`1 美元`
- 即期汇率：`1 : 7`
- 远期汇率：`1 : 8`
- 本币利率：`4%`
- 外币利率：`8%`

---

## 套利思路

设现在买入 `x` 美元。

### 第一步：兑换美元

现在使用：

```text
7x 本币
```

兑换：

```text
x 美元
```

并存入美元账户。

---

### 第二步：美元升值

三个月后：

```text
x 美元 → 1.02x 美元
```

---

### 第三步：按远期汇率换回本币

远期汇率：

```text
1 美元 = 8 本币
```

则：

```text
1.02x × 8 = 8.16x 本币
```

---

### 第四步：套利收益

初始投入：

```text
7x 本币
```

最终得到：

```text
8.16x 本币
```

利润：

```text
8.16x - 7x = 1.16x
```

若仅比较利差套利部分：

```text
额外收益 = 0.16x 本币
```

---

# 5. Delta 的两种计算方法

## 方法一：公式法

```math
\Delta = N(d_1)
```

对应 Python：

```python
delta = stats.norm.cdf(d1)
```

---

## 方法二：差分法（数值法）

```math
\Delta = \frac{C_2 - C_1}{S_2 - S_1}
```

其中：

- `C_1, C_2`：期权价格变化
- `S_1, S_2`：标的资产价格变化

用于近似计算 Delta。