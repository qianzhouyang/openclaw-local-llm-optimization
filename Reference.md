下面是根据 2026‑02‑05 前后 X（Twitter）/Reddit + 中文技术社区（知乎/CSDN/腾讯云）正在流传和被大量转发/讨论的内容，整理出的「OpenClaw 最新本地大语言模型配置方案 + 节省 Token 实战总结」。
（说明：X/Reddit 上直接点名 OpenClaw 的长帖不多，真正的“热度源头”其实是中文技术圈 + OpenClaw 官方文档 + GitHub 实操贴，很多 X 推文只是转这些方案的截图或链接。）

一图总览：当前最被认可的 Token 节省组合

✅ 热帖共识方案
本地 LLM + 本地 Memory（Hybrid / QMD）+ 托管模型兜底

OpenClaw
 ├─ 主模型：本地 LLM（Ollama / LM Studio / vLLM）
 ├─ 记忆：本地 Hybrid Memory 或 QMD
 ├─ 备用模型：Claude / Gemini / GPT（仅必要时调用）
 └─ models.mode = merge


一、为什么大家都在“本地化”OpenClaw（热帖共识）
在 Reddit / X / 博客的高频总结中，Token 爆炸的根本原因被反复提到：

OpenClaw 是 Agent，不是 Chat
会注入：

系统提示（system）
工具说明
技能上下文
历史记忆


导致：

Claude / GPT 几轮就打满上下文
Token 成本不可控



解决思路只有两个方向：

✅ 缩小“进入 prompt 的内容”
✅ 让一部分推理由本地完成

➡ 这正是「本地 LLM + 本地记忆检索」的价值 [alishui.com], [laplusda.com]

二、热帖 1：OpenClaw + 本地 LLM（最小闭环）
🔥 最常被转发的方案
Ollama / LM Studio → OpenClaw（OpenAI 兼容接口）
1️⃣ 本地运行模型（示例）
Shellollama pull qwen2.5:7bollama serveShow more lines
或 LM Studio（GUI，更稳定）

开启 Local Server
默认端口：http://127.0.0.1:1234/v1 [docs.openclaw.ai]


2️⃣ OpenClaw 配置本地模型
JSON{  "models": {    "mode": "merge",    "providers": {      "local": {        "baseUrl": "http://127.0.0.1:11434/v1",        "apiKey": "sk-local",        "api": "openai-completions",        "models": [          {            "id": "qwen2.5:7b",            "name": "Qwen2.5 Local",            "contextWindow": 32768,            "maxTokens": 8192          }        ]      }    }  },  "agents": {    "defaults": {      "model": {        "primary": "local/qwen2.5:7b"      }    }  }}Show more lines
✅ Token 成本：0
❌ 推理能力有限（复杂 Agent 任务不稳）

三、热帖 2：真正节省 Token 的关键 —— 本地 Memory / QMD
X / Reddit / 中文社区已经形成共识：

“本地模型 ≠ 真正省 Token，
本地 Memory 才是 10× 减少 Token 的核心。”

✅ 两条路线

















路线适合场景OpenClaw 原生 Hybrid Memory中小规模笔记QMD（Project Golem）大量 Markdown / 长期运行 Agent

🔥 QMD 热帖重点（反复被引用）

BM25 + 向量 + LLM rerank
只返回相关片段
不把整个 MEMORY.md 塞进 prompt
Token 使用量 ↓ 90% [alishui.com]

典型对比（热帖示例）

















方式Token直接加载 MEMORY.md~2000QMD 搜索命中段落~200

四、热帖 3：最佳实践 —— 混合模式（业内共识）
这是目前讨论度最高、失败率最低的配置：
JSON{  "models": {    "mode": "merge"  },  "agents": {    "defaults": {      "model": {        "primary": "local/qwen2.5:7b",        "fallbacks": [          "anthropic/claude-sonnet-4-5"        ]      }    }  }}Show more lines
优点（被疯狂推荐的原因）：

本地模型承担 80% 日常思考
Claude 只在：

复杂规划
长链推理
卡死/幻觉时
被调用


Token 使用量下降 70–90% [docs.openclaw.ai]


五、微博 / reddit 高频踩坑总结（非常重要）
❌ 错误 1：用小模型 + 超大 Context

7B + 128k → 幻觉 + Prompt 注入风险

❌ 错误 2：本地模型当 Claude 用

高复杂 Agent 必炸

✅ 正解

本地模型：执行、摘要、常识
托管模型：决策、规划、复杂工具调用


六、结论（给你一个“可抄作业”的答案）

2026 年初 OpenClaw 社区的主流共识是：
Token 节省 ≠ 单纯不用云模型，而是
「本地 LLM + 本地记忆检索 + 托管模型兜底」
