# Sanctions Screening Investigation Co-pilot
## A Multi-Agent System for Reducing Investigator Workload

---

## 1. 想做这个的起点

Sanctions screening 这件事是银行每天都在做的:
- 跨境交易 (cross-border payment) → 跑一遍 OFAC, EU, UN, UK HMT 制裁名单
- 新客户开户 (KYC onboarding) → 跑一遍
- 客户的 beneficial owner (受益所有人) → 跑一遍
- 公司客户的 directors → 跑一遍

只要名字、地址、出生日期任何一项跟制裁名单上的某个 entry 看起来像,就会触发一个 **hit / alert**,需要人工 review。

我一开始关注这个领域,是因为读到一个行业数字:**北欧大银行每天处理的 sanctions hit 在 1 万到 10 万之间,其中 false positive rate (误报率) 大概在 97–99%**。也就是说,几乎所有 hit 都是无关的同名同姓、拼写相近、地址类似。但每一个 hit 都必须被人 review,因为漏掉一个真的命中,后果是天文数字的罚款 + 监管风险。

这个 false positive rate 在过去 2 年只增不减,因为:
- 俄乌战争之后 EU sanctions package 增加了第 15-19 轮,sanctioned entity 数量爆炸
- 新型 typology 比如 dual-use goods (军民两用商品) 的扩大解释
- 影子舰队 (shadow fleet)、front company (空壳公司) 让间接命中越来越多

银行内部的 sanctions analyst (合规分析师) 看一个 alert 平均花 **20-40 分钟**,要做的事是:
- 查内部 KYC 文档,看这个客户到底是谁
- 跟制裁名单 entry 比对生日、地址、职业
- 搜公开 adverse media (负面新闻),看有没有别的不利信息
- 判断:auto-clear / escalate / 需要 senior 复核

**这件事的瓶颈不在 detection (检测),detection 已经是完成的——既有 fuzzy matching engine 跑出来了。瓶颈在 investigation (调查) 的人工时长。**

GenAI 在这里的价值很具体:**用 LLM 处理 unstructured data (非结构化数据,比如 KYC 文档、新闻、过往 case notes),帮 analyst 把每个 hit 的初步调查压缩到 5-10 分钟**。这不是替代 analyst,是给 analyst 一个 co-pilot,让他们专注在真正需要人判断的复杂 case 上。

---

## 2. 架构:三件事必须想清楚

在画 agent 图之前,我想清楚三个边界:

### 边界一:Detection 不动

现有的 sanctions screening engine (基于 fuzzy name match、phonetic match、threshold rule) 不应该换掉。这层是 **deterministic、auditable、可解释**——监管要求每一个 alert 都能说清楚为什么被触发。LLM 在这一层会加 hallucination 风险,得不偿失。

GenAI 只在 **alert 触发之后** 介入。

### 边界二:Agent 不做最终决定

Auto-clear 一个 sanctions hit 在法律上是有责任的。如果 agent 自动 clear 一个真实命中,银行要承担 regulatory penalty + reputational damage。所以 agent 的角色是 **提供 recommendation + evidence + confidence**,真正按按钮的是 human analyst。

这一点也符合 EU AI Act 对 high-risk AI system 的要求:**meaningful human oversight (有意义的人工监督)** 不是 rubber-stamp,而是 human 真的看了证据再决定。

### 边界三:每一步可追溯

监管 audit (审计) 会问:为什么 6 个月前那笔交易的 alert 被 clear 了?这意味着每一个 agent step 必须有:
- Timestamp
- 调用了哪些 tool
- 用了哪些 source document
- 输出的 reasoning

这是 audit trail (审计追踪) 的标准要求,在 architecture 里要 baked in,不能事后补。

---

## 3. 总体架构

```
        既有 screening engine 触发 alert
                        │
                        ▼
            ┌─────────────────────────┐
            │   Alert Triage Layer    │
            │  (decide which agents   │
            │   to invoke based on    │
            │   alert type)           │
            └────────────┬────────────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
       ┌────────┐  ┌──────────┐  ┌──────────┐
       │Entity  │  │ Context  │  │ Adverse  │
       │Resolver│  │ Gatherer │  │  Media   │
       │ Agent  │  │  Agent   │  │  Agent   │
       └───┬────┘  └─────┬────┘  └─────┬────┘
           │             │             │
           └─────────────┼─────────────┘
                         ▼
              ┌─────────────────────┐
              │  Risk Synthesizer   │
              │       Agent         │
              └──────────┬──────────┘
                         │
                         ▼
              Recommendation + Evidence
                         │
                         ▼
              ┌─────────────────────┐
              │   Human Analyst     │
              │   reviews & decides │
              └─────────────────────┘
```

整体是 LangGraph state machine。三个 specialist agent 并行 (parallel) 跑,因为它们用不同的数据源 (内部 KYC、外部新闻、name matching),互相不依赖。结果汇总给一个 synthesizer agent,生成给 analyst 看的最终 brief。

---

## 4. 每个 Agent 的角色

### Agent 1:Entity Resolver(实体解析)

**它做什么:** 判断"我们的客户" vs "制裁名单上那个人" 到底是不是同一个人。

**为什么需要 LLM:** Sanctioned individual 的信息经常是多语言、多拼写、多 transliteration (音译) 的。比如:
- 制裁名单上的 Александр Новак (俄文) 
- 客户档案里的 Alexander Novak (拉丁字母)
- 第三方数据库里的 Alexandr Nowak (波兰拼写)
- 新闻里的 Aleksandr Novák (捷克拼写)

经典 fuzzy match (基于 Levenshtein distance 这种字符串相似度算法) 在多语言场景下不够。LLM 能 reason "这四个名字可能是同一个人,因为它们都是西里尔字母原始拼写的不同罗马化版本"。

**调用的 tools (内部函数):**
- `fetch_customer_profile(customer_id)` — 拿客户档案,内部数据库
- `fetch_sanctions_entry(list_id, entry_id)` — 拿制裁名单上对应 entry 的完整字段
- `compute_name_similarity(name_a, name_b, scripts=['latin','cyrillic'])` — 经典字符串相似度,作为 baseline
- `check_transliteration_variants(name)` — 检查跨字母系统的拼写变体

**输出:**
```json
{
  "match_confidence": 0.85,
  "matching_attributes": ["name", "year_of_birth"],
  "diverging_attributes": ["nationality", "profession"],
  "reasoning": "Name matches with high confidence including 
                Cyrillic-Latin transliteration variants. However, 
                customer's profession (software engineer) and 
                nationality (Swedish) diverge from sanctioned 
                entry's profile (Russian energy minister)."
}
```

---

### Agent 2:Context Gatherer(背景收集)

**它做什么:** 把这个客户在银行内部所有的信息整理成一段简洁的 customer profile,让 analyst 不用再翻 5 个系统。

**为什么需要 LLM:** 客户的信息散落在:
- KYC 开户文档 (PDF,通常 5-20 页)
- 历史 case notes (relationship manager 写的备忘,unstructured 文字)
- 过去 6-12 个月的 transaction history (结构化数据,但量大)
- 内部 CRM 系统的备注

LLM 的工作是把这些**异构数据 (heterogeneous data)** 总结成一段 200 字以内的 customer brief,而且关键事实必须 cite 回 source。

**调用的 tools:**
- `search_kyc_documents(customer_id)` — RAG over 这个客户的 KYC 文档
- `fetch_transaction_summary(customer_id, days=180)` — 过去 6 个月交易摘要 (top counterparties, total volume, jurisdictions involved)
- `search_internal_notes(customer_id)` — 内部备注全文搜索
- `fetch_account_relationships(customer_id)` — 关联账户、共同 beneficial owner

**输出:**
```
CUSTOMER BRIEF — ID: SEK-12345678
Status: Private banking, customer since 2008
Nationality: Swedish (verified, KYC doc ref #A-2008-115)
Profession: Software engineer at [redacted company]
Typical transaction pattern: Domestic salary + monthly savings transfers, 
  no cross-border activity in past 12 months
Cross-border exposure: 1 transfer to Estonia, 2023-04, EUR 2,500, marked 
  as "family support" (source: TXN-2023-04-1148)
Internal notes: No prior alerts. No PEP exposure. Standard private banking 
  relationship.
```

**关键设计:** 每个事实必须 cite source,这是 audit trail 的一部分。LLM 不被允许说"客户看起来正常"这种话,必须说"基于过去 12 个月 transaction history (source: ...), 没有出现 cross-border red flag"。

---

### Agent 3:Adverse Media(负面新闻)

**它做什么:** 搜公开新闻、监管公告、制裁机构发布,看有没有跟这个客户相关的负面信息。

**为什么需要 LLM:** 这里 LLM 有两个真正难替代的能力:
1. **Disambiguation (消歧)**:搜 "Alexander Novak" 在新闻里会出来一大堆同名人。LLM 需要 reason "这条新闻里的 Alexander Novak 是俄罗斯能源部长,跟我们的客户(瑞典软件工程师)不是同一人"
2. **Multi-language**:北欧银行的客户来自全球,负面新闻可能在俄文、中文、阿拉伯文、波斯文报道。LLM 可以处理跨语言搜索 + 摘要

**调用的 tools:**
- `search_news(query, languages, date_range)` — 调用外部 news API (比如 Factiva, LexisNexis, Dow Jones Risk Center)
- `search_regulatory_announcements(name)` — 搜监管机构公告 (EU OJ, OFAC press releases, SEC enforcement actions)
- `translate_text(text, target_lang='en')` — 多语言翻译
- `classify_adverse_finding(article)` — 分类:financial crime / political exposure / business news / 无关

**输出:**
```json
{
  "adverse_findings": [
    {
      "type": "irrelevant_match",
      "summary": "Found 12 news articles mentioning 'Alexander Novak' 
                  in past 2 years. All refer to former Russian Deputy 
                  Prime Minister. None match our customer's profile.",
      "confidence": 0.95
    }
  ],
  "sources_checked": ["Factiva", "LexisNexis", "EU OJ"],
  "languages_searched": ["en", "ru", "sv"]
}
```

---

### Agent 4:Risk Synthesizer(综合判断)

**它做什么:** 把前面三个 agent 的输出综合成一段 analyst 5 分钟能读完的 brief,加上 recommendation。

**核心设计原则:** 这一步**不是黑盒打分**。Output 必须显式列出:
- 支持 false positive 的证据 (with sources)
- 支持 true positive 的证据 (with sources)
- 整体 confidence
- Recommended action

如果证据矛盾,要诚实标 "Inconclusive — escalate to senior"。**Agent 宁可 escalate 也不能假装确定。**

**输出 template:**
```
═══════════════════════════════════════════════════════════════
SANCTIONS HIT REVIEW — Alert ID: ALT-2026-05-21-08891
Customer: SEK-12345678  |  Sanctioned Entry: EU-RU-2022-145
═══════════════════════════════════════════════════════════════

RECOMMENDATION: Likely false positive (confidence: 87%)
SUGGESTED ACTION: Auto-clear with analyst spot check

EVIDENCE SUPPORTING FALSE POSITIVE:
  ▸ Year of birth differs (customer: 1985, sanctioned: 1960)
    Source: KYC doc #A-2008-115
  ▸ Profession differs (customer: software engineer; 
    sanctioned: government energy minister)
    Source: KYC doc + Dow Jones Risk Center
  ▸ No transaction history to sanctioned jurisdictions in 12 months
    Source: TXN summary
  ▸ Adverse media search returned no profile match
    Source: Factiva, LexisNexis (12 articles reviewed, none match)

EVIDENCE AGAINST FALSE POSITIVE:
  ▸ Name match score 0.92 (high) — including transliteration variants
    Source: Entity Resolver Agent
  ▸ Single low-value transfer to Estonia 2023-04 (EUR 2,500, family support)
    Source: TXN-2023-04-1148

AUDIT TRAIL: 4 agents invoked, 11 tools called, 7 sources cited.
Full trace available at: [audit log link]
═══════════════════════════════════════════════════════════════
```

---

## 5. State machine 怎么走

用 LangGraph 把上面四个 agent 编排成一个 state machine。每个节点是 agent,每条边是 transition。

```python
graph = StateGraph(SanctionsReviewState)

# Parallel branch: three specialist agents run independently
graph.add_node("entity_resolver", entity_resolver_node)
graph.add_node("context_gatherer", context_gatherer_node)
graph.add_node("adverse_media", adverse_media_node)
graph.add_node("synthesizer", synthesizer_node)

graph.add_edge(START, "entity_resolver")
graph.add_edge(START, "context_gatherer")
graph.add_edge(START, "adverse_media")

# All three feed into synthesizer
graph.add_edge("entity_resolver", "synthesizer")
graph.add_edge("context_gatherer", "synthesizer")
graph.add_edge("adverse_media", "synthesizer")

# Synthesizer routes based on confidence
graph.add_conditional_edges(
    "synthesizer",
    route_by_confidence,
    {
        "high_confidence_false_positive": "auto_clear_recommendation",
        "high_confidence_true_positive": "escalate_to_senior",
        "inconclusive": "human_review_required",
    }
)
```

**为什么用 LangGraph 不用 CrewAI 或 AutoGen:**
- Sanctions investigation 的 workflow 是**确定的 state machine**,不是自由对话。每个 alert 走的路径必须 reproducible
- LangGraph 显式 state schema 让 audit trail 是 native 的,不是补丁
- Conditional routing 支持基于 confidence 的分支,符合"低 confidence 自动 escalate"的合规要求

---

## 6. State schema

这是所有 agent 共享的 state object。每个 agent 只写自己负责的字段,不覆盖别人的。

```python
class SanctionsReviewState(TypedDict):
    # Input
    alert_id: str
    customer_id: str
    sanctioned_entry_id: str
    triggered_at: str  # ISO timestamp

    # Entity Resolver outputs
    match_confidence: Optional[float]
    matching_attributes: Optional[list[str]]
    diverging_attributes: Optional[list[str]]
    er_reasoning: Optional[str]
    
    # Context Gatherer outputs
    customer_brief: Optional[str]
    customer_brief_sources: Optional[list[str]]
    
    # Adverse Media outputs
    adverse_findings: Optional[list[dict]]
    media_sources_checked: Optional[list[str]]
    
    # Synthesizer outputs
    recommendation: Optional[str]
    overall_confidence: Optional[float]
    evidence_for_fp: Optional[list[dict]]
    evidence_against_fp: Optional[list[dict]]
    suggested_action: Optional[str]
    
    # Audit
    audit_trail: list[AgentStep]  # append-only log
```

---

## 7. 评估这个系统怎么做

评估是**最不能省**的一块,而且 sanctions 的评估有几个独特的难点:

### 难点一:真正的 true positive 极少
False positive rate 97-99% 意味着,一年里真的命中可能只有几百到几千个。这是个极度不平衡的 dataset。

**对策:** 评估指标不能只看 overall accuracy。要看:
- **Recall on true positives** — 真实命中我们 catch 了多少 (这个不能掉)
- **False positive reduction rate** — 我们 auto-clear 了多少,准确率多高
- **Escalation precision** — escalate 给 senior 的 case 里,真的需要 escalate 的比例

### 难点二:Ground truth 怎么来
评估需要有"标准答案"的历史 case。可以用:
- 过去 12-24 个月已 closed 的 case (analyst 已经判过的)
- Senior analyst review 过的 high-stakes case
- 模拟红队 case (red team) — 安全团队故意构造的边界 case

### 难点三:Hallucination 怎么测
LLM 可能编造一个 source 来支持它的 reasoning。这是 sanctions 场景的致命错误。

**对策:** 每个 agent 输出的 evidence 必须 cite source,自动化测试要 verify 每个 cite 的 source 真的存在且真的支持这个 claim。这叫 **citation grounding test**,在 RAG evaluation 里是标准做法。

具体的实现:对每个 evidence statement,从 source document 里抽出原文片段,LLM-as-judge 判断"这个 claim 是不是真的被这段原文支持"。

---

## 8. 在 Nordic / EU regulatory context 下的几个 specific 考虑

### 数据驻留 (Data residency)
Sanctions data 包含客户 PII。OpenAI 公开 API 在 US,不能直接调。需要:
- **Azure OpenAI EU region** (Sweden Central, North Europe) 
- 或者 **on-prem 部署的 open-source 模型** (Mistral, Llama 70B+)
- Embedding 模型也要本地跑,不能用 OpenAI Embeddings

### EU AI Act high-risk classification
Sanctions screening 自动决策受 EU AI Act Annex III 监管。意味着:
- Risk management system 是 mandatory
- Data governance + bias monitoring 是 mandatory
- Human oversight 必须 meaningful (不是 rubber-stamp)
- 系统的 conformity assessment 要 documented

这些不是可选,是法律要求。Architecture 设计的时候每一条都要对应一个具体的 implementation。

### 跟既有系统的集成
银行的 sanctions screening engine 通常是商业产品 (Fircosoft, LexisNexis WorldCompliance, Refinitiv World-Check)。新的 agent 系统是 **augmentation layer**,不替换 screening engine。集成点是:
- 接收 screening engine 输出的 alert
- 写回 recommendation + evidence
- 不修改 screening engine 内部逻辑

这是个 architectural 的关键决定:**少改动现有系统,通过 API 做 augmentation**。

---

## 9. 时间线

如果要实际做这个系统,粗略估计是:

**Phase 1 — POC (4-6 weeks):**
- Single agent (Entity Resolver) + mock data
- 用 LangGraph 跑通基础 workflow
- 在 historical closed case 上跑一遍 benchmark

**Phase 2 — Full multi-agent prototype (8-10 weeks):**
- 4 个 agent 全部上
- 接 real KYC system + news API
- Citation grounding evaluation framework
- Internal red team testing

**Phase 3 — Pilot in production (12-16 weeks):**
- Shadow mode — agent 跑但不影响真实决策,只对比 analyst 的判断
- 收集 1000+ alerts 的 ground truth
- 调 confidence threshold

**Phase 4 — Limited production (rolling out):**
- 先在某一类低风险 alert (比如 EU non-sanctioned country 的 cross-border) 启用 auto-clear
- 监控 false negative rate
- 逐步扩展到更多 alert type

---

## 10. 这个设计的几个 trade-off 是诚实的

不假装完美:

**Trade-off 1:Parallel vs Sequential**
三个 specialist agent 并行跑 latency 短,但成本高 (LLM token 3x)。如果一个 alert 99% 是 false positive,跑完三个 agent 才发现这点其实是浪费。
**可能的优化:** Adaptive routing — 先跑便宜的 Entity Resolver,如果 confidence 已经很高再决定要不要跑后两个。

**Trade-off 2:LLM 还是经典 ML**
有些 sub-task (比如 name match score) 经典 ML 更便宜更可控。LLM 用在它真正有附加值的地方 (multi-language reasoning, unstructured doc summarization),不是 everything。

**Trade-off 3:External news API 的成本**
Factiva, LexisNexis 这种 API 按 query 计费,1 万 alert/天意味着大量 API call。需要 caching layer + 复用 query。

**Trade-off 4:这个系统 onboard 一个新 analyst 反而更难**
如果 junior analyst 只看 agent 生成的 brief,他们可能学不会自己做 sanctions investigation。需要 training program 保证 junior 还在做完整 investigation 的练习。

---

## 11. 总结

GenAI 在 sanctions screening 的核心价值是**把 analyst 时间从 routine investigation 里释放出来,让他们专注真正复杂的 case**。

设计上的几条 hard line:
- Detection layer 不动
- Agent 不做最终决定,human 永远在 loop 里
- 每一步可追溯,audit trail 是 first-class citizen
- 评估必须包含 citation grounding,LLM hallucination 在 sanctions 场景是致命的
- 数据驻留 + EU AI Act 合规是 day-one 的设计要求,不是事后补

这是一个 multi-agent system 在受监管的金融环境里能站住脚的样子。
