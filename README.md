# n8n 工作流合集

我在自建 n8n 实例上做的一批自动化工作流，逐个整理成可导入的模板。每个工作流有自己的子目录，里面放工作流 JSON 和配套文件。

## 工作流列表

| 工作流 | 作用 | 文件 |
| --- | --- | --- |
| [AI 新闻日报](#1-ai-新闻日报) | 从五个科技来源挑选新条目，用千问生成中英双语简报，通过邮件发送 | [工作流 JSON](./AI_News_Email/AI_News_Workflow.json) · [数据表 CSV 模板](./AI_News_Email/ai_news_candidates_template.csv) |
| [每日职位筛选与投递清单](#2-每日职位筛选与投递清单) | 每天抓取香港的数据分析类岗位，按关键词与时间窗口初筛，用 DeepSeek 按简历偏好打分，达标的写入 Notion 并记为已推送 | [工作流 JSON](./job_digest/job_digest.json) · [岗位明细表 CSV](./job_digest/job_review_log.csv) · [已推送表 CSV](./job_digest/job_seen_jobs.csv) |

## 通用说明

在自己的 n8n 实例中导入 JSON，并在 **Credentials** 里配置自己的服务账号。先手动运行验证结果，再发布或启用工作流。使用本地 Docker 部署时，定时触发需要电脑保持唤醒，且 n8n 容器正在运行。

## 1. AI 新闻日报

工作流把当天选中的条目汇成一封主题为 **AI每日简报** 的 HTML 邮件。每个条目先给英文摘要，再给对应的中文摘要和原文链接。收集AI 行业新闻及开发者关注的技术文章、开源项目和研究论文。

### 五个数据来源

| 来源 | 能读到什么 | 实际订阅及选取方式 | 每次上限 |
| --- | --- | --- | ---: |
| [Hacker News](https://news.ycombinator.com/) | 开发者社区讨论的技术文章、个人项目和行业话题 | 通过第三方 [HNRSS](https://hnrss.org/newest?points=100) 订阅最新投稿，设置 100 分门槛，按 Feed 中的顺序取条目 | 5 |
| [AIHOT](https://aihot.news/) | 中文 AI 行业动态，包括模型发布、产品更新和应用消息 | 使用 [AIHOT 自有 RSS](https://aihot.news/feed.xml)，按发布时间从新到旧，从条目中的“阅读原文”等链接找到抓取目标。导出的订阅 URL 带有 `aihot_actor` 参数，可换成自己的订阅地址 | 3 |
| [Techmeme](https://www.techmeme.com/) | 科技媒体对公司、产品和行业事件的报道 | 使用 [Techmeme 自有 RSS](https://www.techmeme.com/feed.xml)，按发布时间从新到旧，提取每条主标题所链接的媒体原文 | 3 |
| [GitHub Trending](https://github.com/trending) | 当天在开发者中受到关注的开源仓库 | 使用社区项目 [GitHubTrendingRSS](https://github.com/mshibanami/GitHubTrendingRSS) 的[每日全语言榜 RSS](https://mshibanami.github.io/GitHubTrendingRSS/daily/all.xml)，按榜单顺序选取，抓取仓库页面 | 3 |
| [Hugging Face Daily Papers](https://huggingface.co/papers) | 社区推荐和投票的研究论文 | 使用社区项目 [huggingface-daily-paper-feed](https://github.com/huangboming/huggingface-daily-paper-feed) 的[论文 RSS](https://raw.githubusercontent.com/huangboming/huggingface-daily-paper-feed/refs/heads/main/feed.xml)，按条目中的 `Upvotes` 降序。遇到 arXiv PDF 链接时改抓论文摘要页 | 2 |

Hacker News、AIHOT 和 Techmeme 会跳过**已知发布时间早于 72 小时**的条目。GitHub Trending 和 Daily Papers 按上榜顺序或票数选取，仓库创建时间、论文首次发表时间可以早于本期日报。

**自定义抓取篇数：** 按上表顺序，五个来源分别对应 `Limit`、`Limit1`、`Limit2`、`Limit3`、`Limit4`。打开想调整的节点，在 **Max Items** 可自定义该来源每次最多抓取的篇数。新条目不足或跨来源链接重复时，邮件里的篇数会少于上限总和。

### 从 RSS 到邮件

1. **选条目。** 一个 `Schedule Trigger` 启动五路 RSS。每路先整理链接、排序，再查历史表。已有相同 `canonical_url` 且 `scrape_status = ok` 的条目会被跳过，然后才应用该路的 `Limit`。默认五路合计每次最多选 **16 条**，可自行更改。
2. **合并、去重、抓取。** 五路合并后过滤空链接，并按 `canonical_url` 跨来源去重。比较网址可避免同一原文被多站收录时被抓多次。Firecrawl 逐条抓网页，正常返回并整理后等待 10 秒，再处理下一条。
3. **生成日报。** `Package` 把本轮材料打成一份交给LLM。材料包含网页正文或 RSS 摘要。正文材料合计最多约 40,000 字符，每条最多 6,000 字符。提示词要求标注摘要来源和截断情况。没有有效链接送入打包节点时，本轮则不会生成邮件。
4. **发信、记历史。** `Markdown` 把日报转为 HTML，然后分出发信和写表两条线。在当前画布和 `v1` 执行顺序下，先执行位置较高的邮件分支，再执行写表分支。`Upsert row(s)` 只记录抓取状态为 `ok` 且有规范网址的条目，按 `canonical_url` 更新或新增。[n8n 分支执行顺序](https://docs.n8n.io/flow-logic/execution-order/)

### 前置准备

- 安装 Firecrawl 验证社区节点 `@mendable/n8n-nodes-firecrawl`。导出的 `Scrape URL` 使用这个节点包。[Firecrawl 安装说明](https://docs.firecrawl.dev/integrations/n8n)
- 准备 Firecrawl API Key、任意LLM API Key 和可发信的 SMTP 账号。默认配置选用 `qwen3.8-flash`，也可以在模型节点选择自己账户可用的模型。

### 安装步骤

1. **建去重表。** 在目标项目的 **Data Tables** 中创建 `ai_news_candidates`，可导入[数据表模板](./AI_News_Email/ai_news_candidates_template.csv)，也可按下表手动建表。模板包含表头和空白占位行，导入后可清除空白行。[n8n 数据表导入说明](https://docs.n8n.io/data/data-tables/)

   | 类型 | 列名 |
   | --- | --- |
   | String | `key`、`source`、`title`、`url`、`canonical_url`、`rss_excerpt`、`body`、`scrape_status`、`scrape_error` |
   | Date / DateTime | `published_at`、`first_seen_at`、`scraped_at` |
   | Number | `sort_score` |

2. **导入工作流。** 下载[工作流 JSON](./AI_News_Email/AI_News_Workflow.json)，在同一 n8n 项目新建工作流，从画布右上角 `⋯` 选择 **Import from File**。
3. **重新选择数据表。** 导出文件按实例内的表 ID 引用数据表。在 `If row does not exist` 以及后缀 `1` 至 `4` 的五个查询节点，再加上 `Upsert row(s)` 写入节点，共 **6 处**，分别重新选择刚创建的表。
4. **连接凭据、填写邮箱。** 在 `Scrape URL` 选择 Firecrawl 凭据，在 `Qwen Cloud Chat Model` 选择api凭据，也可更改为任意model节点，在 `Send an Email` 选择 SMTP 凭据。将 `fromEmail` 改为 SMTP 账号允许使用的发件地址，将 `toEmail` 改为自己的收件地址。
5. **自定义邮件推送时间。** 打开 `Schedule Trigger`，在每天触发的规则中设置小时和分钟。默认每天 **09:00** 启动（可自行修改时区与时间）。[定时触发器说明](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/)
6. **手动试跑后启用。** 点击 **Execute Workflow**。确认 `Package` 输出 1 个 item、`articles` 列出了本轮入选材料，检查双语邮件中的原文链接，并核对数据表是否写入了本轮抓取成功的网址，然后发布或启用工作流。试跑会实际调用服务、发送邮件并写表。若想先少读几条，可临时把五个 `Limit` 都调为 1。

### 常见问题

- **某一路有候选，但没出日报。** 沿该路查看 `Normalize RSS`、历史查询和 `Limit` 的输出，再看五路 `Merge` 是否执行。该模板的五路查询未开启空输出占位，某一路提前结束时，要先确认合并节点收到了哪些输入。若全部是历史条目，没有新邮件属于预期。
- **邮件到了，表里却没新增记录。** 在执行详情中查看 `Code in JavaScript` 是否输出了 `ok` 条目，以及 `Upsert row(s)` 是否成功。若报找不到表或字段类型错误，检查上述 6 处表引用和 13 列类型。只靠 RSS 摘要入选的条目不会写入成功记录，后续仍有机会再次入选。
- **导入后提示 Agent 缺少工具。** 模板给 `AI Agent` 连接了Qwen模型，Tool 接口为空。若所用版本要求工具，可在该接口连接 `Think Tool` 后试跑。[n8n AI Agent 节点说明](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/)

### 内容来源

邮件保留原文链接，方便核对报道、仓库和论文。RSS 源与抓取内容归原作者所有，本项目仅做聚合与摘要，请遵守各来源的使用条款。

## 2. 每日职位筛选与投递清单

工作流每天定时抓取一次香港的数据分析类岗位，先按关键词、排除词和发布时间窗口做初筛，再用 DeepSeek 按你的简历与偏好逐条打分。达到分数门槛的岗位按分数从高到低写进 Notion 职位库，并记入已推送表，之后不再重复推送。明细表记录本轮有职位 ID 的岗位及其筛选结果。有评分的显示分数，被初筛淘汰的显示原因。

Notion 职位库累积保存历次入选的岗位，方便继续跟踪投递状态。`job_seen_jobs` 保存跨轮去重所用的链接，`job_review_log` 用于查看本轮岗位明细和运行提示。

### 一份设置控制整轮

整轮筛选的所有条件都放在 `筛选条件（在此修改）` 这一个节点里，用中文名标注，改完下一个周期就生效，不需要动其它节点。

| 设置项 | 默认值 | 作用 |
| --- | --- | --- |
| `搜索词` | `data analyst hong kong` | 传给 JSearch 的检索词，决定这一轮从哪个方向捞岗位 |
| `地区代码` | `hk` | 检索的地区限定，按 ISO 国家代码填写（e.g., us、cn、de、gb、jp） |
| `语言` | `en` | 结果的语言倾向 |
| `最近几个日历日` | `7` | 按日历日（含今天）过滤发布时间可解析的岗位 |
| `最低匹配分` | `60` | 打分低于这个值的岗位不会进入最终清单 |
| `最多输出几条` | `30` | 最终清单最多留几条 |
| `抓取页数（每页10条）` | `3` | 每页最多 10 条，填 3 表示本轮最多抓 30 条 |
| `关注关键词` | 见节点 | 在标题、公司名和描述中匹配，填了关键词时至少要命中一个，逗号分隔 |
| `排除词` | 见节点 | 只匹配岗位标题，命中即淘汰，例如实习、销售或外包岗位 |
| `公司黑名单` | 空 | 只匹配公司名，命中即淘汰，逗号分隔 |
| `简历与偏好` | 见节点 | 逐条送给模型作为打分依据，写经验、技能、行业偏好、薪资期望和明确不考虑的方向 |
| `时区` | `Asia/Shanghai` | 发布时间的显示口径，按标准 IANA 时区名填写（e.g., Asia/Shanghai、Asia/Hong_Kong、America/New_York、Europe/London），也决定「日历日」从哪天开始算 |

这三类词共用一套写法。英文按完整单词或连续短语匹配，忽略大小写，短语末词支持常见单复数，例如 `data analyst` 能匹配 `Data-Analysts`。中文按包含匹配。用 `intern|internship` 可以列出多个可选词，用 `recruit*` 可以匹配 `recruiter` 和 `recruitment`。保留默认的 `intern|internship` 可避开 `International`，改成 `intern*` 则会把它也匹配进去。

匹配按文字执行：标题 `Sales Operations Analyst` 会命中排除词 `sales`。关注词的英文短语是在标题、公司名和描述拼接后匹配的，可能跨字段命中；`C++`、`C#` 等带标点的英文词都会按 `C` 处理。

`抓取页数` 与 `最多输出几条` 是两个独立设置：前者决定这一轮捞多少条，后者决定最后留几条。抓得多、只输出少数几条是常规用法——池子大了，能过门槛的才会多。

### 抓取范围与发布时间

工作流根据 `最近几个日历日`，在 JSearch 请求中使用下列四档 `date_posted`，再按配置时区比较已知的发布日期。没有可比较的发布日期时，这项筛选会放行。

| 最近几个日历日 | 请求使用的档位 | 本地保留 |
| ---: | --- | --- |
| 1 | `today` | 今天发布的 |
| 2–3 | `3days` | 最近 2–3 个日历日 |
| 4–7 | `week` | 最近 4–7 个日历日 |
| 8–30 | `month` | 最近 8–30 个日历日 |

工作流按页请求，每页最多 10 条，`抓取页数` 最多 10 页（100 条）。搜索结果不足页数上限时，返回有多少条就算多少条，不会补足。

### 从搜索到投递清单

1. **读设置。** `筛选条件（在此修改）` 输出本轮原始设置，`生效筛选条件` 把它解析成下游直接可用的值：检索参数、档位、窗口起点日期、分数门槛、输出条数、页数、关键词与排除词数组、时区，以及配置提醒。部分数值会回落到默认值或被限制在允许范围内；节点输出的 `warnings` 会写入明细表。
2. **抓取。** `搜索 JSearch 职位` 带着检索词、地区、语言、档位和页数请求接口，超时 60 秒，失败自动重试。仍然失败时走异常支路，在明细表写一行 `运行异常`，写明错误原文，本轮不会假装成功。
3. **清理旧结果。** 搜索成功后先删除明细表中 `job_id` 非空的旧岗位行，再处理本轮结果，返回 0 条时也一样。搜索报错则跳过清理，保留旧岗位行并追加 `运行异常`。
4. **初筛。** `归一化并初筛` 把接口返回的字段整理成统一结构，并逐条判定。先跳过没有职位 ID 的记录。其余按「同批 ID 重复 → 关注关键词 → 标题排除词 → 公司黑名单 → 发布时间窗口」逐项检查，以第一个不通过的条件作为淘汰原因。
5. **去重。** `只留通过初筛` 只放行通过初筛的岗位；`剔除已推送职位` 用链接去查已推送表，之前推送过的岗位这一轮直接跳过。
6. **打分。** `DeepSeek 匹配打分` 把公司、职位、地点、发布时间、薪资和职位描述，连同 `简历与偏好` 一起交给模型，要求只回一个 JSON：`score` 是 0 到 100 的整数，`reasons` 用中文说明为什么匹配且必须引用职位描述里的具体证据，`gaps` 指出缺口或风险。调用按每批 3 条、间隔 0.5 秒分批进行。
7. **排序与输出。** `汇总评分结果` 解析分数，丢掉低于 `最低匹配分` 的，按分数降序排，再截取 `最多输出几条`，并为每条拼好一段 Markdown 正文（公司、地点、发布时间、匹配分、匹配理由、缺口与风险、薪资、原文投递链接）。
8. **写入。** 达标的岗位在 Notion 职位库里建页，标题是「公司 · 职位」，正文是上面那段 Markdown，同时填好结构化属性；紧接着写入已推送表，作为下一轮去重的依据。这两步在同一分支且顺序固定，先建页再记录。
9. **留档。** 另一条支路把本轮有职位 ID 的岗位写进明细表，包括被淘汰的：每条记录带 `stage`（进了哪一步、结果如何）、`drop_reason`（没进清单的具体原因）、`match_score`、`passed_min_score`、`reasons`、`gaps`。这条支路从初筛后面分出，所以即使一条都没通过、一条都没抓到，明细表里也会有内容。

### 明细表里的 stage

| stage | 含义 |
| --- | --- |
| `入选` | 通过初筛和去重，分数与排名进入本轮清单；实际建页结果可在 `写入 Notion 职位库` 节点查看 |
| `分数达标但超出输出条数` | 分数够，但按分数排序后排在 `最多输出几条` 之外 |
| `未达分数线` | 打分低于 `最低匹配分`，`drop_reason` 里写着实际得分与门槛 |
| `初筛淘汰` | 同批职位 ID 重复，或被关键词、排除词、公司黑名单、发布时间窗口挡掉，原因见 `drop_reason` |
| `已推送过，本轮跳过` | 已推送表里已有同一链接的岗位 |
| `打分未完成` | 模型没有返回可解析的分数 |
| `本轮无结果` | 搜索成功，但没有带有效职位 ID 的岗位可处理 |
| `配置提醒` | 配置存在回落、取值限制或需要留意的组合，具体说明见 `drop_reason` |
| `运行异常` | 搜索接口调用失败，写明错误原文 |

### 两张数据表

| 类型 | `job_review_log`（明细） | `job_seen_jobs`（已推送） |
| --- | --- | --- |
| String | `job_id`、`title`、`company`、`location`、`posted_at`、`url`、`stage`、`drop_reason`、`reasons`、`gaps` | `job_id`、`title`、`company`、`location`、`url` |
| Number | `match_score` | `match_score` |
| Boolean | `passed_min_score` | — |
| Date / DateTime | `run_at` | `posted_at`、`first_seen_at` |

`本轮无结果`、`配置提醒`、`运行异常` 的 `job_id` 为空，会保留历次记录。查看最新一次提示时按 `run_at` 筛选。已推送表只增不清，用于按链接跨轮去重。

### 前置准备

- 一个 [JSearch（OpenWebNinja）](https://www.openwebninja.com/api/jsearch) 账号与 API Key，账号需处于已订阅状态。
- 一个 DeepSeek API 凭据。模型节点默认选 `deepseek-v4-pro`，也可以换成自己账户可用的模型。
- 一个 Notion 集成凭据，以及一个**已把该集成加为连接**的目标数据库，库里需要下列属性。集成没有加进数据库时，建页会因权限失败。

  | 类型 | 属性名 |
  | --- | --- |
  | Title | `岗位` |
  | Rich text | `公司`、`匹配理由`、`缺口/风险`、`地点`、`job_id` |
  | Number | `匹配分` |
  | Date | `发布时间`、`抓取日期` |
  | URL | `链接` |
  | Select | `状态`——工作流不写这一列，留给你自己标记投递进度 |

### 安装步骤

1. **建两张数据表。** 在目标项目的 **Data Tables** 中按上面的列类型分别创建 `job_review_log` 和 `job_seen_jobs`。[n8n 数据表说明](https://docs.n8n.io/data/data-tables/)
2. **导入工作流。** 下载[工作流 JSON](./job_digest/job_digest.json)，在同一 n8n 项目新建工作流，从画布右上角 `⋯` 选择 **Import from File**。
3. **重新选择数据表。** 导出文件按实例内的表 ID 引用数据表，共 **5 处**需要重新选择：写明细的 `清空上轮岗位明细`、`写入岗位明细记录`、`写入运行异常` 三处选 `job_review_log`；查已推送和记已推送的 `剔除已推送职位`、`记录已推送职位` 两处选 `job_seen_jobs`。
4. **连接凭据。** 在 `搜索 JSearch 职位` 选择 JSearch 的 API Key 凭据（请求头 `x-api-key`），在 `DeepSeek Chat` 选择 DeepSeek 凭据，在 `写入 Notion 职位库` 选择 Notion 凭据。
5. **重新选择 Notion 数据源。** 在 `写入 Notion 职位库` 的 **Data Source** 中选择自己的职位表；使用 **By ID** 时，从 Notion 的 **Manage data sources → ••• → Copy data source ID** 复制数据源 ID。核对标题列 `岗位` 和另外 9 个写入属性的名称、类型。`状态` 可选，留给你手动标记。
6. **填写筛选条件。** 打开 `筛选条件（在此修改）`，把 `简历与偏好` 换成自己的真实背景与偏好，按需调整搜索词、日历日、最低分、抓取页数和各类关键词。
7. **设置执行时间。** 打开 `Trigger`，在每天触发的规则里设置小时与分钟（默认 18:00，按实例时区计算）。这个时间只决定什么时候跑；发布时间的换算口径由 `时区` 设置决定。[定时触发器说明](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/)
8. **手动试跑后发布。** 点击 **Execute Workflow**，确认 `生效筛选条件` 输出的检索参数、档位、窗口起点日期都符合预期，再打开明细表看每条的 `stage` 与 `drop_reason` 是否符合自己的判断，最后核对 Notion 里新建的页面属性是否都填上了，然后发布或启用工作流。试跑会实际调用接口、消耗打分额度、真实写入 Notion 与两张表。若只想先看流程，可把 `抓取页数` 临时调为 1。

### 常见问题

- **最终清单比「最多输出几条」少很多。** 这个数字是上限，不是配额。实际留下几条取决于抓到的池子里有多少条同时通过了初筛、没被推送过、且分数达到门槛。打开明细表按 `stage` 一看就清楚：`初筛淘汰` 说明关键词或时间窗口收得太紧，`未达分数线` 说明分数门槛偏高或缺的是简历侧的匹配度，`分数达标但超出输出条数` 说明该调大输出条数。
- **抓回来的条数不到「抓取页数」乘 10。** 接口按相关度返回，翻到没有结果就提前结束，返回多少算多少；搜索词太窄时尤其明显。想扩大池子，可以放宽搜索词或调大 `最近几个日历日`。
- **一个眼熟的岗位这几轮都不见了。** 它应该已经在已推送表里，去重按链接判断，命中后不再重复推送。查 `job_seen_jobs` 的 `url` 可以确认。
- **怎样区分没抓到职位和全部被筛掉？** 按 `run_at` 找到本轮记录。没有可处理的职位时会出现 `本轮无结果`；有职位但全部被初筛挡掉时，每个岗位各有一条 `初筛淘汰` 和对应原因。
- **明细表里出现 `运行异常`。** 搜索接口这一轮调用失败，`drop_reason` 里是接口返回的错误原文，工作流已自动重试过。下一轮会照常运行，不需要手动清理。
- **Notion 建页报错或列值缺失。** 若找不到数据源或提示权限不足，检查 Data Source 和集成授权；若提示属性校验失败，按第 5 步核对目标列名与类型。已建页但某列为空时，也要检查该岗位传入的字段值。

### 内容来源

岗位数据来自 JSearch 聚合的公开招聘信息，原文与投递入口保留在 Notion 的 `链接` 属性和页面正文里，方便回看原始职位页。聚合内容归各招聘平台与发布方所有，本项目只做检索、筛选与打分整理。