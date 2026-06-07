# 💻 前端展示与部署层系统规范 (v1.0 边缘消费与云原生适配规范)

## 1.0 模块定义

### 1.1 核心定义

前端展示与部署层是整个多模型编排系统的**边缘用户交互触点、数据流终端消费视窗与云原生无状态运行时宿主契约**。

### 1.2 系统架构位置

本层作为整个 6 层管道的包裹外壳与物理物理边界，负责将核心方法论落地为可交付的 Web 服务：

Plaintext

```
 ┌────────────────────────────────────────────────────────┐
 │ 部署适配层 (Google App Engine Standard 无状态沙箱容器)    │
 │   ┌────────────────────────────────────────────────────┤
 │   │ 前端展示层 (Jinja2 静态骨架 + Vanilla JS 动态异步消费) │
 │   │   ├────────────────────────────────────────────────┤
 │   │   │  核心 6 层管道 (路由 -> 上下文 -> 执行 -> 观察 -> 容错 -> 评估)
 └───┴───┴────────────────────────────────────────────────┘
```

### 1.3 核心目标

前端展示与部署层负责执行以下双向转换，在严格的云端沙箱限制下实现极致的用户交互：

Plaintext

```
后端 6 层流水线聚合矩阵 (Backend System Matrices)
                 ↓↑ (通过 HTTP POST / SSE 协议进行无状态泵送)
Jinja2 骨架视图 + 前端事件驱动渲染 (Dynamic Model Cards & Concurrency Animations)
```

## 2.0 前端通信与流式消费架构

前端采用“同步结构加载 + 异步局部增量消费”的混合模型，分为以下三个核心步骤：

Plaintext

```
1. Jinja2 后端同步渲染 (加载控制面板基础 HTML 骨架与 CSS 样式表)
2. Vanilla JS 异步事件激活 (用户提交请求，前端发起 Fetch 或 EventSource 连接)
3. 动态 DOM 矩阵注入 (实时渲染多模型并发加载动画，动态刷新排名前三的模型卡片)
```

## 3.0 前端技术落地实现 (Frontend Implementation)

### 3.1 Jinja2 控制面板基础骨架 (骨架页模板)

#### 📌 核心目标

利用 Jinja2 的纯后端渲染特性，瞬时输出不包含动态数据的静态控制面板骨架，确保秒开率（降低首屏白屏时间）。

#### 📌 生产落地实现

HTML

```
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>Multi-LLM 编排与评测控制面板</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/main.css') }}">
</head>
<body>
    <div id="app-container">
        <section id="control-panel">
            <textarea id="prompt-input" placeholder="输入你的提示词..."></textarea>
            <button id="submit-btn" onclick="triggerInferencePipeline()">运行并发评测</button>
        </section>

        <section id="monitor-view">
            <div id="concurrency-loader" class="hidden">
                <span class="spinner"></span> 正在编译路由并分流执行异步 Fan-out...
            </div>
            <div id="ranking-matrix-cards" class="card-grid">
                </div>
        </section>
    </div>
    <script src="{{ url_for('static', filename='js/app.js') }}"></script>
</body>
</html>
```

### 3.2 Vanilla JS 异步 SSE 流式消费引擎 (Edge Consumer)

#### 📌 核心目标

放弃传统的整页刷新，利用标准浏览器内置的 `EventSource` 消费后端推送的 Server-Sent Events (SSE) 信号。在不引入重型前端框架（如 React/Vue）的前提下，实现多路大模型并行吐字的流式动画。

#### 📌 生产落地实现

JavaScript

```
// static/js/app.js

function triggerInferencePipeline() {
    const prompt = document.getElementById("prompt-input").value;
    const loader = document.getElementById("concurrency-loader");
    const cardGrid = document.getElementById("ranking-matrix-cards");

    if (!prompt) return;

    // 1. 激活多模型并发加载动画
    loader.classList.remove("hidden");
    cardGrid.innerHTML = ""; // 清空上一轮评测卡片

    // 2. 建立标准的服务端发送事件(SSE)异步连接
    const encodedPrompt = encodeURIComponent(prompt);
    const eventSource = new EventSource(`/api/stream_inference?prompt=${encodedPrompt}`);

    // 3. 监听多流并发块实时到达事件
    eventSource.onmessage = function(event) {
        const data = JSON.parse(event.data);
        
        if (data.status === "processing") {
            // 动态更新或创建对应的多模型流式渲染卡片
            updateModelLiveCard(data.model, data.provider, data.chunk_text);
        }

        if (data.status === "completed") {
            // 4. 当后端评估层仲裁完毕，接收最终排名前三的最优解矩阵并关闭连接
            renderFinalTopThreeCards(data.ranked_results);
            loader.classList.add("hidden"); // 隐藏加载动画
            eventSource.close();
        }
    };

    eventSource.onerror = function() {
        loader.classList.add("hidden");
        eventSource.close();
    };
}

function updateModelLiveCard(model, provider, chunk) {
    let card = document.getElementById(`card-${model}-${provider}`);
    if (!card) {
        card = document.createElement("div");
        card.id = `card-${model}-${provider}`;
        card.className = "live-card shadow animate-pulse";
        card.innerHTML = `<h3>${model}</h3><small>${provider}</small><p class="content"></p>`;
        document.getElementById("ranking-matrix-cards").appendChild(card);
    }
    card.querySelector(".content").innerText += chunk;
}

function renderFinalTopThreeCards(rankedResults) {
    const cardGrid = document.getElementById("ranking-matrix-cards");
    cardGrid.innerHTML = ""; // 清洗中间流态，挂载最高公允分值最优解

    // 仅提取评估层计算出的前三名
    rankedResults.slice(0, 3).forEach((item, index) => {
        const finalCard = document.createElement("div");
        finalCard.className = `final-card medal-${index + 1}`;
        finalCard.innerHTML = `
            <div class="badge">TOP ${index + 1}</div>
            <h2>${item.model}</h2>
            <p class="score">综合仲裁得分: ${item.final_score.toFixed(2)}</p>
            <div class="text-body">${item.text}</div>
            <div class="meta-tags">
                <span>TTFT: ${item.runtime_signal.ttft.toFixed(2)}s</span>
                <span>速度: ${item.runtime_signal.tokens_per_sec.toFixed(1)} T/s</span>
            </div>
        `;
        cardGrid.appendChild(finalCard);
    });
}
```

## 4.0 部署适配与云原生沙箱契约 (Deployment Spec)

### 4.1 Google App Engine Standard 核心约束适配

Google App Engine (GAE) Standard 是一个高度受限且完全自动扩缩容的**无状态标准 WSGI 沙箱环境**。为了在这一环境下安全落地，系统必须遵守三大硬性原子契约：

#### 4.1.1 单次 HTTP 生命周期收敛限制 (Max 60s Timeout)

GAE 对标准 HTTP 请求施加了严格的超时斩断机制（通常为 60 秒）。

- **落地契约**：整个 6 层管道设计必须保持极致轻量化。
    
- **落地契约**：复杂的向量相似度计算和多路大模型异步 Fan-out 并发请求，必须全部控制在单次 Flask 请求的生命周期内。通过局部临时的 `asyncio.run()` 运行时快速收敛、合并数据并完成返回。
    

#### 4.1.2 严禁产生常驻后台守护进程 (No Background Daemon Processes)

在 GAE Standard 沙箱环境中，当 Flask 响应结束并返回 HTTP 状态码后，沙箱容器的 CPU 调度会瞬间被云端冻结或回收。

- **落地契约**：本系统内**严格禁止使用原生 `threading.Thread` 创建不随请求销毁的常驻后台守护线程**。
    
- **落地契约**：**严格禁止使用全局常驻的 `asyncio` loop 连接池**。
    
- **落地契约**：任何未在 Flask 响应返回前结束的线程，都会变成“孤儿线程”，并在容器缩容时被云端系统强制无条件 kill 释放，从而导致未定义的数据损坏或特征丢失风险。
    

#### 4.1.3 实例全无状态与动态内存扩缩容 (Stateless Topology)

GAE 实例随时可能因流量归零而缩容至 0 实例，或因并发暴增而瞬间分裂出数十个新容器。

- **落地契约**：本地内存禁止存储任何决定业务确定性的核心持久化资产。
    
- **落地契约**：评估层的反哺特征字典仅作为“当前宿主实例在当前生命周期内的运行增量先验”，不强依赖其进行跨实例同步。
    

### 4.2 生产环境无状态 Flask 视图层入口实现

Python

```
# main.py
import json
from flask import Flask, render_template, request, Response
from core.pipeline import run_routing_compiler, run_context_processor, run_execution_layer

app = Flask(__name__)

@app.route("/")
def dashboard_portal():
    """
    1.0 视图骨架路由
    利用 Jinja2 快速同步加载基础 HTML 骨架，完全无 I/O 阻塞
    """
    return render_template("dashboard.html")

@app.route("/api/stream_inference")
def stream_inference_endpoint():
    """
    2.0 异步流式网关
    将完整的 6 层管道压缩收敛在当前单个 HTTP 请求的独立沙箱生命周期内
    """
    user_prompt = request.args.get("prompt", "")

    def sse_event_generator():
        # Step A. 路由层执行确定性 CPU 编译
        execution_plan = run_routing_compiler(user_prompt)
        
        # Step B. 上下文层执行局部线程池隔离的文本脱水与裁剪
        optimized_context = run_context_processor(user_prompt)
        
        # Step C. 执行层介入：开启生命周期受限的 Ephemeral 局部事件循环
        # 利用生成器逐步将多路 g4f 返回的内容泵出给边缘前端 Vanilla JS
        # 并在生成流的最后一包数据中，附带评估层对多路结果计算出的 ranked_results
        
        # 伪代码：流式块实时外排
        for chunk_data in run_execution_layer(execution_plan, optimized_context):
            yield f"data: {json.dumps(chunk_data)}\n\n"
            
    # 返回标准的标准的 SSE HTTP 响应，GAE 容器在此维持连接，并在数据发送完毕后立刻销毁回收
    return Response(sse_event_generator(), mimetype="text/event-stream")

if __name__ == "__main__":
    # 供本地调试，部署到 GAE 时由 app.yaml 指定的 Gunicorn 自动拉起
    app.run(host="127.0.0.1", port=8080, debug=True)
```

### 4.3 App Engine 标准部署配置文件 (app.yaml)

YAML

```
# app.yaml
runtime: python39
instance_class: F2 # 分配充足的内存与轻量 CPU，确保本地 NumPy 和 scikit-learn 矩阵运算流畅

entrypoint: gunicorn -b :$PORT -w 4 -k gthread main:app

automatic_scaling:
  target_cpu_utilization: 0.65
  min_instances: 0
  max_instances: 10

env_variables:
  FLASK_ENV: 'production'
```

## 5.0 模块定位隔离边界规则

### 5.1 前端与部署层严禁执行的操作 (MUST NOT)

- **严禁在前端 JavaScript 中直接调用服务商密钥或绕过 Flask 网关直接请求大模型**（破坏整体系统分层的确定性边界）。
    
- **严禁在 Flask 视图响应外侧维持后台挂起的异步长轮询线程**。
    

### 5.2 唯一允许的系统资产行为

- 承接核心评估层传出的量化矩阵，将其通过单向事件流（SSE）传输给边缘，驱动多模型并发加载动画并规整展示前三名卡片。
    
- 在严格的 GAE 60 秒单次请求沙箱生命周期内，强制所有并发的异步协程完成合并与销毁。
    

## 6.0 系统终态总结

### 6.1 形式化定义

前端展示与部署层是整个自愈式多模型编排系统的**物理视窗沙箱**。它通过 Jinja2 骨架页保障极速的首屏体验，借助 Vanilla JS 流式流式消费引擎提供高并发动态感知，并无条件服从 Google App Engine Standard 的单次请求高收敛契约，确保整个 6 层方法论体系在无状态云原生环境中具备零驻留、高爆发的生产弹韧性。