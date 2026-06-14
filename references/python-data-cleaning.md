# Python 数据处理参考

## pandas 常用操作速查

### 读取数据
```python
import pandas as pd

# CSV
df = pd.read_csv('file.csv', encoding='utf-8')
df = pd.read_csv('file.csv', encoding='gbk')  # 中文 GBK

# Excel
df = pd.read_excel('file.xlsx', sheet_name='Sheet1')

# JSON
df = pd.read_json('file.json')

# SQL
from sqlalchemy import create_engine
engine = create_engine('mysql+pymysql://user:pass@host:3306/db')
df = pd.read_sql('SELECT * FROM table', engine)
```

### 数据探查
```python
df.head(10)          # 前 N 行
df.info()            # 列信息、类型、非空数
df.describe()        # 数值统计摘要
df.describe(include='object')  # 分类统计
df.isnull().sum()    # 每列缺失值
df.nunique()         # 每列唯一值数
df['col'].value_counts()  # 分类分布
```

### 数据选择
```python
df['col']            # 单列
df[['a', 'b']]       # 多列
df[df['col'] > 100]  # 行过滤
df.query('col > 100 & region == "华北"')  # 查询语法
df.loc[0:10, ['a', 'b']]  # 按标签
df.iloc[0:10, 0:3]        # 按位置
```

### 数据变换
```python
df.rename(columns={'old': 'new'})
df['new_col'] = df['a'] + df['b']
df['category'] = df['value'].apply(lambda x: '高' if x > 100 else '低')
df['date'] = pd.to_datetime(df['date'])
df_grouped = df.groupby('category')['value'].sum().reset_index()
df_pivot = df.pivot_table(index='row', columns='col', values='val', aggfunc='sum')
```

### 合并数据
```python
pd.merge(df1, df2, on='key', how='left')      # SQL 式 JOIN
pd.concat([df1, df2], axis=0)                   # 纵向拼接
pd.concat([df1, df2], axis=1)                   # 横向拼接
```

### 时间序列
```python
df['month'] = df['date'].dt.to_period('M')
df['year'] = df['date'].dt.year
df['weekday'] = df['date'].dt.day_name()
df.set_index('date').resample('M')['value'].sum()  # 按月重采样
```
