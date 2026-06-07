# 🧯 容错自愈层系统规范 (v1.0 闭环自愈控制台规范)

## 1.0 模块定义

### 1.1 核心定义

容错自愈层是一个**事件驱动的动态执行恢复、断路熔断与自适应路由控制引擎**。

### 1.2 系统架构位置

本层不属于普通的线性阶段，而是一个包裹在执行层和观察层外侧的**闭环反馈控制回路**：

Plaintext

```
       路由层 (Routing Layer)
                 ↓
      上下文处理模块 (Context Processing Module)
                 ↓
 ┌──►  执行层 (Execution Layer) 
 │               │
 │               ▼
 │         观察层 (Observability Layer) ── (抛出异常/超时事件) ──┐
 │               │                                             │
 │               ▼                                             ▼
 └──── 容错自愈层 (Resilience Layer) ◄─────────────────────────┘
                 [当前位置：通过动态控制链条重新驱动执行层]
```

### 1.3 核心目标

容错自愈层负责实时承接观察层抛出的故障信号，执行动态故障隔离、指数退避重试或服务商降级切换：

Plaintext

```
运行时故障/异动信号 (TIMEOUT_EVENT / ANOMALY_EVENT)
                 ↓
执行期局部自愈控制与备用服务商重载调配 (Resilience Action Dispatch)
```

## 2.0 控制回路架构与事件分类

### 2.1 事件驱动流向

Plaintext

```
[执行层异常捕获] → [观察层抽取故障信号] → [容错自愈决策引擎] → [非阻塞退避/熔断介入] → ↺ 重新驱动执行层
```

### 2.2 核心监控事件类型

系统通过捕获以下六类异常事件来激活自愈控制逻辑：

- `TIMEOUT_EVENT`（单路计算服务商网络请求硬超时）
    
- `PROVIDER_FAIL_EVENT`（服务商接口返回明确的错误码或空响应）
    
- `LOW_QUALITY_EVENT`（观察层判定文本异常退化或大面积语义重复）
    
- `FREEZE_EVENT`（生成流式文本中途卡死、无 Token 增长）
    
- `HIGH_TTFT_EVENT`（首字响应延迟严重超时）
    
- `ANOMALY_EVENT`（归一化综合异常度得分突破安全阈值）
    

## 3.0 方法论与技术落地设计

### 3.1 异步自适应指数退避重试 (Retry Policy Engine)

#### 📌 架构适配原则

针对 App Engine Standard 的单线程/多线程 WSGI 进程环境，**在异步执行上下文内，绝对禁止使用 `time.sleep()`**。使用同步阻塞会直接卡死当前的 Web 进程，导致 GAE 实例无法承接其他并发的 HTTP 请求。系统必须无条件采用非阻塞的 `await asyncio.sleep()`。同时，重试逻辑必须加入随机抖动（Jitter），防止并发请求在同一瞬间压垮备用的 `g4f` 计算服务商。

#### 📌 生产落地实现

Python

```
import asyncio
import random

class AsyncRetryPolicy:
    def __init__(self, max_retries=3):
        self.max_retries = max_retries

    def compute_jitter_backoff(self, attempt: int) -> float:
        """
        带随机抖动的指数退避算法：2^attempt + random_jitter
        """
        return (2 ** attempt) + random.uniform(0.0, 0.5)

    async def execute_with_resilience(self, task, call_provider_fn):
        """
        带有异步自愈能力的局部单路模型调度包装器
        """
        wait_time = 1.0
        for attempt in range(self.max_retries):
            try:
                result = await call_provider_fn(task)
                if result.get("status") == "success":
                    return result
            except Exception:
                # 出现异常，准备触发退避自愈
                pass
            
            if attempt < self.max_retries - 1:
                # 计算下一次重试的抖动等待时间
                wait_time = self.compute_jitter_backoff(attempt)
                # 绝对禁止使用 time.sleep()，确保临时事件循环的非阻塞特性
                await asyncio.sleep(wait_time)
                
        return {
            "status": "fallback_trigger",
            "model": task["model"],
            "output": "切换备用服务商"
        }
```

### 3.2 备用服务商降级切换 (Fallback Chains)

#### 📌 核心目标

当某一家 `g4f` 服务商彻底崩溃或重试失败后，系统根据预设的静态确定性依赖拓扑，自动降级至备用服务商池。

#### 📌 依赖拓扑模型

Python

```
FALLBACK_CHAIN = {
    "gpt-4o-mini": ["claude-3.5", "mixtral", "fallback_pool"]
}
```

#### 📌 生产落地实现

Python

```
def select_fallback_provider(model: str, failed_provider: str) -> str | None:
    chain = FALLBACK_CHAIN.get(model, [])
    for next_provider in chain:
        if next_provider != failed_provider:
            return next_provider
    return None
```

### 3.3 带本地线程锁的无状态断路器 (Circuit Breaker)

#### 📌 架构适配原则

由于 Google App Engine Standard 部署时属于多实例自动扩缩容环境，多个独立实例之间无法直接共享内存状态。因此，断路器的设计采用**单实例局部线程锁控制**。针对大规模分布式熔断，可后续扩展轻量级外部持久化介质。但在 Flask 内存环境中，必须使用 `threading.Lock` 保护故障计数器，防止发生并发写死锁或计数错乱。

#### 📌 状态机行为

Plaintext

```
CLOSED (正常访问) ──[失败累加突破阈值]──► OPEN (熔断阻断) ──[熔断超期]──► HALF_OPEN (局部试探)
```

#### 📌 生产落地实现

Python

```
import threading
import time

class ThreadSafeCircuitBreaker:
    def __init__(self, failure_threshold=3, recovery_timeout=10.0):
        self.threshold = failure_threshold
        self.timeout = recovery_timeout
        
        self.state = "CLOSED"
        self.failure_count = 0
        self.last_failure_time = 0.0
        
        # 引入标准的线程锁，确保 Flask 视图多线程上下文内的原子性
        self.lock = threading.Lock()

    def record_failure(self):
        with self.lock:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.threshold:
                self.state = "OPEN"

    def record_success(self):
        with self.lock:
            self.failure_count = 0
            self.state = "CLOSED"

    def can_execute(self) -> bool:
        with self.lock:
            now = time.time()
            if self.state == "OPEN":
                if now - self.last_failure_time > self.timeout:
                    self.state = "HALF_OPEN"
                    return True
                return False
            return True
```

## 4.0 自愈策略决策引擎 (Decision Rules)

自愈引擎通过对观察层的异常指标（如 `failure_score`）进行门限条件判定，触发不同的控制分发动作：

Python

```
def make_resilience_decision(failure_score: float) -> str:
    if failure_score >= 0.8:
        return "SWITCH_PROVIDER"  # 严重故障，直接切走服务商
    if failure_score >= 0.4:
        return "RETRY_SAME_PROVIDER"  # 轻度扰动，允许就地退避重试
    return "IGNORE_EVENT"  # 运行状态平稳，忽略信号
```

## 5.0 动态分发控制器 (Action Dispatcher)

容错层接收决策指令后，在临时的局部事件循环中动态装载任务，改变原有的执行走向：

Python

```
async def dispatch_resilience_action(action: str, task: dict, call_fn) -> dict:
    if action == "RETRY_SAME_PROVIDER":
        # 重新打包，执行非阻塞指数退避重试
        policy = AsyncRetryPolicy()
        return await policy.execute_with_resilience(task, call_fn)
        
    if action == "SWITCH_PROVIDER":
        # 激活备用服务商链条
        next_p = select_fallback_provider(task["model"], task["provider"])
        if next_p:
            task["provider"] = next_p
            return await call_fn(task)
            
    return {"status": "failed", "error": "CRITICAL_CIRCUIT_BROKEN"}
```

## 6.0 模块定位隔离边界规则

### 6.1 容错自愈层严禁执行的操作 (Resilience Layer MUST NOT)

- **严禁在没有观察层故障信号的前提下擅自篡改初始路由决定**。
    
- **严禁越过基础限流信号量强制向上游发射高并发重试请求**。
    

### 6.2 唯一允许的系统操作

- 在局部临时事件循环内进行非阻塞的 `asyncio.sleep` 延迟退避调度。
    
- 当单路计算节点完全失效时，执行透明的同等模型备用服务商无缝降级切换。
    

## 7.0 系统终态总结

### 7.1 形式化定义

容错自愈层是一个基于运行时指标信号流的自校正反馈控制回路。它通过非阻塞指数抖动重试机制和本地进程锁断路器，确保整个多模型聚合系统在部分上游节点崩溃、网络延迟悬挂等复杂生产环境风险下，依然能够维持极高的可用性与整体架构的确定性自愈能力。

### 7.2 核心系统定位

Plaintext

```
容错层不参与模型的初始编译，也不承担推理流的直接生产。

容错层是整个系统底层的“防火墙”与“故障自愈调节阀”。
```