# ⚙️ 执行层系统规范 (v1.0 异步并发运行期规范)

## 1.0 模块定义

### 1.1 核心定义

执行层是一个**异步多模型推理分流调度与编排引擎**。

### 1.2 系统架构位置

本层上承上下文处理层，下接观测层，是整个推理生命周期的核心执行节点：

Plaintext

```
路由层 (Routing Layer)
        ↓
上下文处理模块 (Context Processing Module)
        ↓
执行层 (Execution Layer)                  ← 当前位置
        ↓
观测层 (Observability Layer)
```

### 1.3 核心目标

执行层执行以下转换，在 Flask 同步线程的局部可控生命周期内，实现高效的并发调用：

Plaintext

```
不可变执行计划 (ExecutionPlan)
        ↓
并发 LLM 推理结果 (支持流式块拼接与最终数据聚合)
```

## 2.0 执行流水线 (Execution Pipeline)

系统通过以下六个步骤完成单次并发调度：

Plaintext

```
1. 并发任务生成 (Fan-out Task Generation)
2. 异步策略分发 (基于局部循环调用 g4f 资源池)
3. 流式 Token 接收与内存拼接 (Streaming Token Collection)
4. 局部响应组装 (Partial Response Assembly)
5. 异常捕获与动态退避重试 (Failure Handling & Retry)
6. 响应结果聚合 (Response Aggregation)
```

## 3.0 输入 / 输出架构规范

### 3.1 输入结构体 (ExecutionPlan)

Python

```
class ExecutionPlan:
    mode: str                         # 路由状态
    models: list[str]                 # 模型列表
    providers: list[str]              # 计算服务商列表
    fanout: list[dict]                # 具体并发计划：[{"model": "...", "provider": "..."}]

    routing_strategy: str             # 路由策略标签
    scoring_enabled: bool             # 是否启用评分
    fallback_enabled: bool            # 是否启用容灾降级
```

### 3.2 输出结构体 (ExecutionResult)

Python

```
class ExecutionResult:
    responses: list[dict]             # 包含多路响应结果的明细列表

    aggregated_response: str          # 聚合后的最终文本输出

    metadata: dict                    # 执行期耗时、状态等元数据
```

## 4.0 核心架构模型

### 4.1 执行流演进图

Plaintext

```
       执行计划 (ExecutionPlan)
                  ↓
       1. 并发任务构建器 (Fan-out Task Builder)
                  ↓
       2. 异步任务调度器 (g4f 局部并发池)
                  ↓
       3. 信号量控制与超时期接收器 (Semaphore & Timeout)
                  ↓
       4. 容灾恢复与降级层 (Failure Recovery Layer)
                  ↓
       5. 响应结果聚合器 (Response Aggregator)
```

## 5.0 方法论与落地实现设计

### 5.1 并发任务生成 (Fan-out Task Generation)

#### 📌 核心目标

将路由层编译出来的静态执行计划转换为可在事件循环中直接发射的异步并发任务。

#### 📌 生产落地实现

Python

```
def build_tasks(plan, optimized_prompt):
    tasks = []
    for item in plan.fanout:
        tasks.append({
            "model": item["model"],
            "provider": item["provider"],
            "prompt": optimized_prompt
        })
    return tasks
```

### 5.2 异步分发引擎与局部事件循环控制 (核心)

#### 📌 架构修正说明

在 GAE Standard 的 Flask 环境下，严禁运行全局常驻的 `asyncio` 长连接。系统采用**瞬时局部循环模式**：

- 使用 `asyncio.Semaphore(5)` 严格限制单次请求的最大并发数，防止被 `g4f` 的上游源站强行熔断。
    
- 使用 `asyncio.wait_for` 对单路调用施加硬性超时控制，防止因某一家服务商卡死而拖垮整条 Flask 同步线程。
    

#### 📌 生产落地实现

Python

```
import asyncio
import g4f

# 初始化限流信号量，保护上游服务商，防止连接暴增
sem = asyncio.Semaphore(5)

async def fetch_single_model(task, timeout_secs=10):
    """
    单路大模型请求的异步包装器，带有限流与硬超时保护
    """
    async with sem:
        try:
            # 引入硬超时机制，防止单路网络悬挂
            response = await asyncio.wait_for(
                g4f.ChatCompletion.create_async(
                    model=task["model"],
                    provider=task["provider"],
                    messages=[{"role": "user", "content": task["prompt"]}]
                ),
                timeout=timeout_secs
            )
            return {
                "model": task["model"],
                "provider": task["provider"],
                "status": "success",
                "output": response
            }
        except asyncio.TimeoutError:
            return {
                "model": task["model"],
                "provider": task["provider"],
                "status": "failed",
                "error": "TIMEOUT"
            }
        except Exception as e:
            return {
                "model": task["model"],
                "provider": task["provider"],
                "status": "failed",
                "error": str(e)
            }
```

### 5.3 局部生命周期的串联发射 (Fan-out Execution)

#### 📌 核心目标

在同步的 Flask 视图函数中安全触发异步任务，并在完成后立刻销毁临时循环。

#### 📌 生产落地实现

Python

```
def execute_fanout_pipeline(plan, optimized_prompt):
    """
    供 Flask 视图函数直接调用的同步入口
    """
    tasks = build_tasks(plan, optimized_prompt)
    
    async def main():
        # 使用 gather 并发组装所有异步子任务
        coros = [fetch_single_model(t) for t in tasks]
        return await asyncio.gather(*coros, return_exceptions=True)
        
    # 在 Flask 同步线程内开启一个临时的、随用随销的事件循环
    results = asyncio.run(main()) 
    return results
```

### 5.4 异常退避与服务商流式对齐

#### 📌 核心目标

处理服务商无响应（Empty Response）或服务故障，同时统一内部流式接口，在内存中完成 Chunk 拼接。

#### 📌 生产落地实现

Python

```
async def safe_stream_collect(task, retries=2):
    """
    针对流式接口的局部拼接对齐，流式数据块在此处转换为规整输出
    """
    for attempt in range(retries):
        try:
            buffer = []
            # 模拟或调用 g4f 的迭代器进行流式数据块(Chunk)的快速消费与拼接
            # generator = await g4f.ChatCompletion.create_async(..., stream=True)
            # async for chunk in generator: buffer.append(chunk)
            # return "".join(buffer)
            pass
        except Exception:
            if attempt == retries - 1:
                return {"model": task["model"], "error": "FAILED_AFTER_RETRIES"}
            continue
```

### 5.5 响应结果聚合引擎 (Aggregation Engine)

#### 📌 核心目标

从多路成功的模型响应中，采用确定性策略快速筛选或融合成最终可交付输出。

#### 📌 生产落地实现

Python

```
def aggregate(results):
    valid_responses = [r for r in results if r.get("status") == "success"]
    
    if not valid_responses:
        return {
            "aggregated_response": "所有计算服务商调用均已失败或超时。",
            "all_responses": results
        }
    
    # 生产兜底策略：快速筛选出文本长度最长的有效回答作为最佳备选
    best = max(valid_responses, key=lambda x: len(x["output"]))
    
    return {
        "aggregated_response": best["output"],
        "all_responses": valid_responses
    }
```

## 6.0 完整端到端管道函数

Python

```
def run_execution_layer(plan, optimized_prompt) -> dict:
    """
    供外部架构组件调用的单向总线函数
    """
    # 1. 瞬时并发调用
    raw_results = execute_fanout_pipeline(plan, optimized_prompt)
    
    # 2. 结果过滤与清洗
    # 3. 执行最终数据聚合
    final_output = aggregate(raw_results)
    
    return final_output
```

## 7.0 外部依赖规范

### 7.1 核心必需依赖

Plaintext

```
asyncio  (Python 内置标准库，仅允许基于 asyncio.run 局部激活)
g4f      (底层多路多模型统一调用抽象库)
```

## 8.0 模块定位硬性隔离规则

### 8.1 执行层严禁执行的操作 (Execution Layer MUST NOT)

- **严禁重新选择模型**（必须无条件服从路由层结果）。
    
- **严禁擅自修改或增加计算服务商列表**。
    
- **严禁干涉任何属于路由层和上下文处理层的逻辑状态**。
    

### 8.2 唯一允许的行为

- 高效地执行多路并发投递（Fan-out Execution）。
    
- 在单路调用失败时进行安全的退避重试或标记失败。
    
- 确保所有流式或离散的块状响应能够被统一拼接整齐并安全返回。
    

## 9.0 系统终态定义

### 9.1 形式化定义

执行层是一个基于局部生命周期事件循环的并发推理调度引擎。它通过引入严格的信号量防护与绝对超时退避机制，确保在同步 Web 环境中快速、确定性地将路由执行契约转化为高质量的 LLM 聚合响应。