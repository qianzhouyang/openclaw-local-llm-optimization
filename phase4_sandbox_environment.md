# 第四阶段：设计沙盒测试环境 详细实施方案

## 任务目标
建立安全的隔离测试环境，用于验证模型集成效果，确保不影响生产系统。

## 1. 沙盒环境架构设计

### 1.1 架构概览
```
┌─────────────────────────────────────────────────────────────┐
│                    生产环境                                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  应用服务    │  │  数据库     │  │  缓存      │         │
│  │  (Production)│  │  (Production)│  │  (Production)│         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────┐
│                   沙盒测试环境                               │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  应用服务    │  │  数据库     │  │  缓存      │         │
│  │  (Sandbox)  │  │  (Sandbox)  │  │  (Sandbox) │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│  ┌─────────────────────────────────────────────────────────┤
│  │  AI模型服务                                           │
│  │  - 本地模型 (Ollama)                                  │
│  │  - 模拟API服务                                        │
│  │  - 第三方API沙盒                                      │
│  └─────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┤
│  │  监控与日志                                           │
│  │  - Prometheus + Grafana                                │
│  │  - ELK Stack (Elasticsearch, Logstash, Kibana)       │
│  └─────────────────────────────────────────────────────────┤
└─────────────────────────────────────────────────────────────┘
```

### 1.2 隔离级别
- **网络隔离**: 沙盒环境与生产环境完全隔离
- **数据隔离**: 使用独立的数据存储
- **资源隔离**: 限制CPU、内存、磁盘使用
- **权限隔离**: 严格控制访问权限

## 2. 沙盒环境基础设施

### 2.1 Docker Compose 配置
```yaml
# docker-compose.sandbox.yml
version: '3.8'

services:
  # 应用服务
  sandbox-app:
    build: 
      context: .
      dockerfile: Dockerfile
    container_name: sandbox-app
    ports:
      - "8080:8000"
    environment:
      - ENVIRONMENT=sandbox
      - DATABASE_URL=postgresql://user:password@db-sandbox:5432/sandbox_db
      - REDIS_URL=redis://redis-sandbox:6379
      - MODEL_PROVIDER_ENDPOINT=http://model-service:11434
    depends_on:
      - db-sandbox
      - redis-sandbox
    volumes:
      - ./logs:/app/logs
    networks:
      - sandbox-net
    resources:
      limits:
        cpus: '1.0'
        memory: 1G
      reservations:
        cpus: '0.5'
        memory: 512M

  # 沙盒数据库
  db-sandbox:
    image: postgres:15
    container_name: db-sandbox
    environment:
      POSTGRES_DB: sandbox_db
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5433:5432"  # 使用不同端口避免冲突
    volumes:
      - db-sandbox-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - sandbox-net
    resources:
      limits:
        memory: 512M

  # 沙盒缓存
  redis-sandbox:
    image: redis:7-alpine
    container_name: redis-sandbox
    ports:
      - "6380:6379"  # 使用不同端口
    volumes:
      - redis-sandbox-data:/data
    networks:
      - sandbox-net
    resources:
      limits:
        memory: 256M

  # 模型服务 (Ollama)
  model-service:
    image: ollama/ollama:latest
    container_name: sandbox-model-service
    ports:
      - "11435:11434"  # 使用不同端口
    volumes:
      - model-sandbox-data:/root/.ollama
    networks:
      - sandbox-net
    resources:
      limits:
        memory: 2G
        devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]

  # 监控服务
  prometheus:
    image: prom/prometheus:latest
    container_name: sandbox-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus-sandbox.yml:/etc/prometheus/prometheus.yml
      - prometheus-sandbox-data:/prometheus
    networks:
      - sandbox-net
    resources:
      limits:
        memory: 512M

  grafana:
    image: grafana/grafana:latest
    container_name: sandbox-grafana
    ports:
      - "3001:3000"  # 使用不同端口避免冲突
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-sandbox-data:/var/lib/grafana
    networks:
      - sandbox-net
    depends_on:
      - prometheus
    resources:
      limits:
        memory: 512M

networks:
  sandbox-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

volumes:
  db-sandbox-data:
  redis-sandbox-data:
  model-sandbox-data:
  prometheus-sandbox-data:
  grafana-sandbox-data:
```

### 2.2 网络安全配置
```yaml
# network-security.yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sandbox-isolation
  namespace: sandbox
spec:
  podSelector:
    matchLabels:
      environment: sandbox
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          environment: sandbox  # 只允许沙盒内部通信
  egress:
  - to:
    - podSelector:
        matchLabels:
          environment: sandbox  # 只允许沙盒内部通信
    ports:
    - protocol: TCP
      port: 53  # DNS
    - protocol: TCP
      port: 443 # HTTPS for model downloads
```

## 3. 数据管理策略

### 3.1 数据脱敏
```python
import re
import hashlib
from faker import Faker

class DataSanitizer:
    def __init__(self):
        self.fake = Faker()
    
    def sanitize_text(self, text: str) -> str:
        """脱敏文本中的敏感信息"""
        # 替换邮箱
        text = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', 
                      lambda m: f"{self.fake.email()}", text)
        
        # 替换手机号
        text = re.sub(r'1[3-9]\d{9}', 
                      lambda m: f"{self.fake.phone_number()}", text)
        
        # 替换身份证号
        text = re.sub(r'\b\d{17}[\dXx]\b', 
                      lambda m: f"{self.fake.ssn()}", text)
        
        return text
    
    def hash_sensitive_fields(self, data: dict, fields: list) -> dict:
        """哈希处理敏感字段"""
        sanitized = data.copy()
        for field in fields:
            if field in sanitized:
                sanitized[field] = hashlib.sha256(
                    str(sanitized[field]).encode()
                ).hexdigest()[:16]
        return sanitized
```

### 3.2 测试数据生成
```python
import json
from datetime import datetime, timedelta
from faker import Faker

class TestDataGenerator:
    def __init__(self):
        self.fake = Faker()
    
    def generate_user_data(self, count: int) -> list:
        """生成测试用户数据"""
        users = []
        for _ in range(count):
            users.append({
                "id": self.fake.uuid4(),
                "username": self.fake.user_name(),
                "email": self.fake.email(),
                "created_at": self.fake.date_time_between(start_date='-1y', end_date='now').isoformat(),
                "last_login": self.fake.date_time_between(start_date='-1m', end_date='now').isoformat()
            })
        return users
    
    def generate_interaction_data(self, count: int) -> list:
        """生成测试交互数据"""
        interactions = []
        for _ in range(count):
            interactions.append({
                "id": self.fake.uuid4(),
                "user_id": self.fake.uuid4(),
                "prompt": self.fake.text(max_nb_chars=200),
                "response": self.fake.text(max_nb_chars=500),
                "timestamp": self.fake.date_time_between(start_date='-1m', end_date='now').isoformat(),
                "model_used": self.fake.random_element(elements=["llama3", "phi3", "gpt-3.5-turbo"])
            })
        return interactions
    
    def export_to_json(self, data: list, filename: str):
        """导出数据到JSON文件"""
        with open(filename, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
```

## 4. 模型测试环境

### 4.1 本地模型部署
```bash
#!/bin/bash
# setup_sandbox_models.sh

set -e

echo "设置沙盒环境模型..."

# 拉取测试模型
echo "拉取Llama3模型..."
ollama pull llama3

echo "拉取Phi-3模型..."
ollama pull phi3

echo "拉取Mistral模型..."
ollama pull mistral

# 验证模型可用性
echo "验证模型可用性..."
for model in llama3 phi3 mistral; do
    echo "测试 $model..."
    echo "Hello" | ollama run $model 2>/dev/null
    if [ $? -eq 0 ]; then
        echo "$model 部署成功"
    else
        echo "$model 部署失败"
    fi
done

echo "沙盒模型设置完成"
```

### 4.2 模拟API服务
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import asyncio
import random

app = FastAPI(title="Sandbox Model API Simulator")

class ModelRequest(BaseModel):
    model: str
    prompt: str
    max_tokens: Optional[int] = 100
    temperature: Optional[float] = 0.7

@app.post("/v1/chat/completions")
async def simulate_chat_completion(request: ModelRequest):
    """模拟OpenAI兼容的API响应"""
    # 模拟API延迟
    delay = random.uniform(0.5, 2.0)
    await asyncio.sleep(delay)
    
    # 模拟不同的错误情况
    if random.random() < 0.05:  # 5% 错误率
        raise HTTPException(status_code=500, detail="Simulated API error")
    
    if random.random() < 0.02:  # 2% 限流
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    
    # 生成模拟响应
    simulated_response = f"这是对 '{request.prompt}' 的模拟响应。模型: {request.model}, 长度: {request.max_tokens} tokens。"
    
    return {
        "id": f"chatcmpl-{random.randint(100000, 999999)}",
        "object": "chat.completion",
        "created": int(asyncio.get_event_loop().time()),
        "model": request.model,
        "choices": [{
            "index": 0,
            "message": {
                "role": "assistant",
                "content": simulated_response
            },
            "finish_reason": "stop"
        }],
        "usage": {
            "prompt_tokens": len(request.prompt.split()),
            "completion_tokens": request.max_tokens,
            "total_tokens": len(request.prompt.split()) + request.max_tokens
        }
    }

@app.get("/v1/models")
async def list_models():
    """列出可用模型"""
    return {
        "object": "list",
        "data": [
            {"id": "gpt-3.5-turbo", "object": "model"},
            {"id": "llama3", "object": "model"},
            {"id": "phi3", "object": "model"}
        ]
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8001)
```

## 5. 监控与日志

### 5.1 日志收集配置
```yaml
# monitoring/fluent-bit-sandbox.conf
[SERVICE]
    Flush         1
    Log_Level     info
    Daemon        off
    Parsers_File  parsers.conf

[INPUT]
    Name              tail
    Path              /app/logs/*.log
    Parser            json
    Refresh_Interval  5
    Mem_Buf_Limit     5MB

[FILTER]
    Name                modify
    Match               *
    Add                 environment sandbox

[OUTPUT]
    Name          stdout
    Match         *
    Format        json_lines

[OUTPUT]
    Name          file
    Match         *
    Path          /var/log/sandbox-collected/
    File          logs.json
```

### 5.2 监控面板配置
```yaml
# monitoring/grafana-dashboard-sandbox.json
{
  "dashboard": {
    "id": null,
    "title": "Sandbox Environment Dashboard",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(sandbox_requests_total[5m])",
            "legendFormat": "{{model}}"
          }
        ]
      },
      {
        "id": 2,
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(sandbox_response_time_seconds_bucket[5m]))",
            "legendFormat": "{{model}} P95"
          }
        ]
      },
      {
        "id": 3,
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(sandbox_errors_total[5m]) / rate(sandbox_requests_total[5m]) * 100",
            "legendFormat": "Error Rate %"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    }
  }
}
```

## 6. 测试用例设计

### 6.1 功能测试用例
```python
import pytest
import asyncio
from httpx import AsyncClient

@pytest.mark.asyncio
class TestSandboxEnvironment:
    async def test_basic_model_call(self):
        """测试基本模型调用功能"""
        async with AsyncClient(base_url="http://localhost:8080") as client:
            response = await client.post("/api/v1/models/generate", json={
                "prompt": "Hello, world!",
                "model_preference": ["llama3", "phi3"]
            })
            
            assert response.status_code == 200
            data = response.json()
            assert "content" in data
            assert "model_used" in data
    
    async def test_model_fallback(self):
        """测试模型故障转移功能"""
        # 模拟主模型不可用，验证是否正确切换到备用模型
        pass
    
    async def test_cache_functionality(self):
        """测试缓存功能"""
        async with AsyncClient(base_url="http://localhost:8080") as client:
            # 发送相同的请求两次，验证第二次是否使用缓存
            first_response = await client.post("/api/v1/models/generate", json={
                "prompt": "Cached test",
                "model_preference": ["llama3"]
            })
            
            second_response = await client.post("/api/v1/models/generate", json={
                "prompt": "Cached test",  # 相同的提示
                "model_preference": ["llama3"]
            })
            
            # 验证响应时间差异（缓存应该更快）
            assert abs(first_response.elapsed.total_seconds() - 
                      second_response.elapsed.total_seconds()) < 0.1
    
    async def test_rate_limiting(self):
        """测试速率限制功能"""
        # 发送大量请求以触发速率限制
        pass
    
    async def test_error_handling(self):
        """测试错误处理"""
        async with AsyncClient(base_url="http://localhost:8080") as client:
            response = await client.post("/api/v1/models/generate", json={
                "prompt": "",  # 空提示
                "model_preference": ["invalid_model"]
            })
            
            # 验证返回适当的错误
            assert response.status_code in [400, 422, 500]
```

### 6.2 性能测试用例
```python
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor
import statistics

class PerformanceTester:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.client = None
    
    async def setup(self):
        """设置测试客户端"""
        from httpx import AsyncClient
        self.client = AsyncClient(base_url=self.base_url)
    
    async def single_request(self, prompt: str) -> dict:
        """发送单个请求并记录性能指标"""
        start_time = time.time()
        
        try:
            response = await self.client.post("/api/v1/models/generate", json={
                "prompt": prompt,
                "model_preference": ["llama3"]
            })
            
            end_time = time.time()
            
            return {
                "success": response.status_code == 200,
                "response_time": end_time - start_time,
                "status_code": response.status_code,
                "size_bytes": len(response.content) if response.content else 0
            }
        except Exception as e:
            end_time = time.time()
            return {
                "success": False,
                "response_time": end_time - start_time,
                "error": str(e),
                "status_code": 0
            }
    
    async def load_test(self, num_requests: int, concurrency: int = 10):
        """执行负载测试"""
        prompts = [f"Performance test request {i}" for i in range(num_requests)]
        
        # 创建并发任务
        semaphore = asyncio.Semaphore(concurrency)
        
        async def limited_request(prompt):
            async with semaphore:
                return await self.single_request(prompt)
        
        start_time = time.time()
        tasks = [limited_request(prompt) for prompt in prompts]
        results = await asyncio.gather(*tasks)
        end_time = time.time()
        
        total_time = end_time - start_time
        
        # 计算统计信息
        successful_requests = [r for r in results if r["success"]]
        failed_requests = [r for r in results if not r["success"]]
        response_times = [r["response_time"] for r in successful_requests]
        
        stats = {
            "total_requests": num_requests,
            "successful_requests": len(successful_requests),
            "failed_requests": len(failed_requests),
            "success_rate": len(successful_requests) / num_requests * 100,
            "total_time": total_time,
            "requests_per_second": num_requests / total_time,
            "avg_response_time": statistics.mean(response_times) if response_times else 0,
            "median_response_time": statistics.median(response_times) if response_times else 0,
            "p95_response_time": self.calculate_percentile(response_times, 95) if response_times else 0,
            "min_response_time": min(response_times) if response_times else 0,
            "max_response_time": max(response_times) if response_times else 0
        }
        
        return stats
    
    def calculate_percentile(self, data, percentile):
        """计算百分位数"""
        if not data:
            return 0
        sorted_data = sorted(data)
        index = (percentile / 100) * (len(sorted_data) - 1)
        if index.is_integer():
            return sorted_data[int(index)]
        else:
            lower = sorted_data[int(index)]
            upper = sorted_data[min(int(index) + 1, len(sorted_data) - 1)]
            return lower + (upper - lower) * (index - int(index))
```

## 7. 安全措施

### 7.1 访问控制
```yaml
# security/access-control-sandbox.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sandbox-access-control
data:
  # IP白名单
  whitelist_ips: |
    127.0.0.1
    10.0.0.0/8
    172.16.0.0/12
    192.168.0.0/16
  
  # API密钥管理
  api_keys: |
    {
      "sandbox_user": {
        "key": "sk-sandbox-xxx",
        "permissions": ["read", "write"],
        "rate_limit": 100
      }
    }
```

### 7.2 资源限制
```yaml
# security/resource-limits-sandbox.yml
apiVersion: v1
kind: LimitRange
metadata:
  name: sandbox-resource-limits
  namespace: sandbox
spec:
  limits:
  - default:
      cpu: 500m
      memory: 1Gi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
  - max:
      storage: 10Gi
    type: PersistentVolumeClaim
```

## 8. 自动化部署脚本

### 8.1 沙盒环境一键部署
```bash
#!/bin/bash
# deploy_sandbox.sh

set -e

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ENVIRONMENT=${1:-sandbox}

echo "开始部署 $ENVIRONMENT 环境..."

# 检查必要工具
command -v docker >/dev/null 2>&1 || { echo >&2 "Docker is required but not installed. Aborting."; exit 1; }
command -v docker-compose >/dev/null 2>&1 || { echo >&2 "Docker Compose is required but not installed. Aborting."; exit 1; }

# 创建必要的目录
mkdir -p logs monitoring data

# 生成配置文件
cat > .env.sandbox << EOF
ENVIRONMENT=$ENVIRONMENT
DB_PASSWORD=sandbox_password
REDIS_PASSWORD=sandbox_redis_password
MODEL_ENDPOINT=http://model-service:11434
EOF

# 启动沙盒环境
echo "启动沙盒环境服务..."
docker-compose -f docker-compose.sandbox.yml up -d --build

# 等待服务启动
echo "等待服务启动..."
sleep 10

# 检查服务状态
SERVICES=("sandbox-app" "db-sandbox" "redis-sandbox" "model-service")
for service in "${SERVICES[@]}"; do
    if [ "$(docker inspect -f '{{.State.Running}}' $service 2>/dev/null)" != "true" ]; then
        echo "错误: 服务 $service 未能正常启动"
        docker logs $service
        exit 1
    fi
done

echo "沙盒环境部署完成！"
echo "应用访问地址: http://localhost:8080"
echo "数据库端口: 5433"
echo "Redis端口: 6380"
echo "模型服务端口: 11435"
echo "监控面板: http://localhost:3001 (admin/admin)"

# 显示运行状态
docker-compose -f docker-compose.sandbox.yml ps
```

### 8.2 环境清理脚本
```bash
#!/bin/bash
# cleanup_sandbox.sh

ENVIRONMENT=${1:-sandbox}
CONFIRMATION=${2:-"no"}

if [ "$CONFIRMATION" != "yes" ]; then
    echo "警告: 此操作将删除 $ENVIRONMENT 环境的所有数据！"
    echo "要继续，请运行: $0 $ENVIRONMENT yes"
    exit 1
fi

echo "停止并删除 $ENVIRONMENT 环境..."

# 停止服务
docker-compose -f docker-compose.sandbox.yml down -v

# 删除相关镜像（可选）
# docker rmi $(docker images --filter "reference=*/sandbox*" -q) 2>/dev/null || true

# 清理卷
docker volume prune -f

# 清理网络
docker network prune -f

echo "沙盒环境清理完成！"
```

## 9. 测试执行流程

### 9.1 测试执行脚本
```bash
#!/bin/bash
# run_sandbox_tests.sh

set -e

echo "开始执行沙盒环境测试..."

# 等待服务完全启动
echo "等待服务启动..."
sleep 15

# 运行单元测试
echo "运行单元测试..."
python -m pytest tests/unit/ -v

# 运行集成测试
echo "运行集成测试..."
python -m pytest tests/integration/ -v

# 运行性能测试
echo "运行性能测试..."
python performance_test.py --concurrency 5 --requests 100

# 运行安全测试
echo "运行安全测试..."
python security_test.py

# 生成测试报告
echo "生成测试报告..."
mkdir -p reports
pytest tests/ --html=reports/test_report.html --self-contained-html

echo "所有测试完成！报告保存在 reports/ 目录下。"
```

## 10. 后续步骤

沙盒环境完成后：
1. 在沙盒环境中进行全面测试
2. 验证所有功能按预期工作
3. 进行性能基准测试
4. 准备生产环境部署方案
5. 制定回滚和应急处理计划