# MySQL 数据查询参考

## 常用查询模式

### 连接数据库
```python
import pymysql
import pandas as pd

conn = pymysql.connect(
    host='localhost',
    port=3306,
    user='username',
    password='password',
    database='db_name',
    charset='utf8mb4'
)

# 使用 pandas 读取
df = pd.read_sql('SELECT * FROM table_name', conn)
```

### 导入数据到数据库
```python
import pandas as pd
from sqlalchemy import create_engine
import pymysql

# 创建数据库（如不存在）
conn = pymysql.connect(host='localhost', port=3306, user='root', password='pwd')
cursor = conn.cursor()
cursor.execute("CREATE DATABASE IF NOT EXISTS db_name CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci")
conn.close()

# 导入 DataFrame
engine = create_engine('mysql+pymysql://root:pwd@localhost:3306/db_name?charset=utf8mb4')
df = pd.read_excel('cleaned_data.xlsx')
df.to_sql(name='table_name', con=engine, if_exists='replace', index=False, chunksize=50)

# 验证
result = pd.read_sql('SELECT COUNT(*) as cnt FROM table_name', engine)
print(f"导入 {result['cnt'].iloc[0]} 条记录")
```

### 数据探查
```sql
-- 表结构
DESCRIBE table_name;

-- 行数
SELECT COUNT(*) FROM table_name;

-- 列唯一值
SELECT COUNT(DISTINCT column_name) FROM table_name;

-- 空值检查
SELECT 
    COUNT(*) - COUNT(column1) AS column1_nulls,
    COUNT(*) - COUNT(column2) AS column2_nulls
FROM table_name;
```

### 清洗类查询
```sql
-- 查找重复
SELECT column1, column2, COUNT(*)
FROM table_name
GROUP BY column1, column2
HAVING COUNT(*) > 1;

-- 查找异常值
SELECT * FROM table_name
WHERE value > (SELECT AVG(value) + 3 * STDDEV(value) FROM table_name);

-- 日期范围
SELECT MIN(date_col), MAX(date_col) FROM table_name;
```

### 主键、索引、外键、视图

```sql
-- 添加自增主键
ALTER TABLE table_name ADD COLUMN id INT AUTO_INCREMENT PRIMARY KEY FIRST;

-- 单列索引
CREATE INDEX idx_keyword ON table_name(keyword);
CREATE INDEX idx_position ON table_name(position);

-- 联合索引（多列查询）
CREATE INDEX idx_cat_date ON orders(category, order_date);

-- 唯一索引
CREATE UNIQUE INDEX idx_email ON users(email);

-- 外键（关联维度表）
ALTER TABLE fact_sales
ADD CONSTRAINT fk_product
FOREIGN KEY (product_id) REFERENCES dim_product(id);

-- 查看表的索引
SHOW INDEX FROM table_name;

-- 删除索引
DROP INDEX idx_keyword ON table_name;

-- 创建视图
CREATE OR REPLACE VIEW v_top_traffic AS
SELECT keyword, traffic, position, search_volume
FROM table_name
ORDER BY traffic DESC
LIMIT 20;

-- 创建月度汇总视图
CREATE OR REPLACE VIEW v_monthly_summary AS
SELECT DATE_FORMAT(order_date, '%Y-%m') AS month,
       SUM(revenue) AS total_revenue,
       COUNT(*) AS order_count,
       AVG(revenue) AS avg_revenue
FROM orders
GROUP BY month;

-- 使用视图
SELECT * FROM v_top_traffic;
SELECT * FROM v_monthly_summary WHERE month >= '2025-01';
```

### 分析类查询
```sql
-- 分组聚合
SELECT category, SUM(revenue) AS total, AVG(revenue) AS avg
FROM sales
GROUP BY category
ORDER BY total DESC;

-- 月度趋势
SELECT DATE_FORMAT(order_date, '%Y-%m') AS month,
       SUM(revenue) AS revenue
FROM sales
GROUP BY month
ORDER BY month;

-- 窗口函数：排名
SELECT product, revenue,
       RANK() OVER (ORDER BY revenue DESC) AS rank
FROM products;

-- 窗口函数：同比
SELECT month, revenue,
       LAG(revenue, 12) OVER (ORDER BY month) AS prev_year,
       (revenue - LAG(revenue, 12) OVER (ORDER BY month))
       / LAG(revenue, 12) OVER (ORDER BY month) * 100 AS yoy_growth
FROM monthly_sales;
```
