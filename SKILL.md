---
name: data-analysis-workflow
description: >-
  从数据采集到可执行建议的端到端数据分析工作流。
  当用户需要以下操作时触发：爬取/抓取网页数据（包括使用 CloakBrowser 绕过反爬网站）、分析 Excel/CSV/JSON 数据、
  清洗脏数据集、将清洗后数据导入数据库（MySQL/PostgreSQL/SQLite/SQL Server）、
  执行统计分析、创建数据模型、构建带切片器的 Streamlit 交互式看板、
  或生成带图表和可视化的数据驱动报告。支持完整流水线：网页爬取
  （requests/BeautifulSoup/selenium/CloakBrowser）、数据清洗（Python pandas）、
  数据库导入（SQLAlchemy + pandas to_sql）、统计建模（假设检验、相关性分析）、
  交互式可视化（Streamlit + plotly 切片器）、数据驱动报告（python-docx）。
  支持中英文数据。本 skill 使用本地代理 127.0.0.1:7897 进行网络请求。
---

# 数据分析工作流 (Data Analysis Workflow)

从原始数据到可执行建议的端到端数据分析工作流。支持 **Streamlit、Python(pandas)、MySQL、plotly** 等多种工具协同完成数据清洗、建模、可视化和报告生成。

---

## 1. 核心原则

### 1.1 数据真实性原则（最高优先级）

**所有分析数据必须来源于真实爬取或用户提供的文件。绝对禁止在未经用户明确同意的情况下生成、编造、模拟数据。**

原因：数据分析的价值建立在数据真实性之上。如果数据是伪造的，整个分析报告毫无意义，甚至会误导用户做出错误决策。

#### 具体规则

1. **爬取阶段**：只使用爬虫实际获取到的数据。如果爬取到 18 条数据，就用 18 条做分析，不要补充到 74 条
2. **数据不足时**：如果爬取数据量太少（如少于 20 条），必须用 `AskUserQuestion` 询问用户：
   - 选项 A：用现有数据继续分析（告知数据量有限，分析结论可能不够全面）
   - 选项 B：换一种爬取策略重新爬取
   - 选项 C：用户手动提供补充数据文件
   - 选项 D：用户明确同意后，才可以生成基于真实市场结构的模拟数据（必须在报告中标注哪些是模拟数据）
3. **数据来源标注**：在最终报告中必须明确标注数据来源（"全部来自真实爬取" / "部分来自真实爬取+部分模拟" / "全部来自用户提供的文件"）
4. **绝不隐瞒**：如果某些字段无法爬取（如价格需要登录），必须告知用户，不要自行用假数据填充

#### 违规示例（绝对禁止）

```
# 禁止：爬取了 18 条数据后，自行补充 56 条"基于市场数据"的记录
# 禁止：在报告中说"分析了 74 款手机"但实际只有 18 条是真实爬取的
# 禁止：用随机数生成销量、评论数等字段
```

### 1.2 阶段确认制

每阶段完成后**必须暂停**，展示阶段摘要，用 `AskUserQuestion` 询问用户确认后再继续。

### 1.3 数据质量门禁

每两个阶段之间自动运行数据质量检查，检查不通过则暂停并告知用户。

### 1.4 遇到选择必须询问

任何需要做决策的地方（爬取策略、数据不足、工具选择等），都用 `AskUserQuestion` 询问用户，不要自行决定。

---

## 2. 项目文件夹结构（强制性规则）

所有阶段的产出文件必须按统一的目录结构组织：

```
{项目名称}/                      ← 顶层项目文件夹（用户指定或自动生成）
├── phase_0_scraping/            ← 阶段 0：数据爬取
│   ├── scraper.py               ← 爬虫脚本
│   ├── raw_data.xlsx            ← 原始爬取数据
│   └── 阶段0_爬取总结.xlsx       ← 阶段总结
├── phase_1_understanding/       ← 阶段 1：数据理解
│   └── 阶段1_数据理解总结.xlsx
├── phase_2_cleaning/            ← 阶段 2：数据清洗
│   ├── cleaned_data.xlsx        ← 清洗后数据
│   └── 阶段2_清洗报告.xlsx
├── phase_2.5_database/          ← 阶段 2.5：数据入库（可选）
│   └── 阶段2.5_入库报告.xlsx
├── phase_3_modeling/            ← 阶段 3：数据建模
│   └── 阶段3_建模结果.xlsx
├── phase_4_visualization/       ← 阶段 4：数据可视化
│   ├── dashboard.py             ← Streamlit 看板
│   ├── dashboard_*.html         ← 静态看板备份（plotly）
│   ├── chart_*.png              ← 静态图表（报告用）
│   └── 阶段4_可视化清单.xlsx
├── phase_5_analysis/            ← 阶段 5：分析与结论
│   └── 阶段5_分析结论.docx
├── phase_6_report/              ← 阶段 6：建议与报告
│   └── 最终分析报告.docx
└── README.md                    ← 项目说明（可选）
```

**规则**：
1. 在阶段 0 开始前，先用 `AskUserQuestion` 询问用户项目名称，创建顶层文件夹
2. 每个阶段的代码、数据、总结文件都放入对应的 phase 文件夹
3. 阶段间的数据传递通过文件路径引用（如阶段 2 读取阶段 0 的 `raw_data.xlsx`）
4. 如果用户已有数据文件，将其复制到 `phase_1_understanding/` 或 `phase_2_cleaning/` 中

---

## 3. 阶段执行流程

```
用户需求 → 创建项目文件夹 → 阶段 0（爬取）
    ↓
质量门禁 0 → 用户确认 → 阶段 1（理解）
    ↓
质量门禁 1 → 用户确认 → 阶段 2（清洗）
    ↓
质量门禁 2 → 用户确认 → 阶段 3（建模）[可选：阶段 2.5 入库]
    ↓
用户确认 → 阶段 4（可视化）
    ↓
用户确认 → 阶段 5（分析）
    ↓
用户确认 → 阶段 6（报告）
    ↓
用户确认 → 完成
```

**每阶段的标准流程**：
```
执行阶段任务
    ↓
产出阶段文件（代码 + 数据 + 总结）
    ↓
运行质量门禁（自动检查）
    ↓
展示阶段摘要（关键指标 + 产出文件 + 下一步建议）
    ↓
用 AskUserQuestion 询问用户：
  - 确认继续 → 进入下一阶段
  - 需要修改 → 回到当前阶段调整
  - 跳过后续阶段 → 结束工作流
```

**阶段摘要模板**：
```markdown
## 阶段 {N} 完成：{阶段名称}

### 执行结果
- {关键指标 1}
- {关键指标 2}
- {关键指标 3}

### 产出文件
- {文件路径}

### 下一步建议
- 进入阶段 {N+1}：{下一阶段名称}
- 建议操作：{具体说明}
```

---

## 4. 阶段详情

### 阶段 0：数据爬取（自动获取数据）

当用户没有现成数据文件，需要从网上获取数据时，执行此阶段。

#### 触发条件

- 用户描述了数据需求但没有提供数据文件
- 用户明确要求爬取某个网站的数据
- 用户说"帮我获取XX数据"、"爬取XX网站"等

#### 执行步骤

1. **需求解析** — 确定目标网站、数据字段、数据量
2. **选择爬取策略** — 用 `AskUserQuestion` 询问用户选择方案：
   - **API 接口**（最优先）：如果目标网站有公开 API，直接调用
   - **requests + BeautifulSoup**（次选）：静态页面，轻量快速
   - **CloakBrowser**（反爬首选）：有反爬检测的网站（京东、淘宝、拼多多等），使用反检测浏览器绕过
   - **selenium**（备选）：CloakBrowser 不可用时的降级方案
3. **生成爬虫代码** — 编写 Python 脚本，保存到 `phase_0_scraping/` 文件夹
4. **执行爬虫** — 运行脚本获取数据，保存为 `.xlsx` 或 `.csv`
5. **数据量检查** — 检查爬取结果：
   - 如果数据量充足（≥30 条）：继续
   - 如果数据量不足（<30 条）：**暂停**，用 `AskUserQuestion` 询问用户如何处理
6. **爬取报告** — 记录数据来源、字段、行数、执行状态

#### 登录检测与处理（强制性规则）

在爬取过程中，如果检测到以下**登录信号**，必须**立即暂停爬取**，用 `AskUserQuestion` 向用户报告并请求协助：

**登录信号检测**：
- 页面 URL 跳转到 `passport.*.com`、`login.*.com`、`*.com/login` 等登录页
- 页面内容包含 "登录查看"、"请先登录"、"登录后可查看" 等文字
- 页面返回 403/401 状态码
- 需要验证码（滑块、图形验证码等）
- 数据字段返回空值或 "登录查看" 占位符（如价格显示为 "登录查看价格"）

**检测到登录信号后的处理流程**：

```
爬取执行 → 检测到登录信号 → 暂停爬取 → 用 AskUserQuestion 报告问题
    ↓
提供以下选项：
    1. 提供 cookies/登录凭证（推荐，可获取完整数据）
    2. 换用不需要登录的页面/API
    3. 用户手动登录后导出数据文件
    4. 只爬取不需要登录的公开数据（可能不完整）
    5. 放弃爬取，用户自行准备数据
    ↓
用户选择 → 按用户选择继续执行
```

#### 数据不足处理（强制性规则）

当爬取数据量不足（<30 条）时，**绝对不能自行生成补充数据**。必须用 `AskUserQuestion` 询问用户：

```
问题：本次爬取只获取到 {N} 条数据，数据量较少可能影响分析的全面性。
请选择处理方式：
- 继续分析：用现有 {N} 条数据进行分析，结论可能不够全面（推荐）
- 重新爬取：换一种爬取策略尝试获取更多数据
- 补充数据：你提供额外的数据文件，我合并后一起分析
- 同意生成模拟数据：基于真实市场结构生成模拟数据，报告中会明确标注（需你确认）
```

#### 爬虫代码模板

##### 方案 A：requests + BeautifulSoup（静态页面）

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time
import random
import os

os.environ['HTTP_PROXY'] = 'http://127.0.0.1:7897'
os.environ['HTTPS_PROXY'] = 'http://127.0.0.1:7897'

HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
    'Accept-Language': 'zh-CN,zh;q=0.9',
}

def crawl_page(url):
    resp = requests.get(url, headers=HEADERS, timeout=15)
    resp.raise_for_status()
    soup = BeautifulSoup(resp.text, 'html.parser')
    items = []
    for item in soup.select('.item-selector'):
        data = {
            '字段1': item.select_one('.field1').text.strip() if item.select_one('.field1') else '',
            '字段2': item.select_one('.field2').text.strip() if item.select_one('.field2') else '',
        }
        items.append(data)
    return items

all_data = []
for page in range(1, 6):
    url = f'https://example.com/list?page={page}'
    try:
        items = crawl_page(url)
        all_data.extend(items)
        print(f"第{page}页: {len(items)}条")
    except Exception as e:
        print(f"第{page}页失败: {e}")
    time.sleep(random.uniform(2, 5))

df = pd.DataFrame(all_data)
df.to_excel('phase_0_scraping/raw_data.xlsx', index=False)
print(f"爬取完成: {len(df)}条")
```

##### 方案 B：selenium（动态页面）

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from webdriver_manager.chrome import ChromeDriverManager
import pandas as pd
import time

def create_driver():
    options = Options()
    options.add_argument('--headless=new')
    options.add_argument('--no-sandbox')
    options.add_argument('--disable-dev-shm-usage')
    options.add_argument('--disable-blink-features=AutomationControlled')
    options.add_argument('--window-size=1920,1080')
    options.add_argument('--user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36')
    options.add_experimental_option('excludeSwitches', ['enable-automation'])
    service = Service(ChromeDriverManager().install())
    driver = webdriver.Chrome(service=service, options=options)
    driver.execute_cdp_cmd('Page.addScriptToEvaluateOnNewDocument', {
        'source': 'Object.defineProperty(navigator, "webdriver", {get: () => undefined})'
    })
    return driver

driver = create_driver()
driver.get('https://example.com')
time.sleep(5)
for y in range(0, 5000, 500):
    driver.execute_script(f'window.scrollTo(0, {y})')
    time.sleep(0.3)
# 提取数据...
driver.quit()
```

##### 方案 C：API 调用（最优方案）

```python
import requests
import pandas as pd
import os

os.environ['HTTP_PROXY'] = 'http://127.0.0.1:7897'
os.environ['HTTPS_PROXY'] = 'http://127.0.0.1:7897'

def call_api(keyword, page=1):
    url = 'https://api.example.com/search'
    params = {'keyword': keyword, 'page': page, 'limit': 30}
    headers = {'Authorization': 'Bearer YOUR_TOKEN', 'Content-Type': 'application/json'}
    resp = requests.get(url, headers=headers, params=params, timeout=15)
    resp.raise_for_status()
    return resp.json()['data']

all_data = []
for page in range(1, 6):
    data = call_api('手机', page)
    all_data.extend(data)
    print(f"第{page}页: {len(data)}条")

df = pd.DataFrame(all_data)
df.to_excel('phase_0_scraping/raw_data.xlsx', index=False)
```

##### 方案 D：CloakBrowser（反爬网站首选）

CloakBrowser 是基于 Playwright 的反检测浏览器，通过源码级指纹伪装通过 30/30 检测测试。适用于京东、淘宝、拼多多等有强反爬机制的网站。依赖：`pip install cloakbrowser playwright`，首次使用需 `python -m playwright install chromium`。

```python
import sys
sys.stdout.reconfigure(encoding='utf-8')

import cloakbrowser
from cloakbrowser import CloakBrowser
import pandas as pd
import time
import random

def crawl_with_cloak(url, extract_func, pages=3):
    all_data = []
    browser = CloakBrowser(headless=True)
    try:
        for page_num in range(1, pages + 1):
            page_url = url.format(page=page_num)
            print(f"爬取第{page_num}页: {page_url}")
            tab = browser.new_page()
            try:
                tab.goto(page_url, timeout=30000)
                time.sleep(random.uniform(2, 4))
                for y in range(0, 5000, 500):
                    tab.evaluate(f'window.scrollTo(0, {y})')
                    time.sleep(0.3)
                items = extract_func(tab)
                all_data.extend(items)
                print(f"  获取 {len(items)} 条数据")
            except Exception as e:
                print(f"  第{page_num}页失败: {e}")
            finally:
                tab.close()
            time.sleep(random.uniform(1, 3))
    finally:
        browser.quit()
    return all_data
```

#### 反爬处理策略

当爬虫遇到反爬措施时，按以下顺序尝试。**但遇到需要登录的情况，必须暂停并询问用户，不能自行跳过或用假数据填充。**

| 问题 | 解决方案 |
|------|---------|
| IP 被封 | 使用代理池、增加请求间隔 |
| **需要登录** | **暂停！用 AskUserQuestion 询问用户提供 cookies 或选择其他方案** |
| 验证码 | **首选 CloakBrowser**（通过指纹伪装避免触发）；**如仍失败，暂停并询问用户** |
| JavaScript 渲染 | **首选 CloakBrowser**；降级用 selenium 或 playwright |
| 反爬检测（京东、淘宝等） | **CloakBrowser**（源码级指纹伪装，通过 30/30 检测） |
| 频率限制 | 增加随机延迟（2-10秒） |
| User-Agent 检测 | CloakBrowser 自动处理；或随机轮换 UA 字符串 |
| **数据字段显示"登录查看"** | **暂停！告知用户哪些字段缺失，询问是否提供 cookies 或只用公开数据** |

#### 质量门禁 0：爬取后验证

```python
def quality_gate_0(df, scraped_count):
    """爬取后的数据质量检查"""
    checks = {}
    checks["数据行数 ≥ 10"] = len(df) >= 10
    checks["无完全空行"] = df.dropna(how='all').shape[0] == len(df)

    key_fields = [col for col in df.columns if col in ['商品名称', '名称', 'title', '品牌', 'brand']]
    if key_fields:
        checks["关键字段非空率 ≥ 80%"] = df[key_fields].notna().mean().min() >= 0.8

    if scraped_count > 0:
        checks["爬取数据占比 ≥ 70%"] = scraped_count / len(df) >= 0.7

    failed = [k for k, v in checks.items() if not v]
    if failed:
        print(f"质量门禁 0 未通过: {', '.join(failed)}")
        return False
    print("质量门禁 0 通过")
    return True
```

#### 阶段 0 输出：爬取报告（推荐 .xlsx）

```python
from scripts.export_summary import export_phase_summary_xlsx

path = export_phase_summary_xlsx(
    phase_number=0,
    phase_name="数据爬取",
    summary_data={
        "overview": [
            {"维度": "数据来源", "值": "京东商城"},
            {"维度": "爬取方式", "值": "CloakBrowser"},
            {"维度": "数据行数", "值": len(df)},
            {"维度": "数据列数", "值": len(df.columns)},
        ],
        "details": [{"字段": col, "类型": str(df[col].dtype), "非空率": f"{df[col].notna().mean():.1%}"} for col in df.columns],
        "findings": [
            f"成功爬取 {len(df)} 条数据",
            f"数据包含 {len(df.columns)} 个字段",
            f"数据已保存为 phase_0_scraping/raw_data.xlsx",
        ],
    },
    output_dir="phase_0_scraping",
)
```

---

### 阶段 1：数据获取与理解

在开始任何分析之前，先理解数据是什么。

#### 操作步骤
1. **获取数据** — 读取用户提供的文件（Excel/CSV/JSON/数据库导出等）
2. **数据概览** — 检查数据形状、列名、数据类型、样本行
3. **质量初评** — 识别明显的问题（空值、异常值、格式不一致）
4. **与用户确认** — 总结数据概况并确认分析目标

#### 输出格式
```markdown
## 数据概览

| 维度 | 值 |
|------|-----|
| 行数 | 10,000 |
| 列数 | 15 |
| 日期范围 | 2025-01 至 2025-12 |
| 缺失值 | 3 列存在缺失 |

### 列信息
| 列名 | 类型 | 非空率 | 样例值 |
|------|------|--------|--------|
| order_date | datetime | 100% | 2025-01-15 |
| revenue | float | 95% | ¥1,234.56 |
| region | string | 100% | 华北, 华东 |

### 初步发现
- [列名] 存在 X% 的缺失值，需要处理
- [列名] 有 [N] 个疑似异常值
- 数据时间跨度为 [起] 至 [止]
```

#### 质量门禁 1：理解后验证

```python
def quality_gate_1(df, user_confirmed_types=True, outliers_identified=True, missing_strategy_set=True):
    """理解后的数据质量检查"""
    checks = {}
    checks["数据类型已确认"] = user_confirmed_types
    checks["异常值已标注"] = outliers_identified
    checks["缺失值策略已确定"] = missing_strategy_set

    failed = [k for k, v in checks.items() if not v]
    if failed:
        print(f"质量门禁 1 未通过: {', '.join(failed)}")
        return False
    print("质量门禁 1 通过")
    return True
```

#### 阶段 1 输出：阶段性总结（推荐 .xlsx）

```python
from scripts.export_summary import export_phase_summary_xlsx

path = export_phase_summary_xlsx(
    phase_number=1,
    phase_name="数据获取与理解",
    summary_data={
        "overview": [
            {"维度": "行数", "值": df.shape[0]},
            {"维度": "列数", "值": df.shape[1]},
            {"维度": "缺失值列数", "值": int(df.isnull().sum().gt(0).sum())},
        ],
        "details": [{
            "列名": col,
            "类型": str(df[col].dtype),
            "非空率": f"{df[col].notna().mean():.1%}",
            "唯一值": df[col].nunique(),
        } for col in df.columns],
        "findings": [
            f"数据包含 {df.shape[0]} 行，{df.shape[1]} 列",
            f"发现 {df.isnull().sum().sum()} 个缺失值，分布在 {int(df.isnull().sum().gt(0).sum())} 列中",
            f"共 {df.duplicated().sum()} 行重复数据",
        ],
    },
    output_dir="phase_1_understanding",
)
```

---

### 阶段 2：数据清洗

根据阶段 1 发现的问题，选择最合适的工具进行清洗。

#### 清洗任务清单

| 问题类型 | 推荐工具 | 方法 |
|---------|---------|------|
| 缺失值 | Python pandas | 填充(均值/中位数/前向填充)、删除 |
| 重复行 | Python pandas | `drop_duplicates()` |
| 异常值 | Python pandas | IQR 法、Z-score、业务规则过滤 |
| 格式不一致 | Python pandas | 统一日期格式、文本规范化、单位统一 |
| 数据类型错误 | Python pandas | `astype()`、`to_datetime()`、`to_numeric()` |

#### Python 数据清洗代码模板

```python
import pandas as pd
import numpy as np

df = pd.read_excel('phase_0_scraping/raw_data.xlsx')

# 1. 数据类型修正
for col in ['date_col']:
    df[col] = pd.to_datetime(df[col], errors='coerce')
for col in ['numeric_col']:
    df[col] = pd.to_numeric(df[col], errors='coerce')

# 2. 缺失值处理
for col in df.select_dtypes(include=[np.number]).columns:
    df[col] = df[col].fillna(df[col].median())
for col in df.select_dtypes(include=['object']).columns:
    df[col] = df[col].fillna(df[col].mode()[0] if not df[col].mode().empty else 'Unknown')

# 3. 重复行
df = df.drop_duplicates()

# 4. 异常值处理 (IQR 法)
def remove_outliers_iqr(df, column):
    Q1 = df[column].quantile(0.25)
    Q3 = df[column].quantile(0.75)
    IQR = Q3 - Q1
    lower = Q1 - 1.5 * IQR
    upper = Q3 + 1.5 * IQR
    outliers = df[(df[column] < lower) | (df[column] > upper)]
    print(f"{column}: 移除 {len(outliers)} 个异常值 ({len(outliers)/len(df)*100:.1f}%)")
    return df[(df[column] >= lower) & (df[column] <= upper)]

for col in ['revenue', 'quantity']:
    df = remove_outliers_iqr(df, col)

df.to_excel('phase_2_cleaning/cleaned_data.xlsx', index=False)
```

#### 质量门禁 2：清洗后验证

```python
def quality_gate_2(df_before, df_after):
    """清洗后的数据质量检查"""
    checks = {}
    checks["清洗后行数 ≥ 原始行数 50%"] = len(df_after) >= len(df_before) * 0.5
    checks["缺失值为 0"] = df_after.isnull().sum().sum() == 0
    checks["无重复行"] = df_after.duplicated().sum() == 0

    # 检查数值列没有 object 类型
    numeric_cols = df_after.select_dtypes(include=[np.number]).columns
    for col in numeric_cols:
        if df_after[col].dtype == 'object':
            checks[f"{col} 数据类型正确"] = False

    failed = [k for k, v in checks.items() if not v]
    if failed:
        print(f"质量门禁 2 未通过: {', '.join(failed)}")
        return False
    print("质量门禁 2 通过")
    return True
```

#### 阶段 2 输出：清洗报告（推荐 .xlsx）

```python
from scripts.export_summary import export_cleaning_report_xlsx

path = export_cleaning_report_xlsx(
    before={"行数": 10000, "缺失值": 342, "重复行": 120, "异常值": 28},
    after={"行数": 9850, "缺失值": 0, "重复行": 0, "异常值": 0},
    operations=[
        "修正了 2 列的数据类型（日期、数值）",
        "填充了 revenue 列的缺失值（中位数）",
        "删除了 120 行完全重复数据",
        "移除了 28 个异常值（IQR 法）",
    ],
    output_dir="phase_2_cleaning",
)
```

---

### 阶段 2.5：数据入库（可选）

数据清洗完成后，将干净数据导入数据库。当用户需要数据库查询或后续用 SQL 分析时执行。

#### 操作步骤

1. **确认数据库类型** — 用 `AskUserQuestion` 询问用户使用 MySQL / PostgreSQL / SQLite / SQL Server
2. **检查依赖** — 确认对应的 Python 驱动已安装
3. **创建数据库（如不存在）** — 先连接服务器，创建目标数据库
4. **导入数据** — 使用 pandas + SQLAlchemy 将 DataFrame 写入数据库
5. **设置主键与索引** — 为表添加自增主键、业务字段索引
6. **验证导入** — 查询行数，确认与清洗后数据一致

#### MySQL 导入模板

```python
import pandas as pd
from sqlalchemy import create_engine, text
import pymysql

MYSQL_HOST = "localhost"
MYSQL_PORT = 3306
MYSQL_USER = "root"
MYSQL_PASSWORD = "your_password"
MYSQL_DATABASE = "your_db"
TABLE_NAME = "your_table"

df = pd.read_excel("phase_2_cleaning/cleaned_data.xlsx")

# 创建数据库
conn = pymysql.connect(host=MYSQL_HOST, port=MYSQL_PORT,
                       user=MYSQL_USER, password=MYSQL_PASSWORD)
cursor = conn.cursor()
cursor.execute(f"CREATE DATABASE IF NOT EXISTS `{MYSQL_DATABASE}` "
               f"CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci")
conn.close()

# 导入数据
engine = create_engine(
    f"mysql+pymysql://{MYSQL_USER}:{MYSQL_PASSWORD}"
    f"@{MYSQL_HOST}:{MYSQL_PORT}/{MYSQL_DATABASE}?charset=utf8mb4"
)
df.to_sql(name=TABLE_NAME, con=engine, if_exists="replace", index=False, chunksize=50)

# 添加自增主键
with engine.connect() as conn:
    conn.execute(text(f"""
        ALTER TABLE `{TABLE_NAME}`
        ADD COLUMN `id` INT AUTO_INCREMENT PRIMARY KEY FIRST
    """))
    conn.commit()

result = pd.read_sql(f"SELECT COUNT(*) as cnt FROM `{TABLE_NAME}`", engine)
print(f"导入完成：{TABLE_NAME} 共 {result['cnt'].iloc[0]} 条记录")
```

#### 其他数据库连接字符串

```python
# PostgreSQL
engine = create_engine(f"postgresql+psycopg2://{USER}:{PWD}@{HOST}:{PORT}/{DB}")

# SQLite（本地文件，无需建库）
engine = create_engine(f"sqlite:///data.db")

# SQL Server
engine = create_engine(f"mssql+pyodbc://{USER}:{PWD}@{HOST}:{PORT}/{DB}?driver=ODBC+Driver+17+for+SQL+Server")
```

---

### 阶段 3：数据建模

根据分析目标，选择合适的建模方法。本阶段强调**统计学方法**的使用。

#### 建模方法选择

| 分析目标 | 方法 | 工具 |
|---------|------|------|
| 趋势分析 | 时间序列分解、移动平均 | Python statsmodels |
| 对比分析 | 分组聚合、同比环比 | Python pandas |
| **相关性分析** | **皮尔逊/斯皮尔曼相关系数** | **Python scipy** |
| **假设检验** | **t 检验、ANOVA、卡方检验** | **Python scipy** |
| 贡献度分析 | Pareto 分析、ABC 分类 | Python pandas |
| 预测建模 | 线性回归、ARIMA | Python sklearn / statsmodels |
| 分类/聚类 | K-means、决策树 | Python sklearn |

#### 统计分析代码模板

```python
import pandas as pd
import numpy as np
from scipy import stats

df = pd.read_excel('phase_2_cleaning/cleaned_data.xlsx')

# ===== 描述性统计 =====
def descriptive_stats(df, column):
    """计算描述性统计指标"""
    s = df[column].dropna()
    return {
        '均值': s.mean(),
        '中位数': s.median(),
        '标准差': s.std(),
        '最小值': s.min(),
        '最大值': s.max(),
        '25%分位数': s.quantile(0.25),
        '75%分位数': s.quantile(0.75),
        '偏度': s.skew(),
        '峰度': s.kurt(),
        '变异系数': s.std() / s.mean() if s.mean() != 0 else 0,
    }

# ===== 假设检验 =====
def two_group_test(df, group_col, value_col):
    """两组 t 检验"""
    groups = [g[value_col].dropna() for _, g in df.groupby(group_col)]
    if len(groups) != 2:
        return None
    stat, p = stats.ttest_ind(*groups[0], *groups[1])
    return {'t 统计量': stat, 'p 值': p, '显著': p < 0.05}

def anova_test(df, group_col, value_col):
    """多组 ANOVA 检验"""
    groups = [g[value_col].dropna() for _, g in df.groupby(group_col)]
    stat, p = stats.f_oneway(*groups)
    return {'F 统计量': stat, 'p 值': p, '显著': p < 0.05}

def chi_square_test(df, col1, col2):
    """卡方检验（分类变量独立性）"""
    contingency = pd.crosstab(df[col1], df[col2])
    stat, p, dof, expected = stats.chi2_contingency(contingency)
    return {'卡方统计量': stat, 'p 值': p, '自由度': dof, '显著': p < 0.05}

# ===== 相关性分析 =====
def correlation_matrix(df, method='pearson'):
    """计算相关性矩阵"""
    numeric_df = df.select_dtypes(include=[np.number])
    return numeric_df.corr(method=method)

def correlation_with_pvalues(df, col1, col2, method='pearson'):
    """计算相关系数及 p 值"""
    if method == 'pearson':
        r, p = stats.pearsonr(df[col1].dropna(), df[col2].dropna())
    else:
        r, p = stats.spearmanr(df[col1].dropna(), df[col2].dropna())
    return {'相关系数': r, 'p 值': p, '显著': p < 0.05}
```

#### 分组聚合模板

```python
# 分组聚合
summary = df.groupby('category').agg({
    'revenue': ['sum', 'mean', 'count'],
    'profit': 'sum'
}).round(2)

# 同比环比
df['month'] = df['date'].dt.to_period('M')
df['prev_month_revenue'] = df.groupby('category')['revenue'].shift(1)
df['mom_growth'] = (df['revenue'] - df['prev_month_revenue']) / df['prev_month_revenue'] * 100

# Pareto 分析
pareto = df.groupby('product')['revenue'].sum().sort_values(ascending=False).reset_index()
pareto['cum_pct'] = pareto['revenue'].cumsum() / pareto['revenue'].sum() * 100
pareto['category'] = pareto['cum_pct'].apply(
    lambda x: 'A' if x <= 80 else ('B' if x <= 95 else 'C')
)
```

#### 阶段 3 输出：阶段性总结（推荐 .xlsx）

```python
from scripts.export_summary import export_phase_summary_xlsx

path = export_phase_summary_xlsx(
    phase_number=3,
    phase_name="数据建模",
    summary_data={
        "overview": [
            {"分析方法": "分组聚合", "分组字段": "category", "指标": "revenue"},
            {"分析方法": "相关性矩阵", "维度数": 8},
            {"分析方法": "t 检验", "分组": "品牌 A vs B"},
        ],
        "details": [
            {"类别": "A", "总收入": 562500, "平均收入": 1250, "订单数": 450},
            {"类别": "B", "总收入": 375000, "平均收入": 938, "订单数": 400},
        ],
        "findings": [
            "类别 A 占总收入的 45%，是主要收入来源",
            "revenue 与 quantity 相关系数为 0.72，中等正相关",
            "t 检验显示品牌 A 和 B 的均价差异显著 (p<0.05)",
        ],
    },
    output_dir="phase_3_modeling",
)
```

---

### 阶段 4：数据可视化（Streamlit 看板）

本阶段生成**交互式 Streamlit 看板**，集成切片器、图表联动和统计分析。

#### 看板架构

```
dashboard.py
├── 侧边栏（切片器区域）
│   ├── 分类维度多选器（如品牌、类别）
│   ├── 数值范围滑块（如价格区间、好评率）
│   ├── 时间范围选择器（如有时间列）
│   └── 自定义维度选择器
├── 主区域（图表联动）
│   ├── 行 1：KPI 指标卡（4 个核心指标）
│   ├── 行 2：分布图（直方图 + 饼图）
│   ├── 行 3：对比图（柱状图 + 散点图）
│   └── 行 4：统计面板（描述性统计 + 假设检验 + 相关矩阵）
└── 数据表（可排序、可筛选、可下载）
```

#### Streamlit 看板完整模板

```python
"""
Streamlit 交互式数据分析看板
运行：streamlit run dashboard.py
"""
import streamlit as st
import pandas as pd
import numpy as np
import plotly.express as px
import plotly.graph_objects as go
from scipy import stats

# ===== 页面配置 =====
st.set_page_config(page_title="数据分析看板", layout="wide", page_icon="📊")
st.title("📊 数据分析看板")

# ===== 读取数据 =====
@st.cache_data
def load_data():
    return pd.read_excel('phase_2_cleaning/cleaned_data.xlsx')

df = load_data()

# ===== 侧边栏：切片器 =====
st.sidebar.header("🔍 数据筛选")

# 分类维度多选器
category_cols = df.select_dtypes(include=['object']).columns.tolist()
if category_cols:
    selected_cat_col = st.sidebar.selectbox("选择分类维度", category_cols)
    selected_categories = st.sidebar.multiselect(
        f"选择 {selected_cat_col}",
        df[selected_cat_col].unique().tolist(),
        default=df[selected_cat_col].unique().tolist()
    )
    df = df[df[selected_cat_col].isin(selected_categories)]

# 数值范围滑块
numeric_cols = df.select_dtypes(include=[np.number]).columns.tolist()
if numeric_cols:
    selected_num_col = st.sidebar.selectbox("选择数值维度", numeric_cols)
    min_val, max_val = float(df[selected_num_col].min()), float(df[selected_num_col].max())
    if min_val < max_val:
        selected_range = st.sidebar.slider(
            f"{selected_num_col} 范围",
            min_val, max_val, (min_val, max_val)
        )
        df = df[(df[selected_num_col] >= selected_range[0]) &
                (df[selected_num_col] <= selected_range[1])]

# 显示筛选后数据量
st.sidebar.markdown(f"**筛选后数据量**: {len(df)} 条")

# ===== 行 1：KPI 指标卡 =====
st.subheader("📈 核心指标")
col1, col2, col3, col4 = st.columns(4)

with col1:
    st.metric("总数据量", f"{len(df):,}")
with col2:
    if numeric_cols:
        st.metric(f"平均{numeric_cols[0]}", f"{df[numeric_cols[0]].mean():,.0f}")
with col3:
        st.metric(f"中位{numeric_cols[0]}", f"{df[numeric_cols[0]].median():,.0f}")
with col4:
        st.metric(f"标准差", f"{df[numeric_cols[0]].std():,.0f}")

# ===== 行 2：分布图 =====
st.subheader("📊 数据分布")
col_left, col_right = st.columns(2)

with col_left:
    # 直方图
    if numeric_cols:
        fig_hist = px.histogram(
            df, x=numeric_cols[0], nbins=30,
            title=f"{numeric_cols[0]} 分布",
            labels={numeric_cols[0]: numeric_cols[0]},
            color_discrete_sequence=['#2563EB']
        )
        fig_hist.update_layout(
            xaxis_title=numeric_cols[0],
            yaxis_title="频数",
            font=dict(family="SimHei, Microsoft YaHei, sans-serif")
        )
        st.plotly_chart(fig_hist, use_container_width=True)

with col_right:
    # 饼图
    if category_cols:
        cat_counts = df[category_cols[0]].value_counts().head(8)
        fig_pie = px.pie(
            values=cat_counts.values, names=cat_counts.index,
            title=f"{category_cols[0]} 分布",
            hole=0.4
        )
        st.plotly_chart(fig_pie, use_container_width=True)

# ===== 行 3：对比图 =====
st.subheader("📊 对比分析")
col_left2, col_right2 = st.columns(2)

with col_left2:
    # 柱状图
    if category_cols and numeric_cols:
        bar_data = df.groupby(category_cols[0])[numeric_cols[0]].mean().sort_values(ascending=True)
        fig_bar = px.bar(
            x=bar_data.values, y=bar_data.index,
            orientation='h',
            title=f"各{category_cols[0]}平均{numeric_cols[0]}",
            labels={'x': f"平均{numeric_cols[0]}", 'y': category_cols[0]},
            color_discrete_sequence=['#7C3AED']
        )
        st.plotly_chart(fig_bar, use_container_width=True)

with col_right2:
    # 散点图
    if len(numeric_cols) >= 2:
        fig_scatter = px.scatter(
            df, x=numeric_cols[0], y=numeric_cols[1],
            color=category_cols[0] if category_cols else None,
            title=f"{numeric_cols[0]} vs {numeric_cols[1]}",
            hover_data=[category_cols[0]] if category_cols else None
        )
        st.plotly_chart(fig_scatter, use_container_width=True)

# ===== 行 4：统计面板 =====
st.subheader("📐 统计分析")

# 描述性统计
with st.expander("描述性统计", expanded=True):
    if numeric_cols:
        desc_stats = df[numeric_cols].describe().T
        desc_stats['偏度'] = df[numeric_cols].skew()
        desc_stats['峰度'] = df[numeric_cols].kurt()
        desc_stats['变异系数'] = df[numeric_cols].std() / df[numeric_cols].mean()
        st.dataframe(desc_stats.round(4), use_container_width=True)

# 相关性矩阵
with st.expander("相关性矩阵", expanded=False):
    if len(numeric_cols) >= 2:
        corr_method = st.radio("相关系数类型", ['pearson', 'spearman'], horizontal=True)
        corr = df[numeric_cols].corr(method=corr_method)
        fig_corr = px.imshow(
            corr, text_auto='.2f',
            color_continuous_scale='RdBu_r',
            title=f"{corr_method.title()} 相关系数矩阵"
        )
        st.plotly_chart(fig_corr, use_container_width=True)

# 假设检验
with st.expander("假设检验", expanded=False):
    if category_cols and numeric_cols:
        test_group_col = st.selectbox("分组变量", category_cols, key='test_group')
        test_value_col = st.selectbox("检验变量", numeric_cols, key='test_value')

        groups = df[test_group_col].unique()
        if len(groups) == 2:
            g1 = df[df[test_group_col] == groups[0]][test_value_col].dropna()
            g2 = df[df[test_group_col] == groups[1]][test_value_col].dropna()
            stat, p = stats.ttest_ind(g1, g2)
            st.write(f"**独立样本 t 检验**: {groups[0]} vs {groups[1]}")
            st.write(f"- t 统计量: {stat:.4f}")
            st.write(f"- p 值: {p:.4f}")
            st.write(f"- 结论: {'差异显著 (p<0.05)' if p < 0.05 else '差异不显著 (p≥0.05)'}")
        elif len(groups) > 2:
            group_data = [df[df[test_group_col] == g][test_value_col].dropna() for g in groups]
            stat, p = stats.f_oneway(*group_data)
            st.write(f"**单因素 ANOVA**: {test_group_col}")
            st.write(f"- F 统计量: {stat:.4f}")
            st.write(f"- p 值: {p:.4f}")
            st.write(f"- 结论: {'组间差异显著 (p<0.05)' if p < 0.05 else '组间差异不显著 (p≥0.05)'}")

# ===== 数据表 =====
st.subheader("📋 数据明细")
st.dataframe(
    df.sort_values(numeric_cols[0], ascending=False) if numeric_cols else df,
    use_container_width=True,
    height=400
)

# 下载按钮
csv = df.to_csv(index=False).encode('utf-8-sig')
st.download_button("📥 下载筛选后数据", csv, "filtered_data.csv", "text/csv")
```

#### 运行方式

```bash
# 安装依赖
pip install streamlit plotly pandas scipy openpyxl

# 运行看板
streamlit run phase_4_visualization/dashboard.py
```

#### 静态图表（报告用）

当需要插入 Word/PDF 报告时，用 matplotlib 生成静态图片：

```python
import matplotlib.pyplot as plt
import matplotlib
matplotlib.rcParams['font.sans-serif'] = ['SimHei', 'Microsoft YaHei']
matplotlib.rcParams['axes.unicode_minus'] = False

fig, ax = plt.subplots(figsize=(10, 6))
ax.bar(categories, values, color="#2563EB")
ax.set_title("分类对比", fontsize=16)
ax.set_xlabel("类别")
ax.set_ylabel("数值")
plt.tight_layout()
plt.savefig("phase_4_visualization/chart.png", dpi=150, bbox_inches="tight")
plt.close()
```

#### 图表设计原则

1. **KPI 卡片放顶部** — 一眼看到核心指标
2. **图表从宏观到微观** — 上方放汇总，下方放明细
3. **颜色一致** — 全局使用统一色板
4. **交互提示完整** — 悬停显示中文标签和格式化数值
5. **标题是观点陈述** — 如"小米月销量领先"而非"月销量图表"
6. **中文标签** — 所有图表使用中文标题、坐标轴、提示文字

#### 阶段 4 输出：阶段性总结（推荐 .xlsx）

```python
from scripts.export_summary import export_phase_summary_xlsx

path = export_phase_summary_xlsx(
    phase_number=4,
    phase_name="数据可视化",
    summary_data={
        "overview": [
            {"图表名称": "价格分布直方图", "类型": "直方图", "工具": "plotly"},
            {"图表名称": "品牌销量饼图", "类型": "饼图", "工具": "plotly"},
            {"图表名称": "相关性矩阵", "类型": "热力图", "工具": "plotly"},
        ],
        "details": [
            {"图表": "价格分布", "说明": "显示价格集中在 1000-3000 元区间", "文件": "dashboard.py"},
            {"图表": "品牌对比", "说明": "小米月销量领先", "文件": "dashboard.py"},
        ],
        "findings": [
            "Streamlit 看板已生成，支持切片器联动",
            "集成描述性统计、假设检验、相关性分析",
            "所有图表支持交互式探索",
        ],
    },
    output_dir="phase_4_visualization",
)
```

---

### 阶段 5：分析与结论

综合前面阶段的成果，提炼核心发现。本阶段强调**数据驱动**的分析方式。

#### 分析框架

```
1. 总体概况 → 数据整体表现如何？（KPI 摘要）
2. 关键发现 → 最重要的 3-5 个发现是什么？（每个发现配图表）
3. 统计洞察 → 假设检验和相关性分析的关键结论
4. 异常识别 → 有什么值得关注的异常点？
5. 关联分析 → 哪些因素之间存在关联？（相关系数 + 显著性）
```

#### 阶段 5 输出：阶段性总结（推荐 .docx）

本阶段核心产出为**数据驱动的分析结论**，适合用 `.docx` 导出：

```python
from scripts.export_summary import export_phase_summary_docx

path = export_phase_summary_docx(
    phase_number=5,
    phase_name="分析与结论",
    summary_data={
        "overview": [
            {"指标": "总收入", "数值": "¥1,250,000", "环比": "+12.5%"},
            {"指标": "平均客单价", "数值": "¥328", "环比": "-3.2%"},
            {"指标": "总客户数", "数值": "3,850", "环比": "+8.1%"},
        ],
        "details": [
            {"发现编号": 1, "标题": "Q4 销售额显著增长", "类别": "趋势", "置信度": "高"},
            {"发现编号": 2, "标题": "类别A贡献45%收入", "类别": "贡献度", "置信度": "高"},
            {"发现编号": 3, "标题": "品牌均价差异显著", "类别": "假设检验", "置信度": "高"},
        ],
        "findings": [
            "Q4 销售额环比增长 23%，主要受双11活动驱动",
            "类别 A 贡献总收入的 45%，是核心业务",
            "t 检验显示品牌 A 和 B 的均价差异显著 (p<0.05)",
            "revenue 与广告投入强相关 (r=0.81, p<0.001)",
        ],
    },
    output_dir="phase_5_analysis",
)
```

---

### 阶段 6：建议与报告（数据驱动型）

基于分析结论，生成**数据驱动型**最终报告。图表为主，文字为辅，用数据说话。

#### 报告结构

```
1. 执行摘要（1 页）
   - 3-5 个核心数字（KPI 卡片风格）
   - 一句话结论

2. 关键发现（2-3 页）
   - 每个发现 = 1 张图表 + 1-2 句解读
   - 不超过 5 个发现
   - 图表占页面 70%，文字占 30%

3. 详细分析（3-5 页）
   - 品牌分析：品牌销量饼图 + 均价柱状图
   - 价格分析：价格分布直方图 + 价格段销量对比
   - 销量分析：Top 10 排名图 + 性价比散点图
   - 统计分析：相关矩阵 + 假设检验结果

4. 数据附录（1-2 页）
   - 完整数据表
   - 统计指标汇总
   - 数据来源说明
```

#### 图表规范

- 所有图表中文标签
- 标题是**观点陈述**（如"小米以 107 万月销量领先"而非"月销量图表"）
- 坐标轴有单位
- 颜色一致（使用统一色板）
- 关键数据标注在图表上
- 图片宽度占页面 90%

#### 建议结构

每个建议应包含：
- **问题**：基于什么发现
- **建议**：具体的行动方案
- **预期效果**：如果实施可能带来的改进
- **优先级**：高/中/低
- **难度**：易/中/难

#### 阶段 6 输出：最终报告（推荐 .docx）

```python
from scripts.export_summary import export_final_report_docx

path = export_final_report_docx(
    title="数据分析报告",
    executive_summary=(
        "本次分析覆盖了 10,000 条数据。"
        "核心发现：类别 A 贡献 45% 收入，Q4 销售额环比增长 23%。"
        "建议优先优化类别 A 的定价策略以最大化利润率。"
    ),
    metrics=[
        {"指标": "总收入", "数值": "¥1,250,000", "变化": "+12.5%"},
        {"指标": "平均客单价", "数值": "¥328", "变化": "-3.2%"},
    ],
    findings=[
        {"发现": "Q4 销售额增长 23%", "类别": "趋势",
         "说明": "主要由双11活动推动，11月销售额达到峰值"},
        {"发现": "品牌均价差异显著 (p<0.05)", "类别": "假设检验",
         "说明": "ANOVA 检验确认品牌间均价存在统计学显著差异"},
    ],
    recommendations=[
        {"优先级": "高", "建议": "优化类别A定价策略",
         "预期效果": "提升利润率 5%", "难度": "中"},
        {"优先级": "高", "建议": "加大 Q4 营销投入",
         "预期效果": "延续增长趋势", "难度": "易"},
    ],
    output_dir="phase_6_report",
)
```

---

## 5. 阶段总结导出脚本

本 skill 提供导出函数，每个函数有独立的 `.xlsx` 版和 `.docx` 版，**按需调用其一**：

```python
from scripts.export_summary import (
    export_phase_summary_xlsx,   # → .xlsx（适合表格数据）
    export_phase_summary_docx,   # → .docx（适合文字报告）
)

# 场景 A: 本阶段产出主要是表格数据 → 选 xlsx
path = export_phase_summary_xlsx(
    phase_number=1, phase_name="数据获取与理解",
    summary_data={"overview": [...], "details": [...], "findings": [...]},
    output_dir="phase_1_understanding",
)

# 场景 B: 本阶段产出主要是文字分析 → 选 docx
path = export_phase_summary_docx(
    phase_number=5, phase_name="分析与结论",
    summary_data={"overview": [...], "details": [...], "findings": [...]},
    output_dir="phase_5_analysis",
)
```

各阶段的函数对照表：

| 场景 | .xlsx 函数 | .docx 函数 |
|------|-----------|-----------|
| 阶段总结（通用） | `export_phase_summary_xlsx()` | `export_phase_summary_docx()` |
| 数据清洗报告 | `export_cleaning_report_xlsx()` | `export_cleaning_report_docx()` |
| 最终分析报告 | `export_final_report_xlsx()` | `export_final_report_docx()` |

---

## 6. 失败处理

### 通用处理流程

```
步骤执行 → 检测到问题/失败 → 暂停执行 → 向用户报告问题
    ↓
提供 2-4 个选择方案（每个方案说明利弊）
    ↓
用户选择 → 按用户选择继续执行
```

### 阶段 0 失败处理：数据爬取失败

**触发条件**：爬虫返回空数据、被反爬封锁、网络超时、目标网站结构变化、需要登录等。

**必须暂停并用 `AskUserQuestion` 向用户提供以下选择**：

1. **换用 CloakBrowser** — 使用反检测浏览器绕过反爬（推荐）
2. **换一种爬取策略** — 从 requests 切换到 selenium，或从 selenium 切换到 API 调用
3. **换一个数据源** — 如果目标网站无法爬取，建议从其他网站获取同类数据
4. **手动提供数据** — 让用户自行从网站下载数据文件（CSV/Excel），然后继续后续分析
5. **放弃爬取** — 跳过此阶段，用户自行准备数据后再开始分析

**注意：选项中不包含"使用模拟数据"。** 只有在用户主动提出希望生成模拟数据时，才可提供此选项，且必须在报告中明确标注。

### 阶段 1 失败处理：数据读取/理解失败

**触发条件**：文件格式不支持、编码错误、文件损坏、列名无法识别等。

**必须暂停并用 `AskUserQuestion` 提供以下选择**：

1. **尝试其他编码** — 如 UTF-8 失败，尝试 GBK/GB2312
2. **尝试其他格式** — 如 Excel 失败，尝试 CSV
3. **查看文件原始内容** — 读取文件前几行原始字节，帮助判断格式
4. **用户描述数据结构** — 让用户口头描述列名和数据类型，手动构建 DataFrame

### 阶段 2 失败处理：数据清洗异常

**触发条件**：清洗后数据量骤降、关键列缺失值过多、数据类型转换大量失败等。

**必须暂停并用 `AskUserQuestion` 提供以下选择**：

1. **调整清洗策略** — 如缺失值过多，改为删除列而非填充
2. **保留原始数据** — 不做清洗，直接用原始数据进行分析
3. **分批清洗** — 对不同列使用不同清洗策略
4. **回退重新检查** — 回到阶段 1 重新审视数据质量

### 阶段 2.5 失败处理：数据库导入失败

**触发条件**：数据库连接失败、权限不足、表已存在冲突等。

**必须暂停并用 `AskUserQuestion` 提供以下选择**：

1. **检查数据库配置** — 让用户确认 host/port/用户名/密码
2. **换一种数据库** — 如 MySQL 失败，尝试 SQLite（本地文件，无需服务器）
3. **跳过入库** — 直接用 pandas/Excel 继续后续分析
4. **手动导入** — 生成 SQL 脚本，让用户在 DataGrip 中手动执行

### 阶段 3 失败处理：建模失败

**触发条件**：数据不适合所选建模方法、依赖库未安装、计算结果无意义等。

**必须暂停并用 `AskUserQuestion` 提供以下选择**：

1. **换一种建模方法** — 如时间序列分析失败，改用简单的分组聚合
2. **简化分析维度** — 减少分析维度，聚焦核心指标
3. **安装缺失依赖** — 提示用户安装所需的 Python 库
4. **跳过建模** — 直接进入可视化阶段

### 阶段 4 失败处理：可视化失败

**触发条件**：Streamlit 启动失败、plotly 渲染失败、中文乱码等。

**必须暂停并用 `AskUserQuestion` 提供以下选择**：

1. **换一种图表类型** — 如散点图不适合，改用柱状图
2. **换一种工具** — 如 Streamlit 失败，改用 plotly 静态 HTML
3. **修复中文显示** — 安装中文字体或调整字体配置
4. **简化看板** — 减少图表数量，只展示核心指标

### 阶段 5-6 失败处理：报告生成失败

**触发条件**：docx 库未安装、模板不适用、内容生成异常等。

**必须暂停并用 `AskUserQuestion` 提供以下选择**：

1. **换一种格式** — 如 docx 失败，改用 Markdown 或纯文本
2. **简化报告内容** — 只包含核心发现和建议
3. **安装缺失依赖** — 提示用户安装 python-docx
4. **用户自行撰写** — 提供报告大纲和数据，让用户自己填写

### 通用规则

- **错误信息要具体**：不要说"出错了"，要说"爬取京东搜索页时触发了验证码验证，返回的页面标题是'京东验证'而非商品列表"
- **每个选择要说明利弊**：不要只列选项名称，要说明每个选项的优缺点和预期结果
- **推荐一个选项**：在列出选择后，说明你推荐哪个选项以及为什么
- **等待用户回复**：使用 `AskUserQuestion` 工具获取用户选择，不要自行决定
- **绝不自行生成数据**：无论任何原因，都不能在未经用户明确同意的情况下生成、编造、模拟数据
- **数据来源必须透明**：最终报告中必须清楚标注每条数据的来源

---

## 7. 工作流执行原则

### 7.1 适应性执行
- 并非每个项目都需要所有阶段。根据用户需求灵活调整
- 用户可能只有原始数据，需要完整流程；也可能已有清洗好的数据，从阶段 3 开始
- 阶段 2.5（数据入库）为可选步骤，当用户需要数据库查询或后续用 SQL 分析时执行
- 在每个阶段开始时，**先与用户确认目标**，避免做无用功

### 7.2 工具选择原则
- **用户已有工具优先**：如果用户明确说了用某个工具，就使用它
- **最适合原则**：没有明确指定时，选择最适合当前任务的工具
- **组合使用**：不同阶段可用不同工具，Python 清洗 → Streamlit 看板 → docx 报告

### 7.3 沟通原则
- **中文优先**：用户使用中文沟通时，报告和图表标签都使用中文
- **进度透明**：每个阶段完成后，总结做了什么、发现了什么、下一步建议做什么
- **出现问题及时告知**：数据质量差、缺失严重等问题需与用户协商处理方案
- **遇到选择必须询问**：任何需要做决策的地方，都用 `AskUserQuestion` 询问用户

### 7.4 可重复性原则
- 所有数据清洗、转换步骤应可重复执行
- Python 代码保存为 `.py` 文件
- 关键分析步骤记录处理逻辑

---

## 8. 引用的 skill 清单

本 skill 整合了以下专项 skill 的能力：

| Skill | 主要用途 | 调用时机 |
|-------|---------|---------|
| `data-analysis` | 通用数据分析、统计、报告模板 | 阶段 1/5/6 |
| `excel-automation` | Excel 自动化（xlwings 操作）| 阶段 2（Excel 数据） |
| `powerbi-modeling` | Power BI 语义模型、DAX 度量值 | 阶段 3（Power BI） |
| `data-visualization` | Python 可视化（matplotlib/seaborn/plotly）| 阶段 4 |
| `chart-visualization` | AntV 图表 API 生成 | 阶段 4（快速图表） |

## 代理配置

如需使用网络（如调用 API、下载数据等），使用本地代理：
```bash
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
```
