# 🧠 上下文处理层系统规范 (v1.0 辅助模块规范)

## 1.0 模块定义

### 1.1 核心定义

上下文处理模块是一个**执行前置的上下文优化引擎与语义压缩流水线**。

### 1.2 系统架构位置

本层位于路由层与执行层之间，承上启下：

Plaintext

```
路由层 (Routing Layer)
        ↓
上下文处理模块 (Context Processing Module)   ← 当前位置
        ↓
执行层 (Execution Layer)
```

### 1.3 核心目标

上下文处理层执行以下转换，在满足大模型上下文窗口的前提下，精简提示词（Prompt）以降低延迟并提升召回准确率：

Plaintext

```
原始提示词 + 原始上下文 (Raw Prompt + Context)
        ↓
优化、紧凑且对模型友好的上下文 (Optimized, Compact, Model-ready Context)
```

## 2.0 处理流水线 (Processing Pipeline)

系统通过以下五个步骤，按顺序对上下文进行渐进式裁剪：

Plaintext

```
1. 上下文规范化 (Context Normalization)
2. 上下文脱水 (Context Dehydration)
3. 上下文裁剪 (Context Pruning)
4. 提示词压缩 (Prompt Compression)
5. Token 预算优化 (Token Optimization)
```

## 3.0 输入 / 输出架构规范

### 3.1 输入结构体 (ContextRequest)

Python

```
class ContextRequest:
    prompt: str                       # 原始提示词
    chat_history: list[dict] | None   # 历史对话列表
    system_prompt: str | None         # 系统提示词

    max_tokens: int | None            # 目标模型 Token 预算上限
    target_model: str | None          # 目标模型名称
```

### 3.2 输出结构体 (OptimizedContext)

Python

```
class OptimizedContext:
    final_prompt: str                 # 优化后的最终提示词
    compressed_history: str | None    # 压缩脱水后的历史记录

    token_estimate: int               # 预估 Token 消耗数
    compression_ratio: float          # 压缩率统计

    metadata: dict                    # 优化元数据
```

## 4.0 方法论与落地实现设计

### 4.1 上下文规范化 (Context Normalization)

#### 📌 核心目标

统一格式，消除文本噪声与多余空白。

#### 📌 核心步骤

- 剪裁首尾空格，清洗无意义空白符。
    
- 规范化 Unicode 编码，统一多轮对话的角色标记。
    
- 剔除重复或冲突的系统级指令。
    

#### 📌 生产落地实现

Python

```
import re
import unicodedata

def normalize(text: str) -> str:
    # 转换为统一的 NFKC 格式，防止非标准字符引发 Token 计算偏差
    text = unicodedata.normalize("NFKC", text)
    # 将多个连续空格或换行压缩为单空格
    text = re.sub(r"\s+", " ", text)
    return text.strip()
```

### 4.2 上下文脱水 (Context Dehydration)

#### 📌 核心目标

将复杂的历史对话记录（Chat History）提炼为扁平化的**结构化语义块**。

#### 📌 行为流水线

Plaintext

```
原始对话 (Raw Dialogue) → 语义摘要块 (Semantic Summary Blocks)
```

#### 📌 核心步骤

- 提取核心核心意图，剔除无意义的口语化语气词。
    
- 过滤非必要的确认性交互信息，合并重复讨论的主题。
    

#### 📌 生产落地实现

Python

```
def dehydrate(history):
    summary = []
    for msg in history:
        # 过滤口语化语气词、确认词等对话杂质
        if is_filler(msg):
            continue
        summary.append(extract_key_intent(msg))
    return " | ".join(summary)
```

### 4.3 上下文裁剪 (Context Pruning)与线程池隔离

#### 📌 核心目标

删除低价值 Token，仅保留高相关性的上下文片段。

#### 📌 裁剪规则与重要性权重

系统通过以下权重公式进行滑动窗口排序：

Plaintext

```
重要性得分 = 遗忘时间权重 + 语义相关性 + 指令优先级
```

#### 📌 架构修正：线程池隔离设计

由于向量相似度计算（`scikit-learn` / `NumPy`）属于 CPU 密集型操作，在 Flask 同步架构下会直接卡死当前进程。系统必须引入 `ThreadPoolExecutor` 将其隔离，严禁直接在主线程中运行。

#### 📌 生产落地实现

Python

```
from concurrent.futures import ThreadPoolExecutor
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

# 初始化局部线程池，大小根据 GAE 实例核数做轻量化配置
executor = ThreadPoolExecutor(max_workers=4)

def cpu_bound_vector_search(current_embedding, history_embeddings):
    """
    在独立的线程中进行 CPU 密集型的矩阵相似度计算
    """
    scores = cosine_similarity([current_embedding], history_embeddings)[0]
    return scores

def prune(messages, current_embedding, history_embeddings, k=10):
    # 将 CPU 密集型任务投递至线程池，避免阻塞 Flask 的 HTTP 响应主线程
    future = executor.submit(cpu_bound_vector_search, current_embedding, history_embeddings)
    scores = future.result()  # 在当前生命周期内阻塞同步等待结果
    
    # 提取排名前 K 的高价值上下文片段
    top_indices = np.argsort(scores)[-k:]
    return [messages[i] for i in sorted(top_indices)]
```

### 4.4 提示词压缩 (Prompt Compression)

#### 📌 核心目标

在不损失文本核心语义的前提下，最大限度压缩 Token 数量。

#### 📌 压缩技术

- **基于规则的弱压缩**：使用正则表达式过滤、精简修饰词及冗余短语。
    
- **基于语义的强压缩**：将长指令交由高效率轻量模型进行极简等价重写。
    

#### 📌 生产落地实现

Python

```
import re

def compress_prompt(prompt: str) -> str:
    # 移除常见的口语化弱语义修饰词
    prompt = re.sub(r"\b(basically|actually|just|very|please|uhm|ah)\b", "", prompt, flags=re.IGNORECASE)
    return re.sub(r"\s+", " ", prompt).strip()
```

### 4.5 Token 预算优化 (Token Optimization)

#### 📌 核心目标

确保最终的上下文字符完全控制在目标模型的 Token 预算内。

#### 📌 优化策略

Plaintext

```
IF 预估 Token 数 > 目标大模型限制:
    执行逐级退避策略: 1. 启动相似度裁剪 → 2. 弱规则压缩 → 3. 语义脱水摘要
```

#### 📌 生产落地实现

Python

```
import tiktoken

def count_tokens(text: str) -> int:
    # 默认使用通用分词器 cl100k_base 进行快速、轻量化的本地 Token 计数
    enc = tiktoken.get_encoding("cl100k_base")
    return len(enc.encode(text))
```

## 5.0 完整流水线串联执行

Python

```
def process_context(req: ContextRequest) -> OptimizedContext:
    # 1. 规范化处理
    ctx = normalize(req.prompt)

    # 2. 对话历史脱水
    if req.chat_history:
        history_str = dehydrate(req.chat_history)
    else:
        history_str = ""

    # 3. 线程池隔离的向量裁剪 (Pruning) 阶段
    # 注：实际生产环境中，需在此处提取当前 prompt 向量并传入线程池处理
    
    # 4. 提示词压缩
    ctx = compress_prompt(ctx)

    # 5. Token 预算校验
    tokens = count_tokens(ctx + history_str)

    return OptimizedContext(
        final_prompt=ctx,
        compressed_history=history_str,
        token_estimate=tokens,
        compression_ratio=compute_ratio(req.prompt, ctx)
    )
```

## 6.0 外部依赖组件控制

### 6.1 核心必需依赖（轻量级）

Plaintext

```
tiktoken      (用于本地高效 Token 预估计算)
numpy         (用于轻量化矩阵分析)
scikit-learn  (提供余弦相似度计算算法)
```

### 6.2 严禁在主线程加载的重型依赖

Plaintext

```
sentence-transformers / transformers / torch
原因：冷启动耗时长，且在大并发下会引发极其严重的 CPU 竞争。必须置于线程池或外部独立微服务中。
```

## 7.0 模块定位硬性隔离规则

### 7.1 核心边界约束

- 上下文处理模块**严禁参与任何模型选择行为**。
    
- 上下文处理模块**严禁干预或影响路由层的决策流程**。
    
- 上下文处理模块**严禁在当前层直接执行最终的用户推理（Inference）**。
    

### 7.2 唯一允许的输出资产

本模块只能输出：

Plaintext

```
优化后的紧凑上下文对象 (optimized_context)
```

## 8.0 系统终态总结

### 8.1 形式化定义

上下文处理模块是一个确定性的预执行优化流水线。它负责在严格保持语义完整性的前提下，通过规范化、脱水、裁剪和压缩手段，彻底降低 Token 的复杂度。

### 8.2 系统定位

Plaintext

```
本模块不是大模型的“推理或思考系统”。

本模块是一个纯粹的“语义压缩引擎”。
```