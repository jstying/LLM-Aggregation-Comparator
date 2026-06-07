# 👀 观察层系统规范 (v1.0 跨切面运行时信号智能规范)

## 1.0 模块定义

### 1.1 核心定义

观察层是一个**跨切面的运行时执行信号提取与系统健康度智能引擎**。

### 1.2 系统架构位置

本层不属于传统的上下游串行关系，而是以**跨切面旁路监听**的模式与执行层并行运转：

Plaintext

```
       路由层 (Routing Layer)
                 ↓
      上下文处理模块 (Context Processing Module)
                 ↓
       执行层 (Execution Layer)  ─────────────► 推理流
                 │
                 └── 观察层 (Observability Layer) ← 当前位置 (旁路实时信号提取)
                 ↓
       韧性层 (Resilience Layer)
```

### 1.3 核心目标

观察层负责实时捕获、计算并聚合多路大模型推理流的物理性能和语义异动指标：

Plaintext

```
执行层原生 Token 流 (Native Token Streams)
                 ↓
标准归一化运行时信号与异常度评分 (RuntimeSignal & AnomalyScore)
```

## 2.0 运行时信号模型 (Runtime Signal Model)

### 2.1 信号提取分类

系统主要抽取以下五类运行时基础信号：

1. **延迟响应信号**（如首字时间 TTFT）
    
2. **吞吐量流速信号**（如每秒生成的 Token 数）
    
3. **语义稳定性信号**（如 N-gram 重复率度量）
    
4. **中断悬挂信号**（如卡死、停滞生成检测）
    
5. **异常分值融合信号**（如多维指标归一化评分）
    

### 2.2 信号输出结构体 (RuntimeSignal)

Python

```
class RuntimeSignal:
    model: str                        # 目标模型
    provider: str                     # 计算服务商

    ttft: float                       # 首字响应时间 (秒)
    tokens_per_sec: float             # 每秒生成速率 (吞吐量)

    repetition_score: float           # 文本复读 / 废话循环系数
    freeze_detected: bool             # 是否触发卡死悬挂
    anomaly_score: float              # 综合归一化异常度得分

    timestamp: float                  # 时间戳
```

## 3.0 方法论与技术落地设计

### 3.1 TTFT 监控与状态锁机制 (Time To First Token)

#### 📌 架构适配原则

针对 Jinja2 后端渲染环境，系统采用 **WSGI 异步适配锁** 方案。在执行层局部的 `asyncio` 流式迭代循环中，首字时间必须引入状态锁，确保在第一包数据到达时写入且仅写入一次时间戳，防止后续大批量 Token 覆盖初始计算。

#### 📌 核心公式

$$TTFT = first\_token\_time - request\_start\_time$$

#### 📌 生产落地实现

Python

```
import time

class TTFTTracker:
    def __init__(self):
        self.start_time = None
        self.first_token_time = None
        self.is_locked = False  # 状态锁：防止后续 Token 重复触发写入

    def start(self):
        self.start_time = time.time()

    def mark_first_token(self):
        """
        在流式响应的第一包数据到达时被触发
        """
        if not self.is_locked:
            self.first_token_time = time.time()
            self.is_locked = True  # 立即闭锁，保护 TTFT 指标

    def get_ttft(self) -> float:
        if not self.first_token_time:
            return 0.0
        return self.first_token_time - self.start_time
```

### 3.2 Token 生成流速监控 (Throughput Monitoring)

#### 📌 核心目标

度量大模型在整个请求周期内的整体吐字吐块速度。

#### 📌 核心公式

$$Tokens/sec = \frac{Total\_Tokens}{Current\_Time - Start\_Time}$$

#### 📌 生产落地实现

Python

```
class TokenSpeedTracker:
    def __init__(self):
        self.tokens_count = 0
        self.start_time = None

    def start(self):
        self.start_time = time.time()

    def add_tokens(self, delta_count: int):
        self.tokens_count += delta_count

    def get_speed(self) -> float:
        duration = time.time() - self.start_time
        if duration <= 0:
            return 0.0
        return self.tokens_count / duration
```

### 3.3 语义循环复读检测 (Repetition Detection)

#### 📌 核心目标

在流式吐字的过程中，通过 N-gram 快速截断算法实时捕获大模型的“复读机崩溃”现象。

#### 📌 生产落地实现

Python

```
from collections import Counter

def check_repetition_ratio(text: str) -> float:
    words = text.split()
    if len(words) < 5:
        return 0.0
        
    # 构造 2-gram 词对滑窗
    ngrams = zip(words, words[1:])
    freq = Counter(ngrams)
    
    repeated = sum(v for v in freq.values() if v > 1)
    return repeated / (len(words) + 1)
```

### 3.4 动态停滞卡死检测 (Freeze Detection)

#### 📌 核心目标

检测大模型连接未中断、但上游服务商在超过一定时间窗内不再吐出任何新 Token 的网络挂起情况。

#### 📌 生产落地实现

Python

```
class FreezeDetector:
    def __init__(self, tolerance_secs=2.0):
        self.last_token_count = 0
        self.last_active_time = time.time()
        self.tolerance_secs = tolerance_secs  # 允许卡死挂起的硬限时间窗

    def is_frozen(self, current_tokens: int) -> bool:
        now = time.time()
        if current_tokens == self.last_token_count:
            # Token 数量不再增长时，开始计时校对
            if now - self.last_active_time > self.tolerance_secs:
                return True
        else:
            # Token 有增长，刷新活跃标记位
            self.last_token_count = current_tokens
            self.last_active_time = now
            
        return False
```

### 3.5 多维指标归一化评分引擎 (Anomaly Scoring Engine)

#### 📌 架构适配原则

**严禁将具有不同量纲和物理单位的绝对数值直接相加**（如秒数和词数/秒）。系统必须使用 `NumPy` 或原生矩阵计算进行 **Min-Max 归一化处理**，将所有延迟、速度物理指标线性平滑映射到 $[0, 1]$ 区间内，再执行加权风险融合。

#### 📌 核心公式

$$TTFT_{risk} = \frac{TTFT_{actual} - TTFT_{min}}{TTFT_{max} - TTFT_{min} + \epsilon}$$

#### 📌 生产落地实现

Python

```
import numpy as np

def compute_anomaly_score(ttft, tokens_per_sec, bounds: dict) -> float:
    """
    基于归一化空间映射的运行期综合风险评分算法
    """
    # 1. 对延迟指标(TTFT)进行正向 Min-Max 映射 (延迟越高，风险越大)
    min_t, max_t = bounds["min_ttft"], bounds["max_ttft"]
    ttft_risk = (ttft - min_t) / (max_t - min_t + 1e-5)
    ttft_risk = np.clip(ttft_risk, 0.0, 1.0)
    
    # 2. 对吞吐量速度(Tokens/sec)进行反向映射 (流速越低，风险越大)
    min_s, max_s = bounds["min_speed"], bounds["max_speed"]
    speed_risk = (max_s - tokens_per_sec) / (max_s - min_s + 1e-5)
    speed_risk = np.clip(speed_risk, 0.0, 1.0)
    
    # 3. 按权重矩阵完成风险计算融合 (50% 延迟响应权重 + 50% 吞吐流速权重)
    anomaly_score = 0.5 * ttft_risk + 0.5 * speed_risk
    return float(anomaly_score)
```

## 4.0 异步跨切面串联挂载模式

Python

```
async def observe_execution_stream(task, stream_iterator, bounds):
    """
    由执行层在临时局部的事件循环中直接调用旁路观察函数
    """
    ttft_tracker = TTFTTracker()
    speed_tracker = TokenSpeedTracker()
    freeze_detector = FreezeDetector()
    
    ttft_tracker.start()
    speed_tracker.start()
    
    buffer_text = []
    
    async for chunk in stream_iterator:
        # 1. 流式分块分发接收
        token = chunk.choices[0].delta.content
        buffer_text.append(token)
        
        # 2. 状态锁介入首字统计
        ttft_tracker.mark_first_token()
        
        # 3. 流速与卡死拦截判断
        speed_tracker.add_tokens(1)
        current_text = "".join(buffer_text)
        
        if freeze_detector.is_frozen(len(buffer_text)):
            # 标记故障并可以向韧性层抛出异常中断信号
            break
            
    # 4. 生成归一化最终指标报告
    final_ttft = ttft_tracker.get_ttft()
    final_speed = speed_tracker.get_speed()
    
    score = compute_anomaly_score(final_ttft, final_speed, bounds)
    
    return {
        "model": task["model"],
        "ttft": final_ttft,
        "speed": final_speed,
        "repetition": check_repetition_ratio(current_text),
        "anomaly_score": score
    }
```

## 5.0 模块定位隔离边界规则

### 5.1 观察层严禁执行的操作 (Observability Layer MUST NOT)

- **严禁擅自修改执行层或模型的输出数据流**。
    
- **严禁干涉任何大模型或计算服务商的选择决定**。
    
- **严禁修改路由层已经编译完成的不可变执行契约**。
    

### 5.2 唯一允许的输出资产

本模块在生命周期结束时，只能对外部上游返回：

Plaintext

```
纯粹量化的运行时信号对象与运行状态矩阵 (runtime_signals only)
```

## 6.0 系统终态定义

### 6.1 形式化定义

观察层是一个跨切面的运行时执行信号抽取与数据归一化评估引擎。它在完全不破坏上游推理流管道原子性的前提下，利用高性能矩阵空间映射算法，对整体集群和单路调用的时延、流速、复读率以及异常状态实施确定性的理度量。

### 6.2 核心系统定位

Plaintext

```
观察层既不执行推理，也不做出调度策略。

观察层只负责冷酷、精准地度量现实。
```