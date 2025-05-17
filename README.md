# 设计文档

## 1. 需求分析

### 1.1 项目背景
消费者价格指数（CPI）是衡量通货膨胀的重要指标，但传统 CPI 依赖人工调研，更新频率低（月度）、覆盖范围有限。电商数据具有高频（每日更新）、覆盖面广、数据粒度细等优势，可用于构建更灵敏的价格指数，辅助宏观经济监测。

### 1.2 核心需求
- **数据存储与管理**：使用阿里云对象存储（OSS）管理商品分类和价格数据；通过 ClickHouse 实现高效列式查询，支持大规模数据分析。
- **价格指数计算**：计算分类价格指数（按销量加权）；构建日度价格指数时间序列（支持链式或固定基期计算）。
- **自动化与可视化**：基于 Python 实现自动化计算流程；使用 Python 的 `matplotlib` 或 MATLAB 可视化价格指数趋势。
- **异常处理**：处理缺失价格、异常值等问题。
- **测试与验证**：单元测试（小规模数据验证 SQL 逻辑）；集成测试（覆盖率≥80%，异常过滤准确率≥95%）。

## 2. 数据存储与表设计

### 2.1 数据源
- `category.csv`：商品分类信息（如 `category_id`, `category_name`, `parent_category`）。
- `price.csv`：商品每日价格与销量（如 `product_id`, `date`, `price`, `sales`）。

### 2.2 存储方案

#### 阿里云OSS
- 创建 Bucket（如 `ecommerce-price-index`）。
- 上传 `category.csv` 和 `price.csv`。

#### ClickHouse外部表映射
```sql
-- 创建 category 表
CREATE TABLE category (
    category_id String,
    category_name String,
    parent_category String
) ENGINE = OSS('oss-endpoint', 'bucket-name', 'category.csv', 'CSVWithNames');

-- 创建 price 表
CREATE TABLE price (
    product_id String,
    date Date,
    price Float32,
    sales Int32,
    category_id String
) ENGINE = OSS('oss-endpoint', 'bucket-name', 'price.csv', 'CSVWithNames');

-- 计算每日分类加权平均价格
SELECT 
    date,
    category_id,
    SUM(price * sales) / SUM(sales) AS weighted_avg_price
FROM price
GROUP BY date, category_id;

```

### 2.3 数据预处理
- **统一时间格式**：将所有时间数据格式化为 `YYYY-MM-DD`。
- **标准化价格单位**：将所有价格统一为人民币元。
- **处理缺失值**：填充或剔除异常价格。

## 3. 核心算法设计

### 3.1 价格指数计算逻辑

#### （1）分类价格指数（销量加权）
```sql
-- 计算每日分类加权平均价格
SELECT 
    date,
    category_id,
    SUM(price * sales) / SUM(sales) AS weighted_avg_price
FROM price
GROUP BY date, category_id;

-- 计算基期（如2023-01-01）价格
WITH base_prices AS (
    SELECT 
        category_id,
        SUM(price * sales) / SUM(sales) AS base_price
    FROM price
    WHERE date = '2023-01-01'
    GROUP BY category_id
)

-- 计算每日价格指数（链式）
SELECT 
    p.date,
    SUM(p.weighted_avg_price * c.weight) / SUM(b.base_price * c.weight) * 100 AS price_index
FROM (
    SELECT 
        date,
        category_id,
        SUM(price * sales) / SUM(sales) AS weighted_avg_price
    FROM price
    GROUP BY date, category_id
) p
JOIN base_prices b ON p.category_id = b.category_id
JOIN category_weights c ON p.category_id = c.category_id
GROUP BY p.date
ORDER BY p.date;

```

### 3.2 异常处理
- **缺失价格**：使用前一日价格填充（Last Observation Carried Forward, LOCF）。
- **异常值**：剔除价格波动超过±3倍标准差的记录。

## 4. 可视化与自动化
### 4.1 Python 自动化脚本
```python
import pandas as pd
import matplotlib.pyplot as plt
from clickhouse_driver import Client

# 连接 ClickHouse
client = Client(host='localhost', user='default', password='')

# 查询价格指数数据
query = """
SELECT date, price_index FROM daily_price_index ORDER BY date
"""
df = pd.DataFrame(client.execute(query), columns=['date', 'price_index'])

# 可视化
plt.figure(figsize=(12, 6))
plt.plot(df['date'], df['price_index'], label='Daily Price Index')
plt.title('E-commerce Price Index Trend')
plt.xlabel('Date')
plt.ylabel('Index (Base=100)')
plt.grid()
plt.show()
```
### 4.2 Python Matplotlib
创建折线图展示日度价格指数趋势。

