# 【等价能力】Python 生态能力在 Java 侧的落点

- 对应 ticket：[#3 【等价能力】Python 生态能力在 Java 侧的落点](https://github.com/shangbingo888/xiao_du_information_agent/issues/3)
- 归属地图：[#1 后端 Java 重构](https://github.com/shangbingo888/xiao_du_information_agent/issues/1)
- 结论日期：2026-09-21
- 事实源：本项目 `backend/` 代码、本地 AgentScope Java 仓库、Maven Central 检索接口、上游库官方源码与 README（详见文末「核实来源」）

## 0. 结论速览

| 能力 | 本项目里的真实用途 | 结论 | Java 落点 |
| --- | --- | --- | --- |
| 中文分词 jieba | **无第一方运行时依赖**；只出现在 CodeWizard 的 prompt 文本与静态白名单里 | 不需要 | 若恢复词云图：`com.huaban:jieba-analysis:1.0.2`（jieba 的 Java 移植）或 `com.hankcs:hanlp:portable-1.8.6` |
| token 计数 tiktoken | 上下文裁剪预算：chat 12000 / session 5000 | **可行** | `com.knuddels:jtokkit:1.1.0`，`EncodingType.CL100K_BASE` |
| 文档解析 DocMind | 知识库与文档上传的唯一真实解析路径 | **可行** | `com.aliyun:docmind_api20220711:2.0.6`（阿里云官方 Java SDK） |
| PDF / Word / Excel 本地解析 | **僵尸依赖**：声明了但全仓无 import，PDF/Word 分支只返回占位文本 | 不需要 | 将来若要本地解析：`org.apache.pdfbox:pdfbox:3.0.7`、`org.apache.poi:poi-ooxml:5.5.1`（AgentScope BOM 已锁定） |
| 网页正文抽取 | scout 三级降级：trafilatura → BeautifulSoup → 正则 | **有限可行** | 第 2/3 级用 `org.jsoup:jsoup:1.21.1` 可 1:1 复刻；第 1 级无等价物，候选 `com.chimbori.crux:5.1.0` / `net.dankito.readability4j:1.0.8` |
| 统计与绘图 pandas/numpy/matplotlib/seaborn | 全部在 CodeWizard 沙箱内（exec 生成代码）；numpy 另有两处仅做数组包装 | 不需要（保留 Python 执行面） | 两处 `np.array` → `double[]` / `List<Double>`；沙箱内计算见 [#4](https://github.com/shangbingo888/xiao_du_information_agent/issues/4) |
| nltk | 声明了，全仓零 import | 不需要 | 直接删除依赖 |
| llama-index | 仅作为 DashScope Rerank 与 Node/NodeWithScore 的外壳 | 不需要 | 官方 SDK `com.alibaba:dashscope-sdk-java:2.20.6` 的 `TextReRank`，或直连 REST；覆盖范围另见 [#6](https://github.com/shangbingo888/xiao_du_information_agent/issues/6) |

一句话：**真正需要 Java 侧等价实现的只有三项**——tiktoken、DocMind 接入、网页正文抽取；其余要么可整块丢弃，要么本来就活在 Python 执行面里。

---

## 1. 逐项查证

### 1.1 中文分词 jieba —— 后端根本没有这项能力

全仓（含 `backend/test/`）`jieba` 只出现在 `backend/app/service/deep_research_v2/agents/wizard.py` 的两个位置，且**都不是运行时依赖**：

- 第 233 行：词云图 prompt 模板里要求「使用 wordcloud 和 jieba 库」（这是给 LLM 看的需求描述）
- 第 304–310 行：`ALLOWED_MODULES`，用于对生成代码做**静态**校验的白名单

也就是说，Python 后端自身从不分词；jieba 是「下发给 LLM 生成的代码」的名义能力。

结论：**Java 侧重构不需要任何分词能力**。如果将来要恢复词云图，优先级顺序是「在 Python 沙箱里补齐」（与 Q12 的决策一致），只有在确定要脱离 Python 执行面时才考虑 Java 分词库。

Java 候选（Maven Central 已核实存在）：

| 库 | 坐标 | 说明 |
| --- | --- | --- |
| jieba-analysis | `com.huaban:jieba-analysis:1.0.2` | jieba 的 Java 移植，2.7k star，2023-03 仍有提交，未归档 |
| HanLP | `com.hankcs:hanlp:portable-1.8.6` | 功能远超分词，体积与词典更重 |
| ansj_seg | `org.ansj:ansj_seg:5.1.6` | 老牌中文分词 |
| IKAnalyzer | `com.janeluo:ikanalyzer:2012_u6` | 2012 年后停更，不建议 |

风险：`jieba-analysis` 的词典与 Python jieba 0.42 不同步，分词结果**不保证逐词一致**。由于当前没有任何代码消费分词结果，这个风险现在为零；一旦恢复词云图，需要用同一批文本对拍。

### 1.2 token 计数 tiktoken —— 可行，但要按「预算一致」验收

证据：

- `backend/app/service/chat_service.py:39`：`self.encoding = tiktoken.get_encoding("cl100k_base")`，`self.max_tokens = 12000`
- `backend/app/service/session_service.py:28`：同一个 `cl100k_base`，`self.token_limit = 5000`

用途是**上下文裁剪预算**，不是语义计算，所以只要计数口径一致就等价。

Java 落点：`com.knuddels:jtokkit:1.1.0`。官方 README 明确列出实现的编码为 `r50k_base`、`p50k_base`、`p50k_edit`、`cl100k_base`、`o200k_base`，并给出 `registry.getEncoding(EncodingType.CL100K_BASE)` 的用法；零依赖、支持 Java 8+。

风险评估（诚实标注）：jtokkit 的 README 表述是「为 JVM 提供与 tiktoken 类似的能力」，我**未**逐 token 对拍验证。因为计数结果直接决定上下文裁剪边界（少一个 token 就可能切掉一句话），这项必须进黄金用例：同一批文本，两边输出 `encode(text).size()` 序列并比对。

### 1.3 文档解析 DocMind —— 可行，有官方 Java SDK

证据：

- `backend/app/service/docmind_service.py` 使用 `alibabacloud_docmind_api20220711`，endpoint `docmind-api.cn-hangzhou.aliyuncs.com`，走 `SubmitDocParserJobAdvance` 提交异步任务（`file_url_object` 传文件流）后轮询任务状态取结果
- 调用点：`backend/app/router/document_router.py:72`、`backend/app/router/knowledge_router.py:96`

Java 落点：`com.aliyun:docmind_api20220711:2.0.6`（Maven Central 已核实）。这是同一个 OpenAPI 产品的官方 Java SDK，解析能力等价。

风险不在解析质量，而在**接入层**：异步任务的提交/轮询/超时/重试、文件流上传的写法（Java SDK 是 Darabonba 风格，与 Python 差异较大）。DocMind 是外部计费服务，重试语义写错会产生重复任务或费用，接入层要有幂等与超时上限。

### 1.4 PDF / Word / Excel 本地解析 —— 僵尸依赖，不需要等价

证据：

- `backend/requirements.txt` 声明了 `PyPDF2`、`python-docx`、`openpyxl`
- 全仓（含测试）对 `docx` / `PyPDF2` / `openpyxl` / `pypdf` **零真实 import**
- `backend/app/router/attachment_router.py:78-84`：`.pdf` 与 `.docx/.doc` 分支直接产出占位文本（`f"[PDF 文件: {att.filename}]"`），注释写着「可以后续集成 DocMind 或 PyPDF2」

结论：这是三个装了不用的依赖。Java 侧重构**不需要**为它们做等价实现。

将来若要本地解析：`org.apache.pdfbox:pdfbox:3.0.7` 与 `org.apache.poi:poi-ooxml:5.5.1`。注意这两者已经被 `agentscope-java/agentscope-dependencies-bom/pom.xml` 锁定版本，且 `agentscope-extensions-rag-simple` 已直接依赖它们——属于「框架内已带」，不用额外评估。

风险提示：DocMind 具备扫描件 OCR 与版面还原能力，PDFBox 没有。若将来把 PDF 解析从 DocMind 挪到本地，是**能力降级**而不是等价替换。

### 1.5 网页正文抽取 —— 唯一「有限可行」的一项

证据：`backend/app/service/deep_research_v2/agents/scout.py:1221-1294`，三级降级策略：

1. `trafilatura.extract(html, url=..., include_comments=False, ...)` —— 注释里写明「效果最好」
2. `BeautifulSoup(html, 'lxml')` —— 移除 `script/style/nav/header/footer` 等标签后取文本，再用正则压缩空白
3. 简单正则兜底

Java 落点：

- 第 2、3 级可以**几乎 1:1 复刻**：`org.jsoup:jsoup:1.21.1` 解析并 `remove()` 同名标签，取 `text()`，正则清洗逻辑直接照搬。
- 第 1 级没有对等物。trafilatura 是 Python 生态的启发式正文抽取器，JVM 侧最接近的是 Readability 系实现：`com.chimbori.crux:5.1.0`（活跃）或 `net.dankito.readability4j:1.0.8`。`de.l3s.boilerpipe:boilerpipe:1.1.0` 自 2011 年起停更，不建议引入。

风险（重要）：抽取出的正文会**直接进入 LLM 上下文并影响研究结论**，两个实现的正文差异会传导成报告内容差异。所以这项不能用「逐字节相等」验收，建议在黄金用例里按「正文长度 + 关键词覆盖率 + 关键片段命中」这类软指标定阈值。这也意味着：如果对报告质量有强要求，可以考虑保留一个 Python 抽取 sidecar，与 [#4](https://github.com/shangbingo888/xiao_du_information_agent/issues/4) 的沙箱形态一起决策。

### 1.6 统计与绘图 —— 不需要 Java 替代（前提是 Q12 落地）

证据：

- `backend/app/service/deep_research_v2/agents/wizard.py:1098-1105`：沙箱预热时**真实 import** 了 `matplotlib`（强制 `Agg` 后端）、`pandas`、`numpy`、`seaborn`，注入 `exec_globals`（第 1193-1195 行）后 `exec(code)` 执行 LLM 生成的代码（第 1240 行）
- 同文件第 1116-1120 行的运行时白名单 `allowed_base_modules` 只含 `pandas/numpy/matplotlib/seaborn/datetime/math/statistics/json/collections/re`
- `numpy` 在其他两处的用法都只是包装：`backend/app/service/chat_service.py:167` 与 `backend/app/service/embedding_service.py:134` 把 rerank 分数包成 `np.array`

结论：pandas / numpy / matplotlib / seaborn 是 **CodeWizard 沙箱的资产，不是后端服务的能力**。既然 Q12 已定「保留 Python 执行面」，这些**不需要**在 Java 侧重写；Java 侧唯一要做的是把那两处 `np.array` 换成 `double[]` 或 `List<Double>`（零难度）。

绘图本身归属 [#4 Python 执行面](https://github.com/shangbingo888/xiao_du_information_agent/issues/4)。本 ticket 只记录「若将来真要迁」的候选：`tech.tablesaw:tablesaw-core:0.44.4`、`org.ejml:ejml-core:0.44.0`、`org.jfree:jfreechart:1.5.5`、`com.kennycason:kumo:1.28`（词云）。其中**图表的像素级等价不可行**，这条在 #4 里已经是硬门槛。

### 1.7 nltk —— 僵尸依赖

`backend/requirements.txt` 声明了 `nltk>=3.8.0`，全仓（含测试）**零 import、零引用**。直接删除。

### 1.8 llama-index 与 DashScope rerank —— llama-index 可以整体丢弃

证据（本项目侧）：

- `backend/app/service/embedding_service.py:16-18` 与 `backend/app/service/chat_service.py:10-12` 只从 llama-index 取了 `Node`、`NodeWithScore`、`DashScopeRerank`
- `embedding_service.py:125-131`：把文本包成 `NodeWithScore`，交给 `DashScopeRerank(top_n=..., api_key=...)` 的 `postprocess_nodes`，再把 `res.score` 收成数组

证据（上游侧，读的是 llama-index 官方仓库源码 `llama-index-postprocessor-dashscope-rerank` 的 `base.py`）：`DashScopeRerank._postprocess_nodes` 的全部实质工作就是调用

```python
dashscope.TextReRank.call(model=self.model, top_n=self.top_n,
                          query=query_bundle.query_str, documents=texts,
                          api_key=self._api_key)
```

默认模型是 `gte-rerank`，返回结果按 `result.index` 回填原节点、用 `result.relevance_score` 作为分数。本项目两处调用都没传 `model`，所以用的就是默认的 `gte-rerank`。

结论：**llama-index 在这套代码里只是 Node 容器 + rerank 转发壳，没有承担任何检索或索引职责，可以整体丢弃。**

Java 落点：`com.alibaba:dashscope-sdk-java:2.20.6`（Maven Central 已核实，官方 SDK）的 `TextReRank`，或直接调 DashScope 的 text-rerank REST 接口。另外 `agentscope-extensions-model-dashscope` 是否已覆盖 rerank，需要在 [#6 检索栈](https://github.com/shangbingo888/xiao_du_information_agent/issues/6) 里作为同一条结论确认，避免两处各写一套。

---

## 2. 顺带发现的两个现存缺陷

这两条与迁移无关，但会影响迁移后的验收基线，建议在动手前处理或明确记录。

### 2.1 词云图的 prompt 与沙箱白名单不一致（现存 bug）

`wizard.py` 第 233 行的 prompt 明确要求 LLM「使用 wordcloud 和 jieba 库」；第 301-310 行的静态白名单 `ALLOWED_MODULES` 也放行了 `wordcloud` 与 `jieba`，所以生成的代码能通过校验。

但第 1116-1120 行真正生效的**运行时**白名单 `allowed_base_modules` 里**没有**这两个模块，而第 1127-1133 行的 `safe_import` 对不在白名单的模块直接抛 `ImportError`。

后果：只要 LLM 按要求生成词云图代码，就会在 `exec()` 阶段因 `ImportError` 失败。这意味着**词云图功能当前是坏的**（除非该分支从未被触发）。迁移时如果不清楚这一点，会误以为是新栈的实现缺陷。建议把它作为一条独立的 bug 记录，在 [#4](https://github.com/shangbingo888/xiao_du_information_agent/issues/4) 里一并决策（修白名单，还是改 prompt）。

### 2.2 四个僵尸依赖

`nltk`、`PyPDF2`、`python-docx`、`openpyxl` 声明在 `requirements.txt` 里但全仓零 import；`markdown`、`pyyaml`、`python-dateutil` 同样零 import。Java 侧不需要为它们准备对应物，也说明「Python 依赖清单」不能直接当作迁移工作量清单来读——**必须按真实 import 重新清点**。

---

## 3. 建议

1. 把本 ticket 的结论当作 [#4 Python 执行面](https://github.com/shangbingo888/xiao_du_information_agent/issues/4) 的输入之一：沙箱内的 pandas/numpy/matplotlib/seaborn 保留在 Python，Java 侧不重写绘图与统计。
2. Java 侧只需实现三项：jtokkit 计数、DocMind Java SDK 接入、jsoup/Readability 正文抽取。
3. llama-index 与 nltk 在选型清单里直接删除，不要迁移。
4. 词云图的白名单不一致，建议在迁移前先修掉，让「迁移前基线」本身是可用的。

## 4. 待验证清单（黄金用例的一部分）

| 项 | 验收方式 |
| --- | --- |
| jtokkit vs tiktoken | 同一批中英混排文本，比对 `encode().size()` 序列，要求完全一致 |
| 正文抽取 | 同一批 URL 的 HTML，比对正文长度/关键词覆盖率/关键片段命中，设阈值而非逐字节相等 |
| DocMind Java SDK | 同一批文件，比对解析出的文本与页数；并验证异步任务的幂等与超时 |
| rerank | 同一 query + 文档集，比对 `relevance_score` 与排序（同模型同参数，应高度一致） |

## 5. 核实来源

- 本项目代码：`backend/requirements.txt`、`backend/app/service/{chat_service,session_service,embedding_service,docmind_service}.py`、`backend/app/service/deep_research_v2/agents/{wizard,scout}.py`、`backend/app/router/{attachment,document,knowledge}_router.py`
- AgentScope Java 本地仓库：`agentscope-dependencies-bom/pom.xml`（milvus 2.6.17 / pdfbox 3.0.7 / poi 5.5.1）、`agentscope-extensions-rag-simple/pom.xml`
- Maven Central 检索接口 `search.maven.org/solrsearch/select`：逐条核实上文所有坐标与版本号存在
- jtokkit 官方 README（`raw.githubusercontent.com/knuddelsgmbh/jtokkit`）：编码支持清单
- llama-index 官方源码 `llama-index-postprocessor-dashscope-rerank/base.py`（经由 GitHub API 读取）：rerank 的真实调用契约
- GitHub API `repos/huaban/jieba-analysis`：维护状态与最近提交时间
