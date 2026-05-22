# 企业知识库问答 AI Eval 系统设计与落地

企业知识库问答系统进入生产后，最难的往往不是“能不能回答”，而是“回答是否稳定可信”。尤其在运维、灾备、SOP、产品 FAQ、故障处理这类场景中，用户需要的是有依据、可解释、可回归验证的答案，而不是开放式闲聊。

本文基于一个企业知识库问答评估需求，整理一套轻量 AI Eval 系统设计。目标不是一次性建设复杂平台，而是先建立可持续运行的评估闭环，让检索质量、回答质量和版本变更影响能够被量化。

## 一、背景：从能回答到可评估

一个典型的企业知识库问答系统，常见实现如下：

```text
Question
   ↓
中文分词
   ↓
BM25 Retrieval
   ↓
TopK Docs
   ↓
LLM Generate
   ↓
Answer
```

这类系统适合处理：

- 企业知识库问答
- 产品 FAQ
- 故障处理手册
- 技术文档问答

它的特点是 ToB、专业术语多、强调稳定性和正确性。相比开放聊天，企业问答系统更关心“回答是否可信”“是否引用了正确知识”“关键步骤有没有遗漏”。

## 二、没有 Eval 时会遇到什么问题

### 1. 检索质量不可见

知识库问答系统的第一道门槛是检索。如果正确文档没有被召回，后面的生成模型再强，也只能基于错误上下文组织答案。

没有检索评估时，很难回答：

- 正确文档是否进入了 TopK？
- 正确文档排在第几位？
- 某次检索调整后召回是否退化？
- 专业术语、缩写、同义词是否影响召回？

### 2. 回答质量不可量化

只看最终答案，很容易陷入主观判断。尤其在 SOP 和故障处理场景中，回答可能“看起来很像对的”，但实际上遗漏了权限检查、回滚步骤或风险提示。

需要量化的问题包括：

- 回答是否正确？
- 是否严格基于检索内容？
- 是否回答了用户问题？
- 是否遗漏关键步骤？
- 是否存在模型幻觉？

### 3. 缺少回归测试能力

企业知识库问答系统会持续变化：

- Prompt 调整
- 模型升级
- 检索参数调整
- 分词词典更新
- 文档结构变化
- TopK 策略变化

如果没有 Eval，每次变更只能靠人工抽查，很难判断效果是否整体变好，或者在某些关键问题上悄悄退化。

## 三、第一阶段目标：轻量、可持续、可 Debug

第一阶段不建议直接建设重型标注平台，也不必一开始就引入复杂评测框架。更现实的目标是建立一套轻量、可落地、可持续迭代的 AI Eval 系统。

设计原则：

- 简单：优先用少量表、少量指标跑起来。
- 可持续：评估流程可以异步执行，不阻塞用户请求。
- 可 Debug：保留问题、检索结果、prompt、回答和模型信息。
- 可解释：评分结果要有 reason，而不是只有分数。
- 可回归：能对比模型、prompt 和检索策略变更前后的效果。

## 四、Eval 总体架构

推荐在现有问答链路后增加日志记录和异步评估流程：

```text
用户问题
   ↓
BM25 Retrieval
   ↓
TopK Docs
   ↓
LLM Generate
   ↓
Answer
   ↓
日志记录
--------------------------------
question
retrieved_docs
retrieval_scores
prompt
answer
latency
model
--------------------------------
   ↓
Eval Queue
   ↓
Background Worker
   ↓
LLM Judge
   ↓
JSON 评分结果
```

关键点是：Eval 必须异步执行。用户请求只负责完成问答和日志落库，评估任务进入队列后由后台 worker 处理。

## 五、第一阶段必须记录什么

AI Eval 的基础不是评分，而是日志。没有完整日志，后续无法复盘、无法定位问题，也无法做历史回归。

建议每次问答至少保存以下结构：

```json
{
  "question": "如何恢复某类资源",
  "retrieved_docs": [
    {
      "doc_id": "doc_restore_001",
      "title": "资源恢复操作说明",
      "score": 12.3,
      "content": "脱敏后的文档片段内容"
    }
  ],
  "answer": "脱敏后的 AI 回答内容",
  "prompt": "脱敏后的 Prompt 内容",
  "model": "model-name",
  "latency_ms": 1234,
  "created_at": "2026-01-01T00:00:00Z"
}
```

这里要注意两点：

1. 日志中不要保存敏感凭证、真实客户名称、内部 IP、账号密码、工单号等信息。
2. 如果文档内容包含敏感信息，应在入库前做字段级脱敏，或只保存片段引用和 hash。

## 六、Retrieval Eval：先评估正确文档有没有被找回来

检索评估是第一阶段重点。对于企业知识库问答来说，正确文档能否被召回，直接决定回答上限。

### 1. 建立问题到期望文档的映射

可以先维护一个轻量测试集：

```json
[
  {
    "question": "如何恢复某类虚拟资源",
    "expected_doc": "doc_resource_restore"
  },
  {
    "question": "如何处理某类同步异常",
    "expected_doc": "doc_sync_troubleshooting"
  }
]
```

这里的 `expected_doc` 使用脱敏后的文档 ID，不使用真实文档路径或内部系统名称。

### 2. TopK Hit Rate

TopK Hit Rate 用于判断正确文档是否出现在前 K 个检索结果中。

例如 Top3 Hit：

```text
如果 expected_doc 出现在 retrieved_docs 前 3 个结果中，则命中。
Top3 Hit Rate = 命中的问题数 / 总问题数
```

这个指标简单但很有用。对企业问答来说，Top3 是否命中通常比 Top20 是否命中更重要，因为生成模型最容易使用前几个上下文。

### 3. MRR

MRR 是 Mean Reciprocal Rank，用于衡量正确文档排序位置。

如果正确文档排第 1，得分为 `1/1`；排第 2，得分为 `1/2`；排第 3，得分为 `1/3`；未命中则为 0。

```text
MRR = 所有问题第一个正确文档排名倒数的平均值
```

TopK Hit Rate 关注“有没有召回”，MRR 关注“排得够不够靠前”。

## 七、Answer Eval：用 LLM Judge 评估回答质量

回答评估建议使用 LLM Judge。输入包括：

- 用户问题
- 检索到的知识内容
- AI 最终回答

输出为结构化 JSON，便于后续统计和看板展示。

### 1. 评分维度

第一阶段建议保留四个维度：

| 维度 | 含义 |
| --- | --- |
| Correctness | 回答是否正确 |
| Groundedness | 回答是否基于检索内容，是否减少幻觉 |
| Relevance | 回答是否直接回应问题 |
| Completeness | 回答是否完整覆盖关键步骤 |

每个维度使用 0 到 3 分：

- 0：严重错误或完全无关。
- 1：部分相关但问题明显。
- 2：基本可用，有轻微遗漏。
- 3：准确、相关、完整且有依据。

### 2. Judge Prompt 示例

```text
你是企业 AI 问答质量评估器。

请基于：
1. 用户问题
2. 检索得到的知识内容
3. AI 最终回答

对 AI 回答进行评分。

重点评估：
1. Correctness（正确性）
2. Groundedness（知识依据）
3. Relevance（相关性）
4. Completeness（完整性）

评分范围为 0 到 3。

请只返回 JSON，不要返回 Markdown。
```

### 3. Eval 输出格式

```json
{
  "correctness": 3,
  "groundedness": 3,
  "relevance": 2,
  "completeness": 2,
  "reason": "回答整体正确且有依据，但遗漏了权限检查和恢复后验证步骤。"
}
```

`reason` 很重要。它是后续 Debug 的入口，可以帮助开发者判断问题来自检索、prompt、模型还是知识内容本身。

## 八、推荐的数据结构

第一阶段可以使用 SQLite 这类轻量文件数据库，也可以用已有业务数据库。核心是先把数据结构稳定下来。

### 1. qa_logs

```sql
CREATE TABLE qa_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    question TEXT NOT NULL,
    answer TEXT,
    prompt TEXT,
    model_name VARCHAR(128),
    latency_ms INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2. retrieval_logs

```sql
CREATE TABLE retrieval_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    qa_id INTEGER NOT NULL,
    doc_id VARCHAR(256),
    title TEXT,
    score REAL,
    content TEXT,
    rank INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3. eval_results

```sql
CREATE TABLE eval_results (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    qa_id INTEGER NOT NULL,
    correctness INTEGER,
    groundedness INTEGER,
    relevance INTEGER,
    completeness INTEGER,
    reason TEXT,
    judge_model VARCHAR(128),
    judge_prompt_version VARCHAR(64),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 4. eval_dataset

建议额外增加一个轻量评测集表，用于 Retrieval Eval 和回归测试：

```sql
CREATE TABLE eval_dataset (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    question TEXT NOT NULL,
    expected_doc_id VARCHAR(256),
    category VARCHAR(128),
    enabled INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 九、推荐代码模块

### 1. retrieval_logger.py

负责保存每次问答的检索结果：

- question
- TopK 文档
- doc_id
- title
- score
- rank
- answer
- prompt
- model
- latency

它的价值是让每一次线上问答都可复盘。

### 2. eval_worker.py

负责异步消费 Eval Queue：

- 读取待评估 qa_id。
- 拼装 Judge 输入。
- 调用 Judge LLM。
- 解析 JSON。
- 保存评分结果。

如果 Judge 失败，应记录错误并支持重试，不能影响用户问答主流程。

### 3. judge_prompt.py

负责管理 Judge Prompt 模板和版本号。

Prompt 一旦变更，评分口径可能变化，因此建议保存 `judge_prompt_version`。这样后续统计时可以区分不同评分口径。

### 4. retrieval_eval.py

负责离线计算检索指标：

- TopK Hit Rate
- MRR
- Recall
- 按分类统计命中率

它可以定时跑，也可以在每次检索策略变更后手动触发。

## 十、推荐 API

第一阶段 API 可以保持克制，只提供必要入口。

### 1. 手动触发 Eval

```text
POST /api/eval/run/{qa_id}
```

用于对某条历史问答重新执行 Judge。

### 2. 获取 Eval 结果

```text
GET /api/eval/result/{qa_id}
```

用于查看某次问答的评分和原因。

### 3. 获取统计指标

```text
GET /api/eval/stats
```

示例返回：

```json
{
  "avg_correctness": 2.6,
  "avg_groundedness": 2.8,
  "avg_relevance": 2.7,
  "avg_completeness": 2.4,
  "top3_hit_rate": 0.87,
  "mrr": 0.73
}
```

## 十一、Dashboard 指标优先级

第一阶段 Dashboard 不需要复杂，先把关键指标展示出来。

### 第一优先级

| 指标 | 含义 |
| --- | --- |
| Top3 Hit Rate | 检索是否能找到正确文档 |
| Groundedness Avg | 回答是否基于知识内容，间接反映幻觉风险 |
| Correctness Avg | 回答整体正确性 |

### 第二优先级

| 指标 | 含义 |
| --- | --- |
| MRR | 正确文档排序能力 |
| Completeness Avg | SOP 或处理步骤完整性 |
| Latency P95 | 问答体验和性能波动 |
| Eval Failure Rate | Judge 评估链路稳定性 |

## 十二、回归测试怎么做

当发生以下变更时，应自动或手动触发回归评估：

- Prompt 修改
- 模型版本升级
- BM25 参数调整
- 分词词典更新
- TopK 数量变化
- 文档清洗策略变化

推荐流程：

```text
选择固定 Eval Dataset
   ↓
使用旧配置跑一遍
   ↓
使用新配置跑一遍
   ↓
对比 Retrieval 指标和 Answer 指标
   ↓
输出退化问题清单
```

重点关注：

- Top3 Hit Rate 是否下降。
- MRR 是否下降。
- Groundedness 是否下降。
- Correctness 是否下降。
- 关键问题是否从可用变为不可用。

## 十三、后续演进方向

第一阶段跑通后，可以逐步增强。

### 1. Query Rewrite

对用户问题做改写，提升检索召回。

例如：

```text
对象存储同步
↓
replication mirror bucket sync
```

### 2. 同义词扩展

维护领域词表：

```text
VM = 虚拟机
恢复 = 回滚
同步 = 复制
故障 = 异常
```

对于专业术语较多的场景，同义词和缩写处理往往比直接换模型更有效。

### 3. Hybrid Search

后续可以从纯 BM25 演进为混合检索：

```text
BM25 + Embedding + Rerank
```

BM25 擅长关键词和术语精确匹配，Embedding 擅长语义相似，Rerank 负责把最相关的文档排到前面。

### 4. 用户反馈闭环

引入轻量反馈：

```text
👍 有帮助
👎 没帮助
```

用户反馈不一定直接作为评分标准，但可以用于发现高频问题、补充测试集、修正文档和优化 prompt。

## 十四、总结

企业知识库问答系统的目标，不是像通用聊天机器人一样什么都能聊，而是稳定回答企业真实问题，尤其是运维、SOP 和故障处理类问题。

因此，AI Eval 的价值也不只是“给回答打分”。它真正要解决的是三件事：

1. 检索是否找到了正确依据。
2. 回答是否基于依据且足够完整。
3. 系统变更后效果是否可回归、可解释、可持续优化。

第一阶段只要把日志、Retrieval Eval、LLM Judge、异步 worker 和基础统计跑起来，就已经能显著提升 RAG 系统的工程可控性。后续再逐步加入 Query Rewrite、同义词扩展、Hybrid Search 和用户反馈闭环，整个企业问答系统就会从“能用”走向“可信赖”。
