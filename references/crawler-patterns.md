# 爬虫模式参考

## 爬取策略选择

| 场景 | 推荐方案 | 依赖 |
|------|---------|------|
| 静态 HTML 页面 | requests + BeautifulSoup | `pip install beautifulsoup4 requests` |
| 有反爬检测的网站（京东、淘宝等） | **CloakBrowser**（反检测浏览器） | `pip install cloakbrowser playwright` + `python -m playwright install chromium` |
| 动态 JS 渲染页面 | CloakBrowser 或 selenium | `pip install cloakbrowser playwright` 或 `pip install selenium webdriver-manager` |
| 公开 API 接口 | requests 直接调用 | `pip install requests` |
| 需要登录的网站 | CloakBrowser 模拟登录 | `pip install cloakbrowser playwright` |
| 大规模爬取 | scrapy 框架 | `pip install scrapy` |

## 常见网站结构

### 电商网站（京东、淘宝等）
- 搜索页通常为 JS 动态渲染，需要 CloakBrowser 或 selenium
- 商品详情页可能有 API 接口可直接调用
- 反爬措施较强：验证码、IP封禁、频率限制、浏览器指纹检测
- **推荐使用 CloakBrowser**（反检测浏览器，通过 30/30 指纹检测）

### 新闻网站
- 多为静态 HTML，requests + BeautifulSoup 即可
- 注意分页结构（页码参数或无限滚动）

### 政府/公开数据
- 通常有开放数据平台，提供 API 或批量下载
- 数据格式可能是 CSV、JSON、XML

## 反爬处理详解

### 1. IP 封禁
```python
# 使用代理池
PROXIES = [
    'http://proxy1:port',
    'http://proxy2:port',
]
import random
proxy = random.choice(PROXIES)
resp = requests.get(url, proxies={'http': proxy, 'https': proxy})
```

### 2. User-Agent 检测
```python
USER_AGENTS = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
    'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36',
]
headers = {'User-Agent': random.choice(USER_AGENTS)}
```

### 3. 频率限制
```python
import time
import random

# 请求间随机延迟
time.sleep(random.uniform(2, 5))
```

### 4. 验证码处理
- 简单验证码：使用 OCR 库（pytesseract）
- 复杂验证码：使用第三方打码平台
- 滑动验证码：selenium 模拟人类操作

### 5. Cookies/Session
```python
session = requests.Session()
# 先访问首页获取 cookies
session.get('https://example.com')
# 再访问目标页
resp = session.get('https://example.com/data')
```

## CloakBrowser 常用操作（反检测浏览器）

CloakBrowser 是基于 Playwright 的反检测浏览器，通过源码级指纹伪装通过 30/30 检测测试。适用于京东、淘宝、拼多多等有强反爬机制的网站。

**安装**：`pip install cloakbrowser playwright` + `python -m playwright install chromium`

```python
from cloakbrowser import CloakBrowser
import time

# 创建浏览器实例
browser = CloakBrowser(headless=True)

# 新建标签页
page = browser.new_page()

# 访问页面
page.goto('https://search.jd.com/Search?keyword=手机', timeout=30000)
time.sleep(3)  # 等待加载

# 滚动加载更多内容
for y in range(0, 5000, 500):
    page.evaluate(f'window.scrollTo(0, {y})')
    time.sleep(0.3)

# CSS 选择器提取数据
items = page.query_selector_all('.gl-item')
for item in items:
    name = item.query_selector('.p-name a').text_content()
    price = item.query_selector('.p-price i').text_content()

# 获取页面标题
title = page.title()

# 获取当前 URL
url = page.url

# 获取页面文本
text = page.text_content('body')

# 执行 JavaScript
result = page.evaluate('document.title')

# 关闭标签页
page.close()

# 关闭浏览器
browser.quit()
```

### CloakBrowser vs selenium 对比

| 特性 | CloakBrowser | selenium |
|------|-------------|----------|
| 反检测能力 | 源码级指纹伪装，30/30 通过 | 容易被检测到 `navigator.webdriver` |
| 依赖 | playwright（自动管理浏览器） | webdriver-manager（需下载 ChromeDriver） |
| API 风格 | 类似 Playwright，简洁 | WebDriver API，较冗长 |
| 适用场景 | 有反爬检测的网站 | 无反爬或弱反爬的动态页面 |
| 安装复杂度 | 低（pip + playwright install） | 中（需匹配 Chrome 版本） |

## selenium 常用操作

```python
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

# 等待元素出现
element = WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.CSS_SELECTOR, '.target'))
)

# 点击按钮
driver.find_element(By.CSS_SELECTOR, '.btn').click()

# 输入文本
driver.find_element(By.ID, 'search').send_keys('关键词')

# 执行 JavaScript
driver.execute_script('window.scrollTo(0, 1000)')

# 获取属性
value = element.get_attribute('data-id')
```

## 数据保存格式

| 格式 | 适用场景 | 代码 |
|------|---------|------|
| Excel | 通用，中文友好 | `df.to_excel('data.xlsx', index=False)` |
| CSV | 大数据量，跨平台 | `df.to_csv('data.csv', index=False, encoding='utf-8-sig')` |
| JSON | 嵌套数据 | `df.to_json('data.json', orient='records', force_ascii=False)` |
| 数据库 | 持久化存储 | `df.to_sql('table', engine, index=False)` |
