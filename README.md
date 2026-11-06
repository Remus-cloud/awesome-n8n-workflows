# n8n 工作流合集

我在自建 n8n 实例上做的一批自动化工作流，逐个整理成可导入的模板。每个工作流有自己的子目录，里面放工作流 JSON 和配套文件。

## 工作流列表

| 工作流 | 作用 | 文件 |
| --- | --- | --- |
| [AI 新闻日报](#1-ai-新闻日报) | 从五个科技来源挑选新条目，用千问生成中英双语简报，通过邮件发送 | [工作流 JSON](./AI_News_Email/AI_News_Workflow.json) · [数据表 CSV 模板](./AI_News_Email/ai_news_candidates_template.csv) |

## 通用说明

在自己的 n8n 实例中导入 JSON，并在 **Credentials** 里配置自己的服务账号。先手动运行验证结果，再发布或启用工作流。使用本地 Docker 部署时，定时触发需要电脑保持唤醒，且 n8n 容器正在运行。

## 1. AI 新闻日报

工作流把当天选中的条目汇成一封主题为 **AI每日简报** 的 HTML 邮件。每个条目先给英文摘要，再给对应的中文摘要和原文链接。除了 AI 行业新闻，还会收录开发者关注的技术文章、开源项目和研究论文。采集阶段没有 AI 关键词白名单。

### 五个数据来源

| 来源 | 能读到什么 | 实际订阅及选取方式 | 每次上限 |
| --- | --- | --- | ---: |
| [Hacker News](https://news.ycombinator.com/) | 开发者社区讨论的技术文章、个人项目和行业话题 | 通过第三方 [HNRSS](https://hnrss.org/newest?points=100) 订阅最新投稿，设置 100 分门槛，按 Feed 中的顺序取条目 | 5 |
| [AIHOT](https://aihot.news/) | 中文 AI 行业动态，包括模型发布、产品更新和应用消息 | 使用 [AIHOT 自有 RSS](https://aihot.news/feed.xml)，按发布时间从新到旧，从条目中的“阅读原文”等链接找到抓取目标。导出的订阅 URL 带有 `aihot_actor` 参数，可换成自己的订阅地址 | 3 |
| [Techmeme](https://www.techmeme.com/) | 科技媒体对公司、产品和行业事件的报道 | 使用 [Techmeme 自有 RSS](https://www.techmeme.com/feed.xml)，按发布时间从新到旧，提取每条主标题所链接的媒体原文 | 3 |
| [GitHub Trending](https://github.com/trending) | 当天在开发者中受到关注的开源仓库 | 使用社区项目 [GitHubTrendingRSS](https://github.com/mshibanami/GitHubTrendingRSS) 的[每日全语言榜 RSS](https://mshibanami.github.io/GitHubTrendingRSS/daily/all.xml)，按榜单顺序选取，抓取仓库页面 | 3 |
| [Hugging Face Daily Papers](https://huggingface.co/papers) | 社区推荐和投票的研究论文 | 使用社区项目 [huggingface-daily-paper-feed](https://github.com/huangboming/huggingface-daily-paper-feed) 的[论文 RSS](https://raw.githubusercontent.com/huangboming/huggingface-daily-paper-feed/refs/heads/main/feed.xml)，按条目中的 `Upvotes` 降序。遇到 arXiv PDF 链接时改抓论文摘要页 | 2 |

Hacker News、AIHOT 和 Techmeme 会跳过**已知发布时间早于 72 小时**的条目。GitHub Trending 和 Daily Papers 按上榜顺序或票数选取，仓库创建时间、论文首次发表时间可以早于本期日报。

**自定义抓取篇数：** 按上表顺序，五个来源分别对应 `Limit`、`Limit1`、`Limit2`、`Limit3`、`Limit4`。打开想调整的节点，在 **Max Items** 填入该来源每次最多抓取的篇数。例如，把 `Limit` 从 5 改成 2，每次就最多选 2 条 Hacker News 文章。新条目不足或跨来源链接重复时，邮件里的篇数会少于上限总和。

### 从 RSS 到邮件

1. **选条目。** 一个 `Schedule Trigger` 启动五路 RSS。每路先整理链接、排序，再查历史表。已有相同 `canonical_url` 且 `scrape_status = ok` 的条目会被跳过，然后才应用该路的 `Limit`。按默认配额，五路合计每次最多选 **16 条**，首次运行也适用。
2. **合并、去重、抓取。** 五路合并后过滤空链接，并按 `canonical_url` 跨来源去重。比较网址时会忽略片段和常见跟踪参数，因此同一原文被多站收录时只抓一次。不同网址报道同一事件仍可能同时入选。Firecrawl 逐条抓网页，正常返回并整理后等待 10 秒，再处理下一条。
3. **生成日报。** `Package` 把本轮材料打成一份交给千问。材料包含网页正文或 RSS 摘要。正文材料合计最多约 40,000 字符，每条最多 6,000 字符。提示词要求标注摘要来源和截断情况。没有有效链接送入打包节点时，本轮不会生成邮件。
4. **发信、记历史。** `Markdown` 把日报转为 HTML，然后分出发信和写表两条线。在当前画布和 `v1` 执行顺序下，先执行位置较高的邮件分支，再执行写表分支。`Upsert row(s)` 只记录抓取状态为 `ok` 且有规范网址的条目，按 `canonical_url` 更新或新增。已收信但写表失败时，下次可能重复收到同一原文。[n8n 分支执行顺序](https://docs.n8n.io/flow-logic/execution-order/)

### 前置准备

- 使用支持 **Data Tables** 和 **Qwen Cloud Chat Model** 的 n8n 实例。
- 安装 Firecrawl 验证社区节点 `@mendable/n8n-nodes-firecrawl`。导出的 `Scrape URL` 使用这个节点包。[Firecrawl 安装说明](https://docs.firecrawl.dev/integrations/n8n)
- 准备 Firecrawl API Key、阿里云百炼 API Key 和可发信的 SMTP 账号。导出文件选用 `qwen3.8-flash`，也可以在模型节点选择自己账户可用的模型。

### 安装步骤

1. **建去重表。** 在目标项目的 **Data Tables** 中创建 `ai_news_candidates`，可导入[数据表模板](./AI_News_Email/ai_news_candidates_template.csv)，也可按下表手动建表。模板包含表头和空白占位行，导入后可清除空白行。核对 13 个自定义列的名称和类型，尤其是三个日期列和 `sort_score`。[n8n 数据表导入说明](https://docs.n8n.io/data/data-tables/)

   | 类型 | 列名 |
   | --- | --- |
   | String | `key`、`source`、`title`、`url`、`canonical_url`、`rss_excerpt`、`body`、`scrape_status`、`scrape_error` |
   | Date / DateTime | `published_at`、`first_seen_at`、`scraped_at` |
   | Number | `sort_score` |

2. **导入工作流。** 下载[工作流 JSON](./AI_News_Email/AI_News_Workflow.json)，在同一 n8n 项目新建工作流，从画布右上角 `⋯` 选择 **Import from File**。
3. **重新选择数据表。** 导出文件按实例内的表 ID 引用数据表。在 `If row does not exist` 以及后缀 `1` 至 `4` 的五个查询节点，再加上 `Upsert row(s)` 写入节点，共 **6 处**，分别重新选择刚创建的表。
4. **连接凭据、填写邮箱。** 在 `Scrape URL` 选择 Firecrawl 凭据，在 `Qwen Cloud Chat Model` 选择百炼凭据，在 `Send an Email` 选择 SMTP 凭据。将 `fromEmail` 改为 SMTP 账号允许使用的发件地址，将 `toEmail` 改为自己的收件地址。
5. **自定义邮件推送时间。** 打开 `Schedule Trigger`，在每天触发的规则中设置小时和分钟。默认每天 **09:00** 启动；例如想改为晚上 **20:30**，把小时设为 `20`、分钟设为 `30`。定时器默认沿用实例时区，北京时间可在工作流设置中选 `Asia/Shanghai`。日报日期也按北京时间生成。保存并发布更新后，新时间生效；邮件在当轮抓取和总结完成后发出。[定时触发器说明](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/)
6. **手动试跑后启用。** 点击 **Execute Workflow**。确认 `Package` 输出 1 个 item、`articles` 列出了本轮入选材料，检查双语邮件中的原文链接，并核对数据表是否写入了本轮抓取成功的网址，然后发布或启用工作流。试跑会实际调用服务、发送邮件并写表。想先少读几条，可临时把五个 `Limit` 都调为 1。

### 常见问题

- **某一路有候选，但没出日报。** 沿该路查看 `Normalize RSS`、历史查询和 `Limit` 的输出，再看五路 `Merge` 是否执行。该模板的五路查询未开启空输出占位，某一路提前结束时，要先确认合并节点收到了哪些输入。若全部是历史条目，没有新邮件属于预期。
- **邮件到了，表里却没新增记录。** 在执行详情中查看 `Code in JavaScript` 是否输出了 `ok` 条目，以及 `Upsert row(s)` 是否成功。若报找不到表或字段类型错误，检查上述 6 处表引用和 13 列类型。只靠 RSS 摘要入选的条目不会写入成功记录，后续仍有机会再次入选。
- **导入后提示 Agent 缺少工具。** 模板给 `AI Agent` 连接了千问模型，Tool 接口为空。若所用版本要求工具，可在该接口连接 `Think Tool` 后试跑。[n8n AI Agent 节点说明](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/)

### 内容来源

邮件保留原文链接，方便核对报道、仓库和论文。RSS 与抓取内容归各来源权利人所有，使用时请遵守各站点的条款。
