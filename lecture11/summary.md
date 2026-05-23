# 波动率微笑曲线计算步骤

1. **读取期权数据**  
   `calls = pd.read_table('callsfor15mar2024.txt')`

2. **获取股票价格与到期时间**  
   - 从 `ibm.pkl` 读取股价，取最新收盘价 `s`  
   - 从合约名解析到期日，计算剩余时间 `t = (exp_date - enddate2).days / 252.0`

3. **计算期权的中间价**  
   `c = (calls['Bid'] + calls['Ask']) / 2.0`，过滤 `c > 0`

4. **循环求隐含波动率**  
   对每个行权价 `x` 和对应 `c`，调用 `implied_vol_call_min(s, x, t, r, c)`（暴力搜索）得到 `vol`

5. **记录并绘图**  
   - 存储 `(strike, implied_vol)`  
   - `plt.plot(strike, implied_vol, marker='o')`  
   - 标题、轴标签：`Strike Price` vs `Implied Volatility`

# SQL：创建表、插入数据与内连接查询

## 创建表

```sql
CREATE TABLE departments (
    dept_id INTEGER PRIMARY KEY,
    dept_name TEXT
);

CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name TEXT,
    dept_id INTEGER
);
```

---

## 插入数据

```python
cursor.executemany(
    'INSERT INTO departments VALUES (?, ?)',
    [
        (10, 'Eng'),
        (20, 'Sales'),
        (30, 'Mkt')
    ]
)

cursor.executemany(
    'INSERT INTO employees VALUES (?, ?, ?)',
    [
        (1, 'Alice', 10),
        (2, 'Bob', 20),
        (3, 'Charlie', None)
    ]
)
```

---

# 内连接查询（INNER JOIN）

```sql
SELECT employees.name, departments.dept_name
FROM employees
INNER JOIN departments
ON employees.dept_id = departments.dept_id;
```

---

## 查询结果示例

| name  | dept_name |
|------|------|
| Alice | Eng |
| Bob | Sales |

说明：

- `INNER JOIN` 只返回两张表中能够匹配的数据
- `Charlie` 的 `dept_id` 为 `NULL`
- 因此不会出现在查询结果中



# Gamma 的三种计算方法

## 1. 公式法（解析法）

```math
\Gamma=\frac{N'(d_1)}{S\sigma\sqrt{T}}
```

对应 Python：

```python
gamma = stats.norm.pdf(d1) / (S * sigma * sqrt(T))
```

---

# 2. 前向差分法（Forward Difference）

```python
s1 = S
s2 = S + bump
s3 = S + 2 * bump

c1 = bsCall(s1, X, T, r, sigma)
c2 = bsCall(s2, X, T, r, sigma)
c3 = bsCall(s3, X, T, r, sigma)

gamma = (c3 - 2 * c2 + c1) / (bump ** 2)
```

特点：

- 使用未来价格点估计
- 实现简单
- 精度一般

---

# 3. 中心差分法（Central Difference）

```python
def gamma3_f(S, X, T, r, sigma):
    s1 = S - tiny
    s2 = S
    s3 = S + tiny

    c1 = bsCall(s1, X, T, r, sigma)
    c2 = bsCall(s2, X, T, r, sigma)
    c3 = bsCall(s3, X, T, r, sigma)

    gamma = (c3 - 2 * c2 + c1) / (tiny ** 2)

    return gamma
```

特点：

- 使用左右对称价格点
- 精度高于前向差分
- 数值计算中更常用