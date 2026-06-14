# data-analysis-workflow

Claude Code 数据分析工作流 skill。从原始数据采集到可执行建议的端到端分析流程。

## 功能特性

- **7 阶段工作流**：爬取 -> 理解 -> 清洗 -> 建模 -> 可视化 -> 分析 -> 报告
- **数据真实性保障**：未经用户明确同意，绝不生成虚假数据
- **交互式看板**：基于 Streamlit，支持切片器、KPI 指标卡和统计面板
- **质量门禁**：阶段间自动运行数据质量检查
- **统计分析集成**：描述性统计、假设检验（t 检验、ANOVA、卡方检验）、相关性分析
- **双格式导出**：结构化 `.xlsx` + 可读 `.docx` 报告

## 支持的工具

| 工具 | 用途 |
|------|------|
| Streamlit | 交互式看板（含切片器） |
| Python pandas | 数据清洗与转换 |
| plotly | 交互式图表 |
| scipy | 统计分析 |
| Selenium / CloakBrowser | 网页爬取 |
| MySQL / PostgreSQL / SQLite | 数据库集成 |
| python-docx | Word 报告生成 |

## 安装

```bash
claude install-skill github.com/sean2112/data-analysis-workflow
```

或手动克隆到你的 skill 目录：

```bash
git clone https://github.com/sean2112/data-analysis-workflow.git ~/.claude/skills/data-analysis-workflow
```

## 使用方式

在 Claude Code 中调用 skill：

```
/data-analysis-workflow 帮我分析这份销售数据
/data-analysis-workflow 爬取京东手机热销数据
/data-analysis-workflow 根据现有数据生成可视化看板
```

## 项目结构

运行分析时，skill 会创建标准化的项目文件夹：

```
{项目名称}/
├── phase_0_scraping/       # 数据采集
├── phase_1_understanding/  # 数据探查
├── phase_2_cleaning/       # 数据清洗
├── phase_2.5_database/     # 数据入库（可选）
├── phase_3_modeling/       # 统计分析
├── phase_4_visualization/  # Streamlit 看板 + 图表
├── phase_5_analysis/       # 分析结论
└── phase_6_report/         # 最终报告（.docx）
```

## Skill 文件结构

```
data-analysis-workflow/
├── SKILL.md              # 主要 skill 指令
├── scripts/
│   └── export_summary.py # xlsx/docx 导出工具
└── references/
    ├── crawler-patterns.md
    ├── mysql-queries.md
    └── python-data-cleaning.md
```

## 许可证

MIT
