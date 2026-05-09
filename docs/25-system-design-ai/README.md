# 📌 AI 场景系统设计

> 大厂二面/三面必考，AI 特有的系统设计变体，当前仓库完全缺失。本模块填补这个空白。

## 📋 目录

### Q1: 设计一个百万 DAU 的 AI 客服系统（核心高频考题）
### Q2: 设计企业知识库 RAG 平台（多租户 + 权限隔离）
### Q3: 设计一个 LLM API 网关（限流 + 路由 + 计费）
### Q4: 如何设计 AI 任务队列系统（避免超时、保证顺序）
### Q5: 设计一个 AI 内容审核系统（实时 + 离线双链路）

---

## 更新记录

| 日期 | 更新内容 |
|------|----------|
| 2026-05-09 | 新增 AI 场景系统设计模块（Q1-Q5）|

---

*版本: v1.0 | 更新: 2026-05-09 | by 二狗子 🐕*

---

### Q1: 设计一个百万 DAU 的 AI 客服系统（核心高频考题）

<details>
<summary>💡 答案要点</summary>

**题目理解：**

```
百万 DAU 的 AI 客服系统：
- 日活用户：100 万
- 峰值 QPS：约 1 万（按 10% 同时在线估算）
- 核心目标：低成本、高可用、快速响应
```

**整体架构：**

```
用户请求
    ↓
┌─────────────────────────────────────────────┐
│              CDN / 边缘节点                  │
│         （静态资源 + 就近接入）               │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│           负载均衡（Nginx/LB）               │
│        健康检查 + 熔断 + SSL 终结            │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│           API 网关层                         │
│    认证鉴权 │ 限流 │ 路由 │ 日志             │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│           应用服务层（无状态）                │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│   │ 服务实例1 │  │ 服务实例2 │  │ 服务实例N │   │
│   └─────────┘  └─────────┘  └─────────┘   │
└─────────────────────────────────────────────┘
    ↓
┌──────────┐  ┌──────────┐  ┌──────────┐
│ LLM 网关  │  │ RAG 服务  │  │ 会话存储  │
│(多模型路由)│  │(知识检索) │  │ (Redis)  │
└──────────┘  └──────────┘  └──────────┘
    ↓
┌──────────┐  ┌──────────┐
│ LLM API  │  │ 知识库   │
│(OpenAI等)│  │(向量数据库)│
└──────────┘  └──────────┘
```

**核心组件设计：**

**1. API 网关（限流 + 鉴权）：**

```python
# 限流策略：令牌桶 + 用户维度
@app.middleware
async def rate_limit_middleware(request: Request, call_next):
    user_id = get_user_id(request)
    
    # 令牌桶：每秒 10 个请求
    key = f"rate_limit:{user_id}"
    allow = await redis.incr(key)
    if allow == 1:
        await redis.expire(key, 1)
    
    if allow > 10:  # 超过每秒 10 请求
        return JSONResponse(
            status_code=429,
            content={"error": "rate limit exceeded"}
        )
    
    return await call_next(request)
```

**2. LLM 网关（多模型路由）：**

```python
class ModelRouter:
    """智能路由：根据请求类型选择最优模型"""
    
    ROUTING_RULES = {
        # 简单问答 → 便宜模型
        "simple_qa": {"model": "gpt-4o-mini", "cost": 0.001},
        # 复杂推理 → 贵但准
        "complex_reasoning": {"model": "gpt-4o", "cost": 0.01},
        # 代码生成 → 代码专用模型
        "code_gen": {"model": "claude-3.5-sonnet", "cost": 0.008},
        # 国内用户 → 国产模型
        "china": {"model": "qwen-plus", "cost": 0.004},
    }
    
    def route(self, request: ChatRequest) -> str:
        # 根据特征路由
        if request.is_code_related:
            return self.ROUTING_RULES["code_gen"]["model"]
        if request.user_region == "china":
            return self.ROUTING_RULES["china"]["model"]
        if request.complexity == "high":
            return self.ROUTING_RULES["complex_reasoning"]["model"]
        return self.ROUTING_RULES["simple_qa"]["model"]
```

**3. RAG 增强（知识库检索）：**

```
用户问题
    ↓
Embedding 模型 → 向量检索（Pinecone/Milvus）
    ↓
Top-K 相关文档 chunks
    ↓
注入 Prompt：[Context] + [用户问题]
    ↓
LLM 生成答案
```

**4. 会话管理（对话上下文）：**

```python
# 对话历史存储：Redis + 定期持久化
class SessionManager:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.MAX_TURNS = 20  # 限制上下文长度
    
    async def get_context(self, session_id: str) -> list[Message]:
        key = f"session:{session_id}"
        history = await self.redis.lrange(key, 0, -1)
        
        # 如果太长，做摘要压缩
        if len(history) > self.MAX_TURNS:
            older = await self.summarize(history[:-self.MAX_TURNS])
            return older + history[-self.MAX_TURNS:]
        
        return [json.loads(m) for m in history]
    
    async def add_message(self, session_id: str, role: str, content: str):
        key = f"session:{session_id}"
        await self.redis.rpush(key, json.dumps({"role": role, "content": content}))
        await self.redis.expire(key, 86400 * 30)  # 30天过期
```

**成本估算（百万 DAU）：**

| 成本项 | 计算 | 月成本（万元）|
|--------|------|--------------|
| LLM API | 100万 DAU × 5次/天 × 1K tokens × $0.002/1K | 30 |
| 向量检索 | 100万 × 5次/天 × 0.1元/千次 | 1.5 |
| 计算资源 | 50台 4C8G 云服务器 | 8 |
| CDN + 带宽 | | 5 |
| **合计** | | **~45万/月** |

**高可用设计：**

```
多活部署（两地三中心）：
北京 ──── 上海
 │           │
Active    Standby
   \        /
    \      /
     切换
```

- **熔断**：下游 LLM 响应慢 → 自动切换备选模型
- **降级**：高峰期 → 关闭 RAG，直接用通用 LLM 回答
- **重试**：返回 5xx → 指数退避重试，最多重试 3 次

**面试话术：**

> "百万 DAU AI 客服系统的核心是'分层 + 限流 + 降级'。我的设计分五层：CDN 做静态资源和入口，API 网关做认证和限流，应用层无状态可以弹性扩缩，LLM 网关做智能路由省成本，RAG 层做知识增强。关键数据：百万 DAU 一天 500 万次调用，LLM 成本占 70%，所以路由优化很重要——简单问题用 mini 模型，成本降 10 倍。面试时能说出'令牌桶限流 + LLM 路由 + RAG 知识增强'三件套，说明你有生产级系统设计能力。"

</details>

---

*版本: v1.0 | 更新: 2026-05-09 | by 二狗子 🐕*
