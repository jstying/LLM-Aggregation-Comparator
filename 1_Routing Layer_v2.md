# 🧭 路由层系统规范 (v3.1 统一生产规范)

## 1.0 系统概述

### 1.1 核心定义

路由层是一个**确定性的三状态 LLM 执行编译器与策略引擎**。

### 1.2 核心职责

路由层的核心职责是执行以下转换：



```
用户意图 (User Intent) → 优化的执行计划 (Execution Plan，支持分流并发)
```

在 Flask 同步请求的生命周期内，路由层必须在入口处立刻完成意图解析，严禁包含任何长耗时的 I/O 阻塞操作。

### 1.3 核心处理流水线

路由层通过以下六个步骤完成编译：



```
1. 模式解析 (Mode Resolution)
2. 策略分发 (Policy Dispatch)
3. 候选组合生成 (Candidate Generation)
4. 条件评分引擎 (Conditional Scoring Engine)
5. 分流并发规划 (Fan-out Planning)
6. 执行计划输出 (ExecutionPlan Emission)
```

## 2.0 请求架构规范

### 2.1 路由请求结构体



```Python
class RoutingRequest:
    # 原始提示词
    prompt: str

    # 用户显式指定的模型与服务商列表
    models: list[str] | None
    providers: list[str] | None

    # 显式模式提示：force_auto | force_hybrid | force_manual
    mode_hint: str | None  

    # 约束条件
    max_cost: float | None
    max_latency: float | None

    # 优化目标：speed (速度) | cost (成本) | quality (质量) | balanced (均衡)
    objective: str  
```

## 3.0 三状态模式系统

### 3.1 模式解析器 (Mode Resolver)

模式解析器根据请求参数，确定性地分流至三种核心状态：



```Python
def resolve_mode(req):
    if req.mode_hint:
        return override(req.mode_hint)

    if req.models and req.providers:
        return "FULL_USER_SPECIFIED"

    if req.models and not req.providers:
        return "HYBRID"

    return "FULL_AUTO"
```

### 3.2 模式行为定义

#### 3.2.1 FULL_USER_SPECIFIED (完全用户指定模式)

##### 行为特征

- 系统不参与任何模型和计算服务商的选择。
    
- 关闭所有实时评分机制与路由优化算法。
    

##### 执行契约输出



```Python
ExecutionPlan(
    models=req.models,
    providers=req.providers,
    routing_strategy="direct_map",
    scoring_enabled=False,
    fallback_enabled=False
)
```

#### 3.2.2 HYBRID_MODE (混合路由模式)

##### 行为流水线



```
用户指定模型 → 服务商动态评分 → 过滤筛选 → 排序优化 → 生成并发计划
```

##### 服务商计算函数

在 GAE 静态环境下，评分数据通过 Evaluation 层定期反哺。评分算法如下：



```
最终得分 = 历史稳定性 + 延迟评分 + 成功率 - 错误率惩罚 - 成本惩罚
```

##### 执行契约输出



```Python
ExecutionPlan(
    models=req.models,
    providers=sorted_providers,
    routing_strategy="model_fixed_provider_dynamic",
    scoring_enabled=True,
    fallback_enabled=True
)
```

#### 3.2.3 FULL_AUTO_MODE (全自动路由模式)

##### 行为流水线



```
提示词 → 静态特征识别 → 模型选择器 → 服务商选择器 → 分流并发构建器
```

##### 模型选择器



```Python
def model_selector(prompt):
    if is_code(prompt):
        return ["gpt-4o-mini", "claude-3.5"]

    if is_reasoning(prompt):
        return ["gpt-4.1", "claude-sonnet"]

    if is_fast_query(prompt):
        return ["gpt-4o-mini"]

    return ["balanced_pool"]
```

##### 执行契约输出



```Python
ExecutionPlan(
    models=selected_models,
    providers=selected_providers,
    fanout=fanout_plan,
    routing_strategy="full_optimization",
    scoring_enabled=True,
    fallback_enabled=True
)
```

## 4.0 候选组合生成层

### 4.1 候选组合构建器

##### 核心职责

通过笛卡尔积快速生成可用的执行对：



```
(模型 × 服务商) 执行候选对
```

##### 生产落地实现



```Python
def build_candidates(models, providers):
    return [
        {"model": m, "provider": p}
        for m in models
        for p in providers
    ]
```

## 5.0 评分引擎

### 5.1 激活规则约束

评分引擎根据不同的编译模式进行条件触发：



```
FULL_USER_SPECIFIED → 关闭 (OFF)
HYBRID              → 局部分流评分 (PARTIAL)
FULL_AUTO           → 全自动完整评分 (FULL)
```

### 5.2 核心评分公式

评分引擎使用纯 CPU 计算，公式如下：



```
最终得分 = w1 * 稳定性 + w2 * 延迟响应度 + w3 * 成本效益比 + w4 * 历史成功率
```

## 6.0 分流并发规划 (Fan-out Planning)

### 6.1 并发策略类型

针对 App Engine 无法长期维持大规模异步连接的限制，分流规划分为以下三种：

#### 6.1.1 PARALLEL_ALL (全并发模式)

所有候选组合通过局部的 `asyncio.run()` 生命周期进行单次并发请求。

#### 6.1.2 CASCADE (级联串行模式)

先调用低成本快速模型，根据过滤条件决定是否投递给高强度的推理模型。

#### 6.1.3 TOP_K_SELECTION (最优筛选模式)

仅选择评分最高的前 $K$ 个组合构建并发调用链。

### 6.2 计划输出格式

Python

```
fanout_plan = [
    {"model": m, "provider": p}
    for (m, p) in top_candidates
]
```

## 7.0 ExecutionPlan 执行契约

### 7.1 结构体定义

Python

```
class ExecutionPlan:
    mode: str

    models: list[str]
    providers: list[str]

    # 具体的候选组合与最终分流并发计划
    candidates: list[dict]
    fanout: list[dict]

    routing_strategy: str

    # 策略开关
    scoring_enabled: bool
    fallback_enabled: bool
```

## 8.0 路由层 → 执行层 硬性契约边界

### 8.1 强不可变性规则

执行层（Execution Layer）接收到 `ExecutionPlan` 后，必须将其视为**不可变对象**。

### 8.2 允许的执行操作

执行层只能进行标准的映射投递：

Plaintext

```
execute(plan.fanout)
```

### 8.3 严禁的违规操作

- 严禁在执行层重新选择模型。
    
- 严禁在执行层重新筛选计算服务商。
    
- 严禁动态修改任何路由编译策略。
    

## 9.0 端到端完整执行流图

Plaintext

```
       路由请求 (RoutingRequest)
                  ↓
       1. 模式解析器 (Mode Resolver)
                  ↓
       2. 策略分发器 (Policy Dispatcher)
               ├── FULL_USER_SPECIFIED (完全用户指定)
               ├── HYBRID_MODE        (混合路由)
               └── FULL_AUTO_MODE     (全自动路由)
                  ↓
       3. 候选组合构建器 (Candidate Builder)
                  ↓
       4. 条件评分引擎 (Conditional Scoring Engine)
                  ↓
       5. 分流并发规划器 (Fan-out Planner)
                  ↓
       6. 输出不可变执行计划 (ExecutionPlan Output)
```

## 10.0 系统终态定义

### 10.1 形式化定义

路由层是一个确定性的三状态执行编译器。它负责在严格的策略约束下，将模糊的用户意图编译为优化的多模型分流并发执行计划。

### 10.2 核心定位

Plaintext

```
路由层不对大模型进行实时动态的“感性选择”。

路由层只负责对执行策略进行确定性的“理性编译”。
```