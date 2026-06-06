# 📘 LLM 聚合比较器项目架构规范 (v3 GAE Production Spec)

---

## 0. 🛠️ 技术栈 (Tech Stack)

### Backend

- **Python 3.12+**：利用最新语言特性的异步管理包。
    
- **Flask**：作为核心 Web 服务器，处理同步 HTTP 请求与页面路由。
    
- **asyncio**：在 Flask 请求生命周期内提供局部的多模型 Fan-out 异步并发能力。
    

### Frontend

- **Jinja2**：负责后端模板渲染，输出基础的控制面板与对比卡片静态结构。
    
- **HTML / CSS / JavaScript**：使用原生 JavaScript（Vanilla JS）异步拦截表单，发起推理请求，并动态刷新评估结果。
    

### LLM Layer

- **g4f (GPT4Free)**：用于接入并调度免费/多路 LLM 的逆向及原生 API。
    
- **sentence-transformers**：用于上下文的语义裁剪和最终输出的语义相似度计算。
    

### Evaluation

- **NumPy**：提供高性能的向量和矩阵运算，用于多维指标的归一化与加权。
    
- **scikit-learn**：用于计算文本余弦相似度等 NLP 量化指标。
    

### Deployment

- **Google App Engine (GAE)**：采用 Standard 环境进行云端无服务器部署。
    

---

## 1. 🧭 Routing Layer（模型与 Provider 选择层）

### 📌 职责

负责在请求执行前决定：

- 使用哪些模型
    
- 使用哪些 provider（g4f pool）
    
- 是否启用多模型 fan-out
    
- 调度策略（cost / speed / quality）
    

### 📌 功能

- **manual model selection**（手动模型选择）
    
- **auto routing policy**（自动路由策略）
    
- **provider scoring / filtering**（服务商评分与筛选）
    
- **fan-out plan generation**（并发执行计划生成）
    

### 🧠 大白话

a. 决定叫哪些模型回答

b. 决定使用哪个 Provider

c. 决定是否多模型并发

### ⚙️ 技术落地与规范 (Implementation)

- **多状态模式解析**：根据请求中的模型和提供商参数，分流至 `FULL_USER_SPECIFIED`、`HYBRID` 或 `FULL_AUTO`。
    
- **不可变契约输出**：路由层输出一个结构化的 `ExecutionPlan` 字典。执行层**严禁**修改路由层指定的模型和服务商组合。
    


```Python
class ExecutionPlan:
    def __init__(self, mode, fanout_list, scoring_enabled=True, fallback_enabled=True):
        self.mode = mode               # 路由状态
        self.fanout = fanout_list       # 并发计划: [{"model": "gpt-4o", "provider": "G4F_A"}]
        self.scoring_enabled = scoring_enabled
        self.fallback_enabled = fallback_enabled
```

---

## 2. 🧠 Context Processing Layer（上下文处理层）

### 📌 职责

在请求执行前对上下文进行预处理与优化。

### 📌 功能

- **prompt compression**（提示词压缩）
    
- **context pruning**（上下文裁剪）
    
- **token optimization**（Token 优化）
    
- **context dehydration**（上下文脱水精简）
    
- **context normalization**（上下文规范化）
    

### ⚙️ 技术落地与规范 (Implementation)

- **线程池隔离**：由于 `sentence-transformers` 提取向量是 CPU 密集型操作，在 Flask 环境下必须使用 `concurrent.futures.ThreadPoolExecutor` 隔离运行，避免阻塞 Flask 的同步工作线程。
    


```Python
from concurrent.futures import ThreadPoolExecutor
from sklearn.metrics.pairwise import cosine_similarity

executor = ThreadPoolExecutor(max_workers=4)

def cpu_bound_vector_search(current_embedding, history_embeddings):
    # 使用 NumPy 和 scikit-learn 进行矩阵运算
    scores = cosine_similarity([current_embedding], history_embeddings)[0]
    return scores
```

---

## 3. ⚙️ Execution Layer（执行层）

### 📌 职责

真正执行 LLM 推理请求。

### 📌 功能

- **async fan-out execution**（异步并发执行）
    
- **request dispatch**（请求分发）
    
- **request lifecycle management**（请求生命周期管理）
    
- **streaming token relay**（流式 Token 中继）
    
- **response collection**（响应收集）
    

### 🧠 大白话

a. 真正把请求发出去

b. 同时调用多个模型

c. 收集模型返回结果

### ⚙️ 技术落地与规范 (Implementation)

- **局部事件循环**：由于 Flask 是同步的，采用“生命周期受限的局部事件循环”。在 Flask 请求到达时，通过 `asyncio.run()` 启动临时循环，并发等待所有 `g4f` 响应完成，随后生命周期结束。
    


```Python
import asyncio
import g4f

async def fetch_single_model(model, provider, prompt):
    try:
        response = await g4f.ChatCompletion.create_async(
            model=model, provider=provider, messages=[{"role": "user", "content": prompt}]
        )
        return {"model": model, "provider": provider, "status": "success", "output": response}
    except Exception as e:
        return {"model": model, "provider": provider, "status": "failed", "output": str(e)}

def execute_fanout_pipeline(plan, optimized_prompt):
    tasks = [fetch_single_model(item["model"], item["provider"], optimized_prompt) for item in plan.fanout]
    async def main():
        return await asyncio.gather(*tasks)
    return asyncio.run(main())
```

---

## 4. 👀 Observability Layer（监控层）

### 📌 职责

实时监控推理过程并产生运行时信号（Runtime Signals）。

### 📌 功能

- **TTFT monitoring**（首字延迟监控）
    
- **token/sec monitoring**（每秒 Token 数监控）
    
- **repetition detection**（重复生成检测）
    
- **entropy collapse detection**（熵崩溃检测）
    
- **freeze detection**（卡死检测）
    
- **anomaly detection**（异常检测）
    

### 📌 特性

- **Cross-Cutting Layer（横切层）**：贯穿整个推理过程，而不是串行步骤。
    

### 🧠 大白话

a. 实时盯着模型运行状态

b. 检测异常

c. 给恢复系统提供信号

### ⚙️ 技术落地与规范 (Implementation)

- **状态锁机制**：监控层在流式接收时通过状态锁锁定第一次触发时间，计算出准确的 TTFT（首字输出时间）。
    
- **指标归一化**：使用 NumPy 将不同量纲的秒数、Token 速率映射到 0~1 的统一风险空间内，生成 `anomaly_score`。
    


```Python
import numpy as np

def compute_anomaly_score(ttft, tokens_per_sec, min_ttft, max_ttft):
    ttft_risk = (ttft - min_ttft) / (max_ttft - min_ttft + 1e-5)
    return float(np.clip(ttft_risk, 0, 1))
```

---

## 5. 🧯 Resilience Layer（恢复控制层）

### 📌 职责

根据监控信号进行自动恢复与容错。

### 📌 功能

- **retry policy engine**（重试策略引擎）
    
- **fallback provider switching**（备用服务商切换）
    
- **circuit breaker**（熔断器）
    
- **request cancel / restart**（请求取消与重启）
    
- **provider degradation tracking**（服务商降级追踪）
    

### 📌 特性

- **Event-Driven Control Loop（事件驱动控制循环）**：不是普通业务层，而是控制系统。
    

### 🧠 大白话

a. 发现异常自动处理

b. Provider 挂掉自动切换

c. 自动恢复服务

### ⚙️ 技术落地与规范 (Implementation)

- **非阻塞重试**：在异步控制流内**绝对禁止**使用 `time.sleep()`，必须使用 `await asyncio.sleep()`。
    
- **单实例锁断路器**：在 Google App Engine 的多线程环境中，使用 Python 线程锁保证全局失败计数器的安全性。
    


```Python
import asyncio
import random

async def execute_with_resilience(task, max_retries=3):
    wait = 1.0
    for attempt in range(max_retries):
        result = await call_g4f_provider(task)
        if result["status"] == "success":
            return result
        if attempt < max_retries - 1:
            wait = wait * 2 + random.uniform(0, 0.5)
            await asyncio.sleep(wait)
    return {"status": "fallback_trigger", "output": "切换备用服务商"}
```

---

## 6. 📊 Evaluation & Ranking Layer（评测与排序层）

### 📌 职责

对所有模型输出进行质量评估、性能分析、可靠性统计与综合排序，生成最终推荐结果。

### 📌 核心评估指标矩阵 (Metrics System)

1. **Runtime Metrics（运行指标）**：TTFT、Response Latency、Token Count、Token/sec、Estimated Cost。
    
2. **Reliability Metrics（可靠性指标）**：Success Rate、Failure Rate、Retry Count、Fallback Count、Timeout Rate、Provider Stability Score。
    
3. **Text Quality Metrics（文本质量指标）**：Sentence Length Average、Vocabulary Diversity、Structure Quality、Readability Score、Repetition Rate。
    
4. **Semantic Metrics（语义质量指标）**：Embedding Similarity（利用 `sentence-transformers`）、Relevance Score、Coverage Score、Consistency Score。
    
5. **Judge Metrics（LLM-as-Judge 评估）**：Accuracy Score、Helpfulness Score、Completeness Score、Clarity Score、Instruction Following Score。
    
6. **Comparative Metrics（比较型指标）**：Win Rate、Pairwise Preference Matrix、Elo Rating（评分体系）。
    
7. **Composite Metrics（综合评分指标）**：Weighted Scoring Function、Comprehensive ROI Score、Quality-Cost Efficiency Score。
    

### 📌 排序逻辑（Ranking Engine）

基于多维指标生成最终排名：

$$\text{Final Score} = w_1 \times \text{Runtime} + w_2 \times \text{Reliability} + w_3 \times \text{Text Quality} + w_4 \times \text{Semantic} + w_5 \times \text{Judge}$$

权重根据用户选择的路由策略动态调整：

- **Speed Mode（速度优先）**：极高提升 $w_1$ 权重。
    
- **Cost Mode（成本优先）**：最大化成本抑制权重。
    
- **Quality Mode（质量优先）**：调高 $w_4$ 与 $w_5$ 的语义和裁判分权重。
    
- **Balanced Mode（均衡模式）**：各维度平滑分配。
    

### 📌 输出 (Output Contract)

- **Response Ranking**：Ranked Response List（排序列表）、Best Answer Selection（最佳答案）、Alternative Answers（备选答案）。
    
- **Comparative Analytics**：Model Comparison Dashboard（模型对比仪表盘）、Provider Performance Dashboard（服务商仪表盘）、Historical Trend Analysis（历史趋势）。
    
- **Routing Feedback**：Provider Reputation Update（服务商声誉更新）、Provider Stability Update（稳定性更新）、Future Routing Optimization Signals（优化信号）。
    

### 🧠 大白话

a. 从速度、成本、稳定性、质量多个维度评分

b. 不只是看“答案好不好”，还看“值不值”

c. 输出排名，并反哺 Routing Layer

### ⚙️ 技术落地与规范 (Implementation)

- 利用 NumPy 矩阵乘法计算得分，并通过 Flask 字典（全局上下文）缓存声誉数据，反哺给路由层。
    


```Python
import numpy as np

def rank_by_mode(inputs, mode="balanced"):
    # 动态权重矩阵分配
    weights = {
        "speed": [0.6, 0.2, 0.1, 0.1, 0.0],
        "quality": [0.1, 0.1, 0.2, 0.3, 0.3],
        "balanced": [0.2, 0.2, 0.2, 0.2, 0.2]
    }.get(mode, [0.2, 0.2, 0.2, 0.2, 0.2])
    
    # 使用 NumPy 执行向量化加权汇总并用 np.argmax 提取 Selection
    # ...
```

---

## 3. ⚙️ 系统整体执行逻辑 (Global Flow)


```Plaintext
User Prompt
   ↓
Routing Layer (model/provider selection + fan-out planning)
   ↓
Context Processing Layer (compression / pruning / optimization)
   ↓
Execution Layer (async fan-out inference via g4f)
   ↓
Observability Layer (parallel monitoring & anomaly detection)
   ↓
Resilience Control Loop (retry / fallback / provider switch) ↺ (may re-enter Execution)
   ↓
Evaluation & Ranking Layer (NLP + semantic + runtime scoring)
   ↓
Ranking & Aggregation
   ↓
Final Output
```

---

## 5. 🧱 系统架构 (Architecture)



```
Frontend (Jinja UI)
        ↓
Flask API Layer
        ↓
Comparator Engine Core
        │
        ├── Routing Layer
        ├── Context Processing Layer
        ├── Execution Layer
        ├── Observability Layer
        ├── Resilience Layer
        └── Evaluation & Ranking Layer
        ↓
g4f Provider Pool
```

---

## 6. 📁 项目目录结构 (Directory Layout)



```
/app  
├── main.py (Flask 入口，包含请求生命周期与底层事件循环桥接)  
├── comparator.py (核心引擎，封装 6 大分层组件)  
├── requirements.txt (包含 Flask, g4f, numpy, scikit-learn, sentence-transformers 等)  
├── app.yaml (Google App Engine Standard 运行时配置文件)  
├── templates/  
│    └── index.html (Jinja2 基础评测控制面板与对比卡片结构)  
└── static/ (包含原生 JavaScript，异步拦截表单并动态刷新评估结果)
```

---
## 7. 💻 Frontend & Deployment（前端展示与部署）

- 作为系统唯一用户入口与执行边界，承载请求触发、结果消费与生命周期终止闭环
- 基于 Jinja2 构建轻量静态骨架，实现低延迟首屏渲染与零前端状态依赖
- 通过 Vanilla JS + SSE 接收执行层流式输出，实现多模型结果的实时增量可视化
- 作为 Execution Layer 唯一外部消费端，负责结果展示、对比与最终排序呈现
- 严格适配 Google App Engine Standard 无状态模型与 60s 单请求生命周期约束
- 所有计算严格在单次 HTTP 请求内完成收敛、返回并释放，禁止跨请求状态残留
- 明确禁止后台线程、常驻连接池或任何形式的异步驻留进程
- 定义系统运行边界：按请求瞬时生成、执行、展示并自毁的云原生计算视窗
