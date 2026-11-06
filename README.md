# n8n 工作流合集

我在自建 n8n 实例上做的一批自动化工作流，逐个整理成可直接导入使用的形式。每个工作流一个 JSON 文件，配套的 Data Table 结构模板和说明都在本仓库里。

## 工作流列表

| # | 工作流 | 作用 | 依赖文件 |
| --- | --- | --- | --- |
| 1 | [AI 新闻日报](#1-ai-新闻日报) | 每日聚合 5 个技术源 → 抓正文 → 大模型生成双语简报 → 邮件推送 | `AI_News_Workflow.json`、`ai_news_candidates_template.csv` |

## 通用说明

- **环境要求**：支持 Data Tables 的 n8n 实例
- **仓库不含凭证**：所有 API Key、邮箱密码都需要你自己在 n8n 的 Credentials 里创建
- **导入方式**：画布右上角 `⋯` → **Import from File**
- **定时任务**：导入后必须发布/激活工作流，且 n8n 实例保持运行，否则定时不会触发

---

## 1. AI 新闻日报

每天定时抓取 5 个技术资讯源，抓取正文，用大模型生成双语简报，邮件推送；同时把文章写入去重表，避免重复发送。

### 文件说明

| 文件 | 说明 |
| --- | --- |
| `AI_News_Workflow.json` | n8n 工作流，导入即用 |
| `ai_news_candidates_template.csv` | 去重表 `ai_news_candidates` 的结构模板 |

### 前置准备

三个凭证（都在 n8n 的 Credentials 里自建）：

1. **Firecrawl** API Key —— 抓取网页正文
2. **SMTP** 邮箱账号 —— 发送邮件
3. **阿里云百炼** API Key —— 默认模型 `qwen3.8-flash`，可在 `Qwen Cloud Chat Model` 节点换其他模型

### 安装步骤

#### 1. 建去重表

新建 Data Table，名称 **`ai_news_candidates`**，共 13 列，类型必须完全一致：

| 列名 | 类型 |
| --- | --- |
| key | string |
| source | string |
| title | string |
| url | string |
| canonical_url | string |
| rss_excerpt | string |
| body | string |
| scrape_status | string |
| scrape_error | string |
| published_at | **date** |
| first_seen_at | **date** |
| scraped_at | **date** |
| sort_score | **number** |

> 也可以直接导入 `ai_news_candidates_template.csv`。但导入后请务必检查三个 date 列和 `sort_score` 是否被正确识别为日期/数字，CSV 导入的类型推断并不可靠；类型不对会在写库节点报错。

#### 2. 导入工作流

n8n 画布右上角 `⋯` → **Import from File**，选择 `AI_News_Workflow.json`。

#### 3. 重新选择数据表（最容易漏的一步）

导入后，工作流里的节点仍然指向原作者的表 ID。以下 **6 个节点**必须逐个打开，把 Data Table 重新选成你自己刚建的 `ai_news_candidates`：

- `If row does not exist`
- `If row does not exist1`
- `If row does not exist2`
- `If row does not exist3`
- `If row does not exist4`
- `Upsert row(s)`

只改表名不生效——节点记的是表 ID，必须重新选一次。

#### 4. 配置凭证

依次打开 `Scrape URL`、`Send an Email`、`Qwen Cloud Chat Model` 三个节点，分别挂上你自己的凭证。

#### 5. 改收发件人

`Send an Email` 节点的 `fromEmail` 和 `toEmail` 改成你自己的邮箱。

#### 6. 激活

工作流默认每天 **09:00** 触发。**必须发布/激活工作流，且 n8n 实例保持运行**，定时才会生效；实例关掉或休眠时不会执行。

### 首次运行

- 空表意味着没有任何去重记录，第一次运行会把当前所有符合条件的文章都发出来，属正常现象。
- 每篇文章抓取后等待 10 秒，整轮约 5–6 分钟，请耐心等待。
- 只有 `scrape_status = ok` 的文章会写入去重表；抓取失败的不会记录，下次运行会重试。

### 常见问题

**运行一开就报错，说列不存在或表找不到** → 第 3 步没做完，回到节点里重新选表。

**邮件发出去了但表里没数据** → 两个分支是并行执行的；如果邮件节点报错，写库分支可能还没轮到。先修邮件问题。

**同一批文章重复收到** → 去重表被清空或写入失败，检查第 1 步的列名和类型。

### 免责

RSS 源与抓取内容归原作者所有；本项目仅做聚合与摘要，请遵守各来源的使用条款。