# OpenClaw 云端模型API信息汇总

基于系统配置文件分析，整理所有云端模型服务的相关信息，为后续实施智能路由模式提供数据支持。

## 服务健康状态概览

根据 `service_status.json` 的最新检查结果（2026-02-01T06:16:24Z）：

- **健康服务**: 1/6 (Moonshot)
- **不健康服务**: 5/6 (Google, DeepSeek, Qwen, OpenAI, VolcEngine)

## 云端模型服务详细信息

### 1. Qwen (通义千问)

#### 配置信息
- **服务提供商**: Qwen Portal
- **API端点**: https://portal.qwen.ai/v1
- **认证方式**: OAuth
- **当前状态**: 不健康
- **模型列表**:
  - `qwen-portal/coder-model` (Qwen Coder) - 上下文窗口: 128000
  - `qwen-portal/vision-model` (Qwen Vision) - 上下文窗口: 128000
- **别名**: `qwen` (coder-model)

#### 当前使用情况
- 作为OpenClaw的primary模型
- 配置在fallback列表中

### 2. Moonshot (月之暗面)

#### 配置信息
- **服务提供商**: Moonshot
- **API端点**: https://api.moonshot.cn/v1
- **认证方式**: API密钥 (已配置)
- **当前状态**: 健康 ✓
- **模型列表**:
  - `moonshot/moonshot-v1-8k` (Moonshot V1 8k) - 上下文窗口: 8192
  - `moonshot/moonshot-v1-32k` (Moonshot V1 32k) - 上下文窗口: 32768
  - `moonshot/moonshot-v1-128k` (Moonshot V1 128k) - 上下文窗口: 未知
- **别名**: `kimi` (moonshot-v1-8k)

#### 当前使用情况
- 配置在fallback列表首位
- 系统中最健康的云端服务

### 3. DeepSeek

#### 配置信息
- **服务提供商**: DeepSeek
- **API端点**: https://api.deepseek.com
- **认证方式**: API密钥 (已配置)
- **当前状态**: 不健康
- **模型列表**:
  - `deepseek/deepseek-chat` (DeepSeek Chat V3) - 上下文窗口: 64000
  - `deepseek/deepseek-reasoner` (DeepSeek Reasoner R1) - 推理模型，上下文窗口: 64000
- **别名**: `deepseek-v3` (chat), `deepseek-r1` (reasoner)

#### 当前使用情况
- 配置在fallback列表中
- 包含推理专用模型

### 4. OpenAI

#### 配置信息
- **服务提供商**: OpenAI
- **API端点**: 标准OpenAI API端点
- **认证方式**: API密钥 (已配置)
- **当前状态**: 不健康
- **模型列表**:
  - `openai/gpt-4o` (GPT-4o)

#### 当前使用情况
- 配置在fallback列表中
- 作为高级推理备选

### 5. Google (Gemini系列)

#### 配置信息
- **服务提供商**: Google (Gemini)
- **API端点**: 标准Google API端点
- **认证方式**: API密钥 (已配置)
- **当前状态**: 不健康
- **模型列表**:
  - `google/gemini-3-pro-preview` (Gemini 3 Pro Preview) - 别名: `gemini`
  - `google/gemini-2.5-flash` (Gemini 2.5 Flash) - 别名: `flash`
  - `google/gemini-2.5-flash-lite` (Gemini 2.5 Flash Lite) - 别名: `lite`
  - `google-vertex/gemini-1.5-pro` (Vertex AI Gemini 1.5 Pro)
  - `google-vertex/gemini-1.5-flash` (Vertex AI Gemini 1.5 Flash)

#### 当前使用情况
- 配置在fallback列表中
- 提供多种性能和成本权衡的选择

### 6. VolcEngine (字节跳动豆包)

#### 配置信息
- **服务提供商**: VolcEngine (字节跳动)
- **API端点**: https://ark.cn-beijing.volces.com/api/v3
- **认证方式**: API密钥 (已配置)
- **当前状态**: 不健康
- **模型列表**:
  - `volcengine/doubao-pro` (Doubao Pro) - 上下文窗口: 32768
  - `volcengine/doubao-lite` (Doubao Lite)
- **别名**: `doubao-pro`, `doubao-lite`

#### 当前使用情况
- 配置在fallback列表末尾
- 作为备用选项

## API轮换配置

根据 `api_rotation_config.json`:

### 轮换策略
- **轮换间隔**: 60分钟
- **健康检查间隔**: 300秒
- **最大重试次数**: 3次
- **超时时间**: 30秒
- **备选策略**: round_robin

### 优先级顺序
1. `moonshot/moonshot-v1-8k` (唯一健康服务)
2. `qwen-portal/coder-model`
3. `google/gemini-2.5-flash`
4. `openai/gpt-4o`
5. `deepseek/deepseek-chat`
6. `volcengine/doubao-pro`

### 错误关键词监测
- 429, rate limit, RateLimit, Too Many Requests
- error, Error, connection refused, timeout, timed out

## 智能路由实施建议

### 1. 服务可用性分级
- **A级（推荐）**: Moonshot (当前唯一健康服务)
- **B级（待修复）**: Qwen, OpenAI, Google, DeepSeek, Volcengine

### 2. 任务类型匹配
- **简单任务**: 优先使用本地Ollama模型
- **中等复杂度**: 使用Moonshot (健康且成本适中)
- **高复杂度**: 修复后的Qwen或OpenAI/Gemini
- **推理任务**: DeepSeek Reasoner (如果修复)

### 3. 降级策略
1. 本地模型 (首选)
2. Moonshot (健康云端模型)
3. 其他云端模型按健康状况排序
4. 备用云端模型

### 4. 监控和告警
- 持续监控各服务商的健康状态
- 当健康服务数量低于阈值时发出告警
- 自动尝试重新配置不健康的服务

## 注意事项

1. **安全性**: 实际API密钥未在此文档中显示，出于安全考虑
2. **配置同步**: 修改配置后需确保所有相关文件同步更新
3. **健康检查**: 定期验证各服务商的实际可用性
4. **成本控制**: 不同服务商的定价模型不同，需监控使用成本

## 后续步骤

1. 优先修复健康状态不正常的API服务
2. 实施智能路由算法，根据任务类型选择最优模型
3. 建立动态权重机制，根据响应质量和成本调整路由策略
4. 实现自动故障转移和恢复机制