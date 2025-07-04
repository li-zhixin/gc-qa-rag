# 从关键词到智能问答：我们如何用 QA 预生成技术打造高精准 RAG 系统

在企业知识服务的世界里，一个精准、高效的搜索系统是提升用户体验和内部效率的基石。然而，传统的关键字搜索常常陷入匹配不准确的困境。用户简洁的提问，与文档中详尽的陈述性描述之间，似乎总隔着一道难以逾越的“语义鸿沟”。

在葡萄城，作为一家企业级开发工具与解决方案提供商，我们同样面临这一挑战。为了彻底解决用户在产品学习、方案搜寻、问题排查中的痛点，我们决定利用大语言模型（LLM）的强大能力，自研一款能像专家一样即时、准确答疑的**产品智能助手**。

本文将完整复盘我们从 0 到 1 的探索之路，分享我们如何通过**QA 预生成**这一创新思路，从根本上提升 RAG 系统的检索精度，并最终构建出一套稳定、高效、可落地的工程化方案。

## 一、 挑战与构想：为何必须自研 RAG？

我们已有的知识服务体系相当完善，包括标准化文档、GCDN 技术社区和支持跨平台检索的搜索中心。但用户的反馈很明确：**关键字查询的精准度不足**。

随着 LLM 与 RAG（Retrieval-Augmented Generation，检索增强生成）技术的成熟，我们看到了曙光。但市面上的开源或商业化方案却难以直接套用，原因有四：

1.  **内容结构异构**：帮助文档的严谨格式与论坛帖子的自由形态，对通用解析方案是巨大挑战。
2.  **技术路径多样**：RAG 的增强策略繁多，我们需要找到最适合自身业务的技术路线。
3.  **知识库高频更新**：社区内容每日都在增长，必须有高效的增量更新机制。
4.  **自主可控**：为了实现灵活的功能迭代与可控的运维成本，自研是必然选择。

### 核心洞察：从“问句 ↔︎ 答案”到“问句 ↔︎ 问句”

传统 RAG 的检索逻辑是，通过用户的**问题（Question）**去匹配知识库中的**文本段落（Answer）**。这存在一个根本性的错位：

-   **用户提问（疑问句）**：“如何设置响应式布局？”
-   **文档答案（陈述句）**：“在属性面板中找到布局设置选项，支持三种模式...”

这种句式和意图上的差异，导致语义匹配的准确率天然受限。

在探索中，我们发现 LLM 强大的信息抽取能力可以帮助我们换一个思路。如果我们能预先让 LLM 阅读所有文档，并为每一个知识点生成一个或多个**“标准问题”**呢？这样，检索过程就从“问句与答案匹配”变成了**“问句与问句匹配”**。

-   **传统方案**：用户问题 ↔ 答案文本（语义错位）
-   **改进方案**：用户问题 ↔ **预设问题**（语义对齐）

这个思路成为了我们整个项目的技术基石。我们不再简单地将文档切分成段落，而是将其加工成结构化的 **QA 对（Question-Answer Pairs）**。一篇文档可以被拆解为多个知识点，对应生成多个 QA 对，这便是我们 **“QA 预生成 RAG”** 方案的核心。

## 二、 技术架构：分层解析我们的 QA-RAG 系统

我们的系统遵循经典的 ETL、检索、生成三阶段架构，但每个环节都融入了针对性的优化。

### 阶段一：知识构建 (ETL) — 精心雕琢的知识“原材料”

这是整个系统的地基。我们的目标是将数万篇文档与帖子，转化为高质量、可检索的 QA 对。

**1. QA 切片 (QA Slicing)**
我们摒弃了传统的按长度或符号切片的方式，直接利用 LLM 将非结构化的文档内容转化为结构化的 QA 对。这使得每个知识单元都聚焦于一个独立、明确的知识点。

**2. 向量化增强 (Text Embedding Enhancement)**
为了让检索更精准，我们不仅对 QA 文本进行向量化，还引入了多种增强机制：

-   **Summary (上下文摘要)**：为每个 QA 对额外生成一段所在文档的摘要。这能帮助生成模型在回答时更好地理解上下文。
-   **Full Answer (扩展答案)**：为帮助文档中的 QA 对生成一个更详尽的答案版本。用户在搜索结果页点击“展开”即可查看，无需跳转，体验更流畅。
-   **Prefix (前缀机制)**：为了解决不同产品文档中可能存在的相似问题（如各产品的“数据连接”功能），我们在向量化时为问题添加了类别和标题前缀。

```
    #### 示例
    [Category/Title] + Question
    [活字格/连接到外部数据库/连接到 Oracle] 连接到 Oracle 数据库前需要先做什么？
```

    这有效地区分了不同场景下的相似问题，避免了“语义空间混叠”。

最终，每个知识点都包含丰富的元数据和多维度向量，为一个高质量的检索奠定了基础。

**Payload 示例：**

```json
{
    "Question": "连接到 Oracle 数据库前需要先做什么？",
    "Answer": "需要先配置 Oracle，配置完成才能连接 Oracle。",
    "FullAnswer": "# 连接 Oracle 数据库前的准备工作...",
    "Summary": "本文档介绍了如何连接到 Oracle 数据库...",
    "Url": "https://...",
    "Title": "连接到 Oracle",
    "Category": "活字格中文文档/第十五章 连接到外部数据库",
    "Date": 1746093654
}
```

### 阶段二：检索 (Retrieval) — 多路召回与智能排序

我们采用**混合检索 (Hybrid Search)** 策略，双管齐下，确保召回率和相关性。

1.  **稀疏检索 (BM25)**：基于关键词匹配，快速高效。
2.  **稠密检索 (Dense Vector)**：基于语义理解，擅长处理模糊和复杂的查询。

我们发现，**同时对 Question 和 Answer 字段进行检索**，效果最好。因为用户的提问有时更接近答案的表述。两路检索分别召回 Top 40 的结果，再通过 **RRF (Reciprocal Rank Fusion) 算法**进行融合排序，最终选出 Top 8 的最优结果送入生成阶段。

### 阶段三：生成 (Generation) — 可靠、可追溯的智能回答

生成阶段的目标是基于检索结果，为用户提供自然、准确的回答。

**1. 精炼的提示词 (Prompt)**
我们将用户问题和检索到的知识片段组合成清晰的 Prompt，引导 LLM 生成回答。

```python
"""
你正在和用户对话，请综合参考上下文以及下面的用户问题和知识库检索结果，回答用户的问题。回答时附上文档链接。
## 用户问题
{keyword}

## 知识库检索结果
{hits_text}
"""
```

**2. 优雅处理多轮对话**
我们没有采用笨重的全量历史记录输入，而是通过一个**问题改写（Question Rewriting）**模型，智能地从对话历史中识别出用户当前**真正想问的核心问题**，再用这个改写后的问题去执行检索。这既保证了上下文的连贯性，又提升了效率。

## 三、 从蓝图到现实：产品设计与工程落地

好的技术需要好的产品设计来承载。我们最初设想的是一个类似 ChatGPT 的对话式助手，但在深入思考后，我们选择了**“传统搜索界面 + 智能问答”**的混合模式。

**为什么放弃纯对话式？**

1.  **信息优先级**：对用户而言，**搜索结果列表**才是最高价值的信息源，AI 的总结性回答是辅助。纯对话界面会本末倒置。
2.  **上下文的困扰**：多轮对话容易让 LLM 在处理跨领域问题时感到困惑，且响应速度会随轮次增加而变慢，给用户带来不确定性。

**最终设计方案：**

-   **界面**：首页聚焦于搜索输入，搜索结果页则将**智能回答区**与**分类结果列表**（帮助文档、求助中心、专题教程）并排展示，主次分明。
-   **体验**：AI 回答采用打字机效果，即时生成；搜索结果直接展示，无分页；提供“有用/没用”、“复制”、“追问”等快捷操作。
-   **平衡**：默认单次搜索保证效率，同时通过“追问”功能满足深度探索的需求。用户每次追问，都会在页面上追加一组新的“问-答-结果”模块，上下文清晰可见。

**工程化保障：**

为了让系统稳定运行，我们构建了一套完整的工程方案：

-   **部署架构**：采用微服务理念，将知识库构建（ETL）与问答服务分离。核心组件包括 Qdrant 向量数据库、Server 应用、Client 前端和统一的 LLM 接入层。
-   **请求限流**：采用**移动窗口算法**，对不同接口、不同时间维度（分/时/天）进行精细化限流，防止系统被突发流量冲垮。
-   **性能目标**：设定了明确的性能指标（如检索响应 < 2s），并建立完善的监控告警体系，确保服务高可用。

## 四、 成果与价值：智能助手带来的改变

系统上线后，成果显著，为用户、技术支持团队和公司都带来了切实的价值。

**1. 大幅提升用户与技术支持效率**

-   用户自助服务能力显著增强，常见问题即问即答，平均问题解决时间从小时级缩短至分钟级。
-   技术支持团队从重复性问答中解放出来，可以将精力投入到更复杂的问题上。

**2. 显著优化运营成本**

-   人力成本和培训成本降低，知识以结构化的 QA 对形式统一管理，更新和维护效率极高。

**3. 分享我们的技术探索**

-   我们成功验证了 QA-RAG 技术路线在企业级知识服务中的巨大潜力。
-   这套从产品设计到工程落地的完整实践，为业界提供了一套可参考、可复用的方案。

## 五、 总结与展望

从一个提升搜索精度的朴素想法出发，我们经历了一场从技术选型、架构创新到产品打磨、工程落地的完整旅程。**QA 预生成**这一核心思路，被证明是解决企业级 RAG 场景下语义错位问题的有效武器。

当然，这只是一个开始。未来，我们将继续拓展知识库的广度（如 API 文档、视频教程），优化算法的深度，并基于用户反馈不断迭代产品体验。

## 我们相信，通过技术与业务的深度融合，智能问答系统将成为连接用户与产品知识的坚实桥梁，为企业构建一个可持续演进的知识中枢。

如需了解更多细节，欢迎访问葡萄城官网或关注我们的技术社区。
