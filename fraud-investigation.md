# Fraud Investigation Co-pilot
## A Multi-Agent System for Real-Time Fraud Case Triage

---

## 1. Background

Fraud 这个领域过去 5 年发生了根本性变化:

**过去的 fraud**:信用卡盗刷为主。客户事后发现一笔不认识的交易,银行做 chargeback,损失由发卡行和商家分摊。整个流程是 asynchronous 的——几天或几周之内解决。

**现在的 fraud**:**APP scam (Authorized Push Payment fraud)** 占主导。Sweden 的 Swish、欧洲的 instant SEPA 让钱秒级到账。客户不是被盗刷,是被骗子用 social engineering 骗了之后**主动**把钱转出去。典型 typology:

- Fake police scam:骗子打电话假冒 Polismyndigheten,说账户被黑,要求转到"säkerhetskonto"
- 假冒银行客服
- Investment scam:假高收益理财平台
- Romance scam

钱一旦发出去就是秒到对方账户,然后骗子立刻 layering——把钱拆分、转 3-5 跳、最后在第三国 ATM 取现或换成加密货币。

**关键约束:对 investigator 来说真正的工作窗口是有限的**。EU 的 SEPA Instant 强制 10 秒结算,资金到对方账户后骗子通常在几分钟到几小时内 layering 出去。每多等一分钟,可以 freeze 的下游资金就少一点。

时间窗口的结构:
- 客户报案的那一刻
- 到资金被取现 / 换加密币失踪之前
- 中间这段时间是黄金期,可以联系下游银行 freeze 还在系统里的钱

这个黄金期里 investigator 要做的事:

- 顺着资金链 trace 看钱去哪了,识别下游 mule account
- 比对历史 case,判断是不是某个 active fraud campaign 的一部分
- 判断要不要 freeze、要不要报 SAR
- 协调跨行 freeze request
- (case closed 后) 写 SAR 报告给 FIU

实际操作下来,黄金期常常被花在**跨行 trace + 翻历史 case** 上,真正能做判断 + 行动的时间被压缩。SAR 报告本身也很耗时——一份正经的 SAR 要 1-2 小时。

跟 sanctions 不一样,fraud 的痛点**不是 false positive 太多**,而是**每个 true case 的处理太慢**。

---

## 2. 设计的前提

现代瑞典银行 (Swedbank, SEB, Nordea, Handelsbanken) 的 fraud team 用既有 case management workbench——Actimize FCC、SAS Fraud Management、或自研系统。这些系统已经做了 unified case view:

- 客户 BankID identifies → PNR 锁定 → KYC、recent transactions、historical case notes 一键展示
- Customer service intake 时,客服一边听客户口述一边把 transaction ID、recipient account、reported amount、scam type 录入 structured form
- Detection 系统传 alert 过来,payload 里已经有 customer_id、trigger transaction、counterparty account、amount、timestamp——全是 structured data

**Investigator 打开 case 时,信息就在那。不需要 piece together,不需要 LLM 去多个系统 fetch 数据,不需要"案件简报"。**

这一点是设计的前提。如果一个 agent 的职责是"把分散的信息汇总展示",那它就是在重做 case management 系统已经做完的工作——浪费 latency budget、增加 hallucination risk、还会模糊"GenAI 在 fraud workflow 里到底加什么价值"这个判断。

GenAI 真正能加价值的环节是**判断性的、跨数据源推理的、长文本生成的**任务。

---

## 3. 架构边界:四件事必须想清楚

### 边界一:Detection 不动

Real-time fraud detection 是一个独立的系统,基于:

- Graph network analysis (账户关联网络分析)
- Behavioral scoring (基于客户历史模式)
- Velocity rules (比如"5 分钟内转 3 笔大额")

这些用的是 graph ML + classical ML,毫秒级别给每笔交易打分。**不应该用 LLM 替换这一层**——latency 不达标、cost 太高、explainability 反而下降。

GenAI 只在 alert 触发后、investigator 开始查案的那一刻介入。

### 边界二:Agent 要解决推理问题,不是 dashboard 替代品

每个 agent 必须能回答这个问题:**这件事 case workbench 现成的 UI / 现成的 query 能不能做?** 能做的话,就不需要 agent。

具体到 fraud workflow,什么是 agent 真正能加价值的:

- Money trace 顺着资金链跨多家银行追,中间需要根据账户特征决定追不追、追多深
- Pattern matching 拿当前 case 跟历史几万 closed case 做语义 + 结构 similarity,判断是否属于某个 active campaign
- SAR drafting 把 verified facts reformat 成 goAML 模板的长文本叙事

什么**不是** agent 该做的:

- 拉 customer 历史交易、显示 KYC、显示 case notes——这是 case management 系统的事
- 把 customer service intake 的 structured fields 重新汇总展示——同上

### 边界三:速度比完美重要

跟 sanctions 不一样,fraud investigation 是 **time-critical**。一个 agent 等 30 秒才返回结果,可能就错过了 freeze 窗口。

这意味着:

- Agent 不能跑得太重——必要时降级到部分结果也比 timeout 强
- LLM 调用要选**便宜 + 快**的模型 (Claude Haiku、GPT-4o-mini),不是最聪明的模型
- 部分功能要**预计算 + 缓存**,不能等 investigator 开 case 才开始查

### 边界四:Agent 给建议,human 按按钮

Freeze 一个账户、报 SAR 给 FIU 都是有法律后果的行为。Agent 提供 **structured findings + recommendation**,human investigator 按按钮。

SAR drafting 是 GenAI 真正能省时间的环节——本质上是把已 verified 的事实 reformat 成监管模板,不是生成新事实。

---

## 4. 总体架构

```
       既有 fraud detection 触发 alert (real-time)
                        │
                        ▼
        ┌─────────────────────────────────┐
        │  Investigator opens case in     │
        │  workbench. Unified case view   │
        │  shows KYC, txn history, case   │
        │  notes, trigger txn already —   │
        │  no agent involved at this      │
        │  stage.                         │
        └────────────────┬────────────────┘
                         │
                         ▼
            ┌────────────┴────────────┐
            ▼                         ▼
       ┌──────────┐              ┌──────────┐
       │  Money   │              │ Pattern  │
       │  Tracer  │              │ Matcher  │
       │  Agent   │              │  Agent   │
       └─────┬────┘              └─────┬────┘
             │                         │
             └────────────┬────────────┘
                          ▼
                Synthesized findings
                          │
                          ▼
       Investigator reads, decides actions:
       • Freeze downstream accounts
       • Contact other banks  
       • Reach victim for more info
                          │
                          ▼
             If reportable to FIU:
                          │
                          ▼
              ┌─────────────────────┐
              │   SAR Drafter       │
              │   Agent             │
              └──────────┬──────────┘
                         │
                         ▼
              Draft SAR (human reviews,
              edits, signs, files)
```

设计要点:

**Fully parallel** — Money Tracer 和 Pattern Matcher 之间没有数据依赖,可以同时启动,缩短 wall-clock latency。两个 agent 用同一个 alert payload 作为输入——detection 系统在毫秒级就生成的 structured data。

**SAR Drafter 是 conditional 的** — 只有 investigator 判断 case 是 reportable 之后才启动,不是每个 case 都跑。

**Case workbench 显式标在 agent 流之外** — 这一层是产品 UI 工程的事,不是 GenAI 工程的事。把它放进 agent 流里讨论会模糊"什么场景该用 LLM"这个判断。

---

## 5. 每个 Agent 的角色

### Agent 1: Money Tracer (资金追踪)

**它做什么**:从 victim 的账户出发,顺着 outgoing transfer 追踪资金到下游 5 跳之内,识别哪些是 mule account,建议哪些下游账户应该 freeze,按"还来得及 freeze 的金额"排序。

**为什么需要 agent (而不是单次 LLM call 或纯 graph algorithm)**:这是一个 multi-step reasoning 任务——

1. Graph algorithm 能 traverse 资金链,但每一跳之后要 decide 下一步:这个下游账户是 cash-out 终点 (ATM withdrawal / crypto exchange) 还是中间 mule node?还要不要追?
2. 每个中间节点都要调多个 tool 取 evidence (账户年龄、velocity、关联 cluster、cross-bank flag status),综合 evidence 才能 classify 节点性质
3. 最后要把 graph 的 raw output 翻译成 investigator 能读懂的叙事,叙事里带 freeze recommendation 的优先级

LLM 在这里**不是**做 graph traversal——graph algorithm 做。LLM 做的是**协调多个 tool 的调用顺序 + 综合输出 + 决定每一跳要不要深追**。

**调用的 tools**:

- `trace_outgoing(account_id, depth=5)` — graph traversal,顺着资金追 5 跳
- `check_account_age(account_id)` — 收款方账户开户时长(mule 通常很新)
- `check_velocity(account_id)` — 资金流转速度(典型 mule:money in & out within minutes)
- `cluster_accounts(account_ids)` — 找出关联的 mule cluster
- `check_cross_bank_freeze_status(account_id)` — 查这个下游账户在哪家银行,是否已经被 flag
- `check_endpoint_type(account_id)` — 判断是不是 cash-out 终点 (ATM / crypto exchange / 跨境)

**输出示例**:

```
═══════════════════════════════════════════════════════════════
MONEY TRACE — Case FR-2026-05-21-01134
Initial outflow: SEK 250,000 at 08:52
═══════════════════════════════════════════════════════════════

HOP 1 (08:52)
  Recipient: S. Nilsson, Swedbank ****7234
  Account age: 6 days (high mule risk indicator)
  Status: Funds DEPARTED at 09:04 — already moved
  
HOP 2 (09:04, 12 min after victim transfer)
  Split into 3 transfers:
    • SEK 100K → J. Karlsson, SEB ****1892 (account age 9 days)
    • SEK 100K → A. Pettersson, Nordea ****5567 (account age 12 days)  
    • SEK 50K → Cash withdrawal at ATM in Södertälje (LOST)
  Source: cross-bank graph trace via SVERIGE-FRAUD-NET

HOP 3 (09:08–09:12)
  J. Karlsson account: Funds DEPARTED, traced to crypto exchange 
    deposit address (Binance), SEK 100K likely converted (LOST)
  A. Pettersson account: Funds STILL PRESENT as of 09:14 (FREEZABLE)

═══════════════════════════════════════════════════════════════
RECOMMENDATION
  Recoverable now: SEK 100,000 at Nordea ****5567
  Lost: SEK 150,000 (cash + crypto)
  
  Priority freeze request → Nordea fraud desk, account 5567
  Cluster note: S. Nilsson + J. Karlsson + A. Pettersson 
    appear in same mule cluster (3 connected nodes opened within 
    7-day window, similar IP fingerprint per SVERIGE-FRAUD-NET data)
═══════════════════════════════════════════════════════════════
```

**关键设计**:

- 每个 hop 都明确标 status (DEPARTED / STILL PRESENT / LOST),让 investigator 一眼看出哪里还能 freeze
- Recommendation 按金额优先级排序,不是按时间顺序
- Cluster 信息显示出来——一个 case 经常 expose 一整个 mule network,这个 insight 比单个 case 的价值高

---

### Agent 2: Pattern Matcher (模式匹配)

**它做什么**:把当前 case 跟过去 6 个月的 closed fraud cases 做 similarity match,判断是不是某个 active fraud campaign 的一部分。如果是 campaign,recommend 其他可能正在被同一 group 攻击的客户。

**为什么需要 agent**:这是一个 RAG + reasoning 任务——

1. Closed case 库里有几万条 case,每条带 narrative + structured fields。Vector search 能找到 top-k similar 的 case,但**相似不等于同一 campaign**。
2. Agent 要看 top-k 的具体特征做 judgment:同一 campaign 通常有 (a) 相同的 scam script / 相同的 cover story,(b) 时间上集中在窗口内,(c) 下游 mule cluster 有重叠,(d) victim demographic 相似。
3. 如果判断是 active campaign,要 reasoning 出"这个 campaign 当前可能还在攻击谁",触发 proactive outreach (例如警告类似 demographic 的客户)。

**调用的 tools**:

- `vector_search_cases(case_summary, top_k=20)` — RAG over closed case 库
- `extract_campaign_features(case_id)` — 取一个 closed case 的 campaign-level features (script keywords、mule cluster ID、time window)
- `check_active_campaigns()` — 查当前内部已标记为 "active" 的 campaign 列表
- `query_similar_demographic(victim_profile)` — 找过去 30 天接到过类似 inbound call 但没 transfer 的客户 (potential next victim)

**输出示例**:

```
═══════════════════════════════════════════════════════════════
PATTERN MATCH — Case FR-2026-05-21-01134
═══════════════════════════════════════════════════════════════

CAMPAIGN MATCH: "Polis-Spoofing Q2-2026" (confidence: HIGH)
  • 7 similar cases in past 14 days
  • Common features:
    - Caller claims to be Polismyndigheten
    - Victim instructed to move funds to "säkerhetskonto"
    - Mule accounts opened within past 14 days at 
      Swedbank/SEB/Nordea
    - All victims aged 60+, urban (Stockholm/Göteborg/Malmö)
  • Mule cluster overlap: HOP-2 recipient J. Karlsson appeared
    in case FR-2026-05-08-00782 (already closed)
  
ACTIVE CAMPAIGN STATUS
  Internal flag: CAMPAIGN-2026-POLIS-Q2 (raised 2026-05-10)
  Coordinator: Maria L. (fraud intelligence team)
  
PROACTIVE WARNING CANDIDATES
  43 customers in past 30 days reported "suspicious inbound 
  calls claiming to be polisen" without transferring funds.
  Recommend: outbound warning campaign + temporary 
  out-of-pattern transfer hold for customers 65+.
═══════════════════════════════════════════════════════════════
```

**关键设计**:

- Campaign attribution 带 confidence level,low confidence 时明说 "no strong pattern match" 而不是强行匹配
- Mule cluster 跨 case 关联——这是单 case 看不到的 insight
- Proactive recommendations 是 campaign-level 的,而不是 case-level——这是 pattern matcher 独有的 value

---

### Agent 3: SAR Drafter (可疑活动报告草拟)

**它做什么**:case 进入 SAR-reportable 状态后,生成一份符合 Sweden FIU (Finanspolisen) goAML 模板的 SAR 草稿。

**为什么是 GenAI 真正合理的应用**:SAR 写作是一份长文档的结构化生成任务——

- Mandatory structured fields (victim ID、amount、dates、counterparties、jurisdiction) 是从前面 agent + case data 里直接 fill in 的
- Narrative section ("Description of suspicious activity") 是长文本叙事,把 Money Tracer + Pattern Matcher 已 verify 的事实 reformat 成监管语言,500-1000 字
- Regulatory boilerplate 是 templated 的

一份 SAR 人工写 1-2 小时,LLM 草稿 + investigator edit + sign 可以压到 15-20 分钟。这是真实可量化的时间节省。

**关键边界**:

- **草稿不自动提交**。Investigator 必读、必改、必签
- **法律责任在 human**。LLM 是辅助工具,不是 reporting entity
- **不杜撰事实**。LLM 只 reformat 前两个 agent 已经 verify 过的事实,任何新的"事实陈述"必须能 trace 回 audit trail 里的 source

**调用的 tools**:

- `format_sar_template(jurisdiction='SE')` — 加载 Finanspolisen 的 goAML XML 模板
- `extract_required_fields(case_data)` — 自动填入结构化字段
- `generate_narrative(case_data, max_words=800)` — 生成 SAR 里的"事件描述"叙事段
- `verify_completeness(draft)` — 检查所有 mandatory field 都填了
- `cross_check_facts(draft, audit_trail)` — 每个叙事 claim 必须 trace 回 audit trail 里的 source

---

## 6. State machine

LangGraph 编排:

```python
graph = StateGraph(FraudCaseState)

# Money Tracer and Pattern Matcher run in parallel from start
graph.add_node("money_tracer", money_tracer_node)
graph.add_node("pattern_matcher", pattern_matcher_node)
graph.add_node("synthesizer", synthesizer_node)
# SAR Drafter is conditional — only invoked on investigator demand
graph.add_node("sar_drafter", sar_drafter_node)

graph.add_edge(START, "money_tracer")
graph.add_edge(START, "pattern_matcher")
graph.add_edge("money_tracer", "synthesizer")
graph.add_edge("pattern_matcher", "synthesizer")

# Synthesizer presents findings to investigator,
# investigator decides if SAR is needed
graph.add_conditional_edges(
    "synthesizer",
    route_by_investigator_decision,
    {
        "draft_sar": "sar_drafter",
        "no_sar_needed": END,
    }
)

graph.add_edge("sar_drafter", END)
```

**为什么这个流程不是完全自动化**:Investigator 在 synthesizer 之后**必须 in the loop**——不能让 agent 自己决定要不要写 SAR。SAR 触发后续监管流程,误报有成本 (浪费 FIU 资源),漏报有合规风险。**这个判断必须 human 做**。

---

## 7. State schema

```python
class FraudCaseState(TypedDict):
    # Input — 来自 detection 系统的 alert payload + case workbench
    # 这些都已经 structured,不需要 agent 去重新收集
    case_id: str
    alert_id: str
    victim_customer_id: str  # PNR
    triggered_at: str
    trigger_transaction: dict  # amount, recipient account, timestamp
    
    # Money Tracer outputs
    money_trace_graph: Optional[dict]  # nodes + edges
    recoverable_amount: Optional[float]
    lost_amount: Optional[float]
    freezable_accounts: Optional[list[dict]]
    mule_cluster: Optional[list[str]]
    
    # Pattern Matcher outputs
    matched_campaign_id: Optional[str]
    campaign_confidence: Optional[str]  # HIGH / MEDIUM / LOW / NONE
    similar_cases: Optional[list[str]]
    proactive_warning_candidates: Optional[list[str]]
    
    # Investigator decisions (entered via UI)
    investigator_actions_taken: Optional[list[str]]
    needs_sar: Optional[bool]
    
    # SAR Drafter outputs (only if needs_sar=True)
    sar_draft: Optional[str]
    sar_required_fields: Optional[dict]
    sar_fact_grounding: Optional[dict]  # claim → audit source mapping
    
    # Audit
    audit_trail: list[AgentStep]
```

---

## 8. 评估这个系统怎么做

Fraud 跟 sanctions 评估上的最大区别:**fraud 有相对清晰的 ground truth**。Case closed 之后,investigator 标了 "confirmed fraud" / "unconfirmed" / "false alert" / "amount recovered"。这意味着可以做 retrospective evaluation——拿过去 12 个月的 case,在历史 case 上 replay 整个 agent 工作流,看 agent 输出跟实际 investigator 决策对比。

具体 metric:

### Metric 1: Time to recommendation
关键指标。投放生产前要确认 agent pipeline 在 P95 latency 下能在 60 秒内出 findings。超过这个 latency,黄金窗口就被吃掉太多。

### Metric 2: Freeze accuracy
Money Tracer 推荐 freeze 的下游账户里,**真的是 mule** 的比例。这个指标重要因为 false freeze (错误冻结无辜账户) 会引起客诉 + 监管投诉。

### Metric 3: Campaign attribution precision/recall
Pattern Matcher 把 case 归入 active campaign 的准确率。

- Precision:归入 campaign X 的 case 里,真的属于 X 的比例 (低 precision 会污染 campaign intelligence)
- Recall:实际属于 campaign X 的 case 里,被识别出来的比例 (低 recall 意味着错过预防其他客户被骗的机会)

### Metric 4: SAR draft edit distance
SAR Drafter 生成的草稿,investigator 实际修改了多少字 (edit distance / token-level diff)?如果 investigator 几乎重写,说明草稿没省时间反而浪费时间。Target:edit ratio < 30%。

### Metric 5: Hallucination test
对 SAR draft 尤其重要——SAR 里编造事实会引起监管事故。每个 narrative claim 必须能 trace 回 audit trail 里的 source。任何 ungrounded claim → 阻塞 submission。

---

## 9. Nordic / EU regulatory context

### 数据驻留 + 模型 hosting
Fraud 涉及 PII,public LLM API 不能用。需要 **Azure OpenAI EU region** 或 on-prem open-source model。

### 跨行数据共享
Money Tracer 需要追踪跨行资金流。Sweden 有 SVERIGE-FRAUD-NET (银行间反欺诈协作网络),但跨行数据共享受 GDPR + bank secrecy 限制。Agent 调用的 cross-bank API 已经走过 legal review,但**新的数据流要重新评估**。

### Real-time SLA
Fraud detection 是 mission-critical 系统。Investigation copilot 不能拖累现有 SLA。意味着:

- Agent infrastructure 要跟既有 fraud platform 解耦部署
- Agent timeout 要有 graceful degradation——超时就降级,不能阻塞 investigator workflow

### 跟既有 case management system 的集成
Agent 系统是 **embedded copilot**,通过 API 嵌入既有 workbench——investigator 在原系统里看到 agent findings,不需要切换工具。这一点跟前面"不重做信息聚合"的边界一致:case management 系统**已经**做了 unified view,agent 是叠加在这个 view 上的判断层。

### EU AI Act classification
Fraud investigation copilot 在 EU AI Act 下大概率属于 high-risk AI system (因为影响 individual 的金融服务 access)。意味着:

- Documentation 要求 (technical file, risk management system)
- Human oversight 必须 enforceable——已经做到 (所有 action 都是 investigator 按按钮)
- Logging + traceability 要求 (audit trail field 已经覆盖)

---

## 10. 时间线

**Phase 1 — Single agent POC (8-10 weeks):**

- 先做 Money Tracer agent (最高 value 的 agent)
- 用 historical closed case 测试 trace accuracy + freeze recommendation precision
- 关键决定:LLM 选型 (Haiku vs Sonnet vs Mistral),找到 quality / latency / cost 三角的甜蜜点

**Phase 2 — Multi-agent prototype (10-12 weeks):**

- 加 Pattern Matcher
- 接 cross-bank trace API (复杂度主要在这里)
- 在 shadow mode 跑,跟 investigator 决策对比

**Phase 3 — Pilot rollout (12-16 weeks):**

- 选一个 fraud team (比如专门做 APP scam 的小组) 做 pilot
- 衡量 investigation time saved + recovered fund increase + campaign detection lead time
- 收集 investigator feedback 迭代 output format

**Phase 4 — SAR Drafter addition (后续):**

- SAR Drafter 复杂度更高,放在主流程稳定之后
- 需要单独的 legal review (因为涉及监管报告)

---

## 11. Trade-offs

诚实承认这个设计的弱点:

**Trade-off 1: Latency vs Depth**
60 秒内出 findings 意味着每个 agent 用的是相对便宜的 LLM,推理深度有限。复杂 case 可能需要 investigator 自己再深挖。**这个系统不是替代深度调查,是加速初步决策**。

**Trade-off 2: Cross-bank data dependency**
Money Tracer 的核心价值依赖跨行数据。如果某些银行没接入 SVERIGE-FRAUD-NET,trace 就只能到对方银行的接收账户那一跳,后续 trace 信息不完整。这是 architectural ceiling,不是 implementation 问题。

**Trade-off 3: Pattern Matcher 的冷启动 + drift**
Pattern Matcher 需要历史 closed case 做 vector search。如果是新 fraud type (从来没见过的手法),Pattern Matcher 会找到错误的相似 case,误导 investigator。应对方法是 confidence threshold——top match similarity 低于阈值,agent 输出 "no strong pattern match"。还要定期 retrain embedding model 应对 scam typology drift。

**Trade-off 4: SAR 草稿质量取决于上游事实完整性**
LLM 写 SAR 草稿质量取决于 Money Tracer + Pattern Matcher 的事实收集是否完整。如果上游漏了关键事实,SAR 草稿就会漏。这意味着 SAR Drafter 不能比上游 agent 强。需要 investigator 在 SAR 阶段做 quality check,不能盲信草稿。

**Trade-off 5: Junior investigator 训练问题**
Agent 把 trace 和 pattern matching 嚼碎了,junior 可能学不会自己做 investigation 的基本功。需要 training program 保证 junior 还在做完整 investigation 的练习。

---

## 12. 跟 Sanctions 那套设计的关系

这是同一个 multi-agent framework 用在不同 use case 上。共同点是 architectural pattern:

- Multi-agent 编排用 LangGraph
- Agent 调 tool,LLM 只做 synthesis + reasoning 不做 prediction
- 每个 agent 输出带 source citation
- Audit trail 是 first-class state field
- Human 永远在最终决策环节
- Compliance 边界 (PII, EU AI Act, data residency) baked in

不同点是 workflow shape + agent count:

- Sanctions 是多个 agent fully parallel,fraud 是 2 个 agent parallel + 1 个 conditional
- Sanctions 没有 conditional report generation,fraud 有 SAR Drafter
- Sanctions 优化 false positive reduction,fraud 优化 investigation latency + recovery rate

最重要的不同:**Fraud 的 agent 数量比表面看起来少**——因为现代 case management 系统已经解决了 information aggregation 问题,GenAI 只在真正需要推理的环节介入。这个设计原则比"加更多 agent"重要:**每个 agent 都要能回答"为什么这件事 case workbench 现成的 UI 做不了"**。回答不了的,就不该是一个 agent。

---

## 13. Summary

The core value of GenAI in fraud investigation is to free investigators from cross-bank trace and historical pattern correlation while the freeze window is still open, so they can spend that time on recovery action — and to compress SAR drafting from hours to minutes when a case is reportable.

A few hard lines in the design:

- The detection layer stays untouched. LLMs only enter after the alert fires.
- The case management workbench stays untouched. Information aggregation is a product problem, not a GenAI problem.
- Speed beats sophistication. Cheap and fast LLMs with focused scope, not the most capable model on the market.
- Every agent must justify its existence against the question "can the existing case workbench do this?" — if yes, no agent.
- Agents recommend, humans decide. SARs are always human-signed.
- SAR drafting is a legitimate use of GenAI because it reformats already-verified facts into a regulatory template — not generating new ones.
- Cross-bank data flows need their own legal review.
- Evaluation must cover latency P95, freeze accuracy, campaign attribution precision/recall, and SAR edit distance.

This is what a multi-agent system can credibly look like in a time-critical financial workflow, when designed against real banking workflow constraints rather than imagined ones.
