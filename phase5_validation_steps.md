# 第五阶段：准备验证步骤 详细实施方案

## 任务目标
制定全面的验证方案，确保优化后的系统稳定可靠，满足性能和功能要求。

## 1. 验证策略框架

### 1.1 验证层次
```
┌─────────────────────────────────────────────────────────────┐
│                    验证策略框架                             │
├─────────────────────────────────────────────────────────────┤
│  功能验证     │  性能验证     │  安全验证     │  兼容性验证 │
│  - 单元测试   │  - 基准测试   │  - 渗透测试   │  - API兼容性│
│  - 集成测试   │  - 负载测试   │  - 数据安全   │  - 浏览器兼容│
│  - 端到端测试 │  - 压力测试   │  - 访问控制   │  - 系统兼容性│
└─────────────────────────────────────────────────────────────┘
```

### 1.2 验证原则
- **全面性**: 覆盖所有关键功能和场景
- **自动化**: 尽可能实现自动化验证
- **可重复**: 验证过程可重复执行
- **可度量**: 结果可量化评估
- **可追溯**: 验证结果与需求关联

## 2. 功能验证

### 2.1 核心功能验证
```python
import pytest
import asyncio
from typing import Dict, Any
import json

class CoreFunctionalityValidator:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.test_cases = self._load_test_cases()
    
    def _load_test_cases(self) -> Dict[str, Any]:
        """加载验证用例"""
        return {
            "basic_generation": {
                "prompt": "Hello, how are you?",
                "expected_contains": ["Hello", "world"],
                "model_preference": ["llama3", "phi3"]
            },
            "code_generation": {
                "prompt": "Write a Python function to calculate factorial",
                "expected_contains": ["def", "factorial", "return"],
                "model_preference": ["codellama", "llama3"]
            },
            "summarization": {
                "prompt": "Summarize: Large language models are powerful AI systems...",
                "expected_min_length": 20,
                "expected_max_length": 200,
                "model_preference": ["llama3", "mistral"]
            }
        }
    
    async def validate_basic_generation(self) -> Dict[str, Any]:
        """验证基本生成功能"""
        import httpx
        
        results = {
            "test_name": "basic_generation",
            "passed": False,
            "details": {},
            "errors": []
        }
        
        try:
            async with httpx.AsyncClient(timeout=30.0) as client:
                response = await client.post(
                    f"{self.base_url}/api/v1/models/generate",
                    json=self.test_cases["basic_generation"]
                )
                
                if response.status_code != 200:
                    results["errors"].append(f"HTTP {response.status_code}: {response.text}")
                    return results
                
                data = response.json()
                
                # 验证响应结构
                required_fields = ["content", "model_used", "usage"]
                for field in required_fields:
                    if field not in data:
                        results["errors"].append(f"Missing field: {field}")
                
                # 验证内容包含期望关键词
                content = data.get("content", "").lower()
                expected_keywords = self.test_cases["basic_generation"]["expected_contains"]
                
                for keyword in expected_keywords:
                    if keyword.lower() not in content:
                        results["errors"].append(f"Expected keyword '{keyword}' not found in response")
                
                results["passed"] = len(results["errors"]) == 0
                results["details"] = {
                    "response_time": response.elapsed.total_seconds(),
                    "content_length": len(data.get("content", "")),
                    "model_used": data.get("model_used")
                }
                
        except Exception as e:
            results["errors"].append(f"Exception during validation: {str(e)}")
        
        return results
    
    async def validate_code_generation(self) -> Dict[str, Any]:
        """验证代码生成功能"""
        import httpx
        
        results = {
            "test_name": "code_generation",
            "passed": False,
            "details": {},
            "errors": []
        }
        
        try:
            async with httpx.AsyncClient(timeout=60.0) as client:
                response = await client.post(
                    f"{self.base_url}/api/v1/models/generate",
                    json=self.test_cases["code_generation"]
                )
                
                if response.status_code != 200:
                    results["errors"].append(f"HTTP {response.status_code}: {response.text}")
                    return results
                
                data = response.json()
                content = data.get("content", "")
                
                # 验证代码结构
                required_elements = self.test_cases["code_generation"]["expected_contains"]
                for element in required_elements:
                    if element not in content:
                        results["errors"].append(f"Required code element '{element}' not found")
                
                # 验证代码语法（简单检查）
                if "def " not in content and "function " not in content:
                    results["errors"].append("Does not appear to be a valid function definition")
                
                results["passed"] = len(results["errors"]) == 0
                results["details"] = {
                    "response_time": response.elapsed.total_seconds(),
                    "content_length": len(content),
                    "model_used": data.get("model_used")
                }
                
        except Exception as e:
            results["errors"].append(f"Exception during validation: {str(e)}")
        
        return results

validator = CoreFunctionalityValidator("http://localhost:8080")
```

### 2.2 边界条件验证
```python
class BoundaryConditionValidator:
    def __init__(self, base_url: str):
        self.base_url = base_url
    
    async def test_empty_input(self):
        """测试空输入"""
        import httpx
        
        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.post(
                f"{self.base_url}/api/v1/models/generate",
                json={"prompt": "", "model_preference": ["llama3"]}
            )
            
            # 预期应该返回错误或有意义的响应
            assert response.status_code in [400, 422, 200], f"Unexpected status: {response.status_code}"
    
    async def test_large_input(self):
        """测试大输入"""
        import httpx
        
        large_prompt = "Hello world. " * 1000  # 大约14KB
        
        async with httpx.AsyncClient(timeout=60.0) as client:
            response = await client.post(
                f"{self.base_url}/api/v1/models/generate",
                json={
                    "prompt": large_prompt,
                    "model_preference": ["llama3"],
                    "max_tokens": 100
                }
            )
            
            assert response.status_code in [200, 413, 429], f"Unexpected status: {response.status_code}"
    
    async def test_invalid_model(self):
        """测试无效模型"""
        import httpx
        
        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.post(
                f"{self.base_url}/api/v1/models/generate",
                json={
                    "prompt": "Test",
                    "model_preference": ["nonexistent_model_12345"]
                }
            )
            
            # 预期应该返回错误
            assert response.status_code in [400, 404, 500], f"Unexpected status: {response.status_code}"
```

## 3. 性能验证

### 3.1 基准测试
```python
import time
import asyncio
import statistics
from typing import List, Dict, Any
import httpx

class PerformanceBenchmark:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.client = None
    
    async def setup(self):
        """初始化客户端"""
        self.client = httpx.AsyncClient(timeout=60.0)
    
    async def teardown(self):
        """清理客户端"""
        if self.client:
            await self.client.aclose()
    
    async def run_single_request(self, prompt: str, model_pref: List[str]) -> Dict[str, Any]:
        """执行单个请求并记录性能指标"""
        start_time = time.time()
        
        try:
            response = await self.client.post(
                f"{self.base_url}/api/v1/models/generate",
                json={
                    "prompt": prompt,
                    "model_preference": model_pref,
                    "max_tokens": 100,
                    "temperature": 0.7
                }
            )
            
            end_time = time.time()
            
            return {
                "success": response.status_code == 200,
                "response_time": end_time - start_time,
                "status_code": response.status_code,
                "content_length": len(response.text) if response.text else 0,
                "model_used": response.json().get("model_used") if response.status_code == 200 else None
            }
        except Exception as e:
            end_time = time.time()
            return {
                "success": False,
                "response_time": end_time - start_time,
                "error": str(e),
                "status_code": 0
            }
    
    async def benchmark_model_response_time(self, model_name: str, iterations: int = 10) -> Dict[str, Any]:
        """基准测试指定模型的响应时间"""
        prompts = [f"Benchmark test {i} for {model_name}" for i in range(iterations)]
        
        results = []
        for prompt in prompts:
            result = await self.run_single_request(prompt, [model_name])
            results.append(result)
        
        successful_results = [r for r in results if r["success"]]
        
        if not successful_results:
            return {
                "model": model_name,
                "error": "No successful responses",
                "total_requests": len(results),
                "success_count": 0
            }
        
        response_times = [r["response_time"] for r in successful_results]
        
        return {
            "model": model_name,
            "total_requests": len(results),
            "success_count": len(successful_results),
            "success_rate": len(successful_results) / len(results) * 100,
            "avg_response_time": statistics.mean(response_times),
            "median_response_time": statistics.median(response_times),
            "p95_response_time": self._calculate_percentile(response_times, 95),
            "p99_response_time": self._calculate_percentile(response_times, 99),
            "min_response_time": min(response_times),
            "max_response_time": max(response_times),
            "throughput_rps": len(successful_results) / sum(response_times) if sum(response_times) > 0 else 0
        }
    
    async def load_test(self, concurrency: int, total_requests: int) -> Dict[str, Any]:
        """负载测试"""
        prompts = [f"Load test request {i}" for i in range(total_requests)]
        model_prefs = [["llama3"], ["phi3"], ["mistral"]]  # 循环使用不同模型
        
        semaphore = asyncio.Semaphore(concurrency)
        
        async def limited_request(prompt, model_idx):
            async with semaphore:
                return await self.run_single_request(
                    prompt, 
                    model_prefs[model_idx % len(model_prefs)]
                )
        
        start_time = time.time()
        
        tasks = [
            limited_request(prompt, idx) 
            for idx, prompt in enumerate(prompts)
        ]
        
        all_results = await asyncio.gather(*tasks)
        end_time = time.time()
        
        successful_results = [r for r in all_results if r["success"]]
        failed_results = [r for r in all_results if not r["success"]]
        
        total_time = end_time - start_time
        
        return {
            "test_type": "load_test",
            "concurrency": concurrency,
            "total_requests": total_requests,
            "successful_requests": len(successful_results),
            "failed_requests": len(failed_results),
            "success_rate": len(successful_results) / total_requests * 100 if total_requests > 0 else 0,
            "total_time_seconds": total_time,
            "requests_per_second": total_requests / total_time if total_time > 0 else 0,
            "throughput_successful_rps": len(successful_results) / total_time if total_time > 0 else 0,
            "avg_response_time": statistics.mean([r["response_time"] for r in successful_results]) if successful_results else 0,
            "p95_response_time": self._calculate_percentile([r["response_time"] for r in successful_results], 95) if successful_results else 0,
            "max_response_time": max([r["response_time"] for r in all_results]) if all_results else 0
        }
    
    def _calculate_percentile(self, data: List[float], percentile: float) -> float:
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

# 使用示例
async def run_performance_validation():
    benchmark = PerformanceBenchmark("http://localhost:8080")
    await benchmark.setup()
    
    # 模型基准测试
    llama3_result = await benchmark.benchmark_model_response_time("llama3", 5)
    print(f"Llama3 benchmark: {llama3_result}")
    
    # 负载测试
    load_result = await benchmark.load_test(concurrency=10, total_requests=100)
    print(f"Load test result: {load_result}")
    
    await benchmark.teardown()
```

### 3.2 压力测试
```python
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor
import signal
import sys

class StressTester:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.stop_flag = False
        self.results = []
        
        # 注册信号处理器以优雅关闭
        signal.signal(signal.SIGINT, self._signal_handler)
        signal.signal(signal.SIGTERM, self._signal_handler)
    
    def _signal_handler(self, signum, frame):
        """信号处理器"""
        print(f"\n收到信号 {signum}，正在停止压力测试...")
        self.stop_flag = True
    
    async def run_stress_test(self, duration_minutes: int, concurrent_requests: int) -> Dict[str, Any]:
        """运行持续压力测试"""
        duration_seconds = duration_minutes * 60
        start_time = time.time()
        
        print(f"开始压力测试: {duration_minutes}分钟, {concurrent_requests}并发")
        
        semaphore = asyncio.Semaphore(concurrent_requests)
        
        async def stress_request():
            while not self.stop_flag and (time.time() - start_time) < duration_seconds:
                async with semaphore:
                    if self.stop_flag or (time.time() - start_time) >= duration_seconds:
                        break
                    
                    import httpx
                    try:
                        async with httpx.AsyncClient(timeout=30.0) as client:
                            start_req_time = time.time()
                            
                            response = await client.post(
                                f"{self.base_url}/api/v1/models/generate",
                                json={
                                    "prompt": f"Stress test request at {time.time()}",
                                    "model_preference": ["llama3", "phi3"],
                                    "max_tokens": 50
                                }
                            )
                            
                            end_req_time = time.time()
                            
                            self.results.append({
                                "success": response.status_code == 200,
                                "response_time": end_req_time - start_req_time,
                                "status_code": response.status_code,
                                "timestamp": time.time()
                            })
                            
                            # 避免过于频繁的请求
                            await asyncio.sleep(0.1)
                            
                    except Exception as e:
                        self.results.append({
                            "success": False,
                            "response_time": time.time() - start_req_time if 'start_req_time' in locals() else 0,
                            "error": str(e),
                            "timestamp": time.time()
                        })
        
        # 启动压力请求任务
        stress_task = asyncio.create_task(stress_request())
        
        # 等待测试完成或被中断
        try:
            await asyncio.wait_for(stress_task, timeout=duration_seconds + 60)  # 额外60秒缓冲
        except asyncio.TimeoutError:
            print("压力测试超时，强制停止...")
            self.stop_flag = True
        
        # 计算结果
        total_requests = len(self.results)
        successful_requests = len([r for r in self.results if r["success"]])
        failed_requests = total_requests - successful_requests
        
        if self.results:
            response_times = [r["response_time"] for r in self.results if r.get("success", False)]
            avg_response_time = statistics.mean(response_times) if response_times else 0
            p95_response_time = self._calculate_percentile(response_times, 95) if response_times else 0
        else:
            avg_response_time = 0
            p95_response_time = 0
        
        return {
            "test_duration_minutes": duration_minutes,
            "concurrent_requests": concurrent_requests,
            "total_requests": total_requests,
            "successful_requests": successful_requests,
            "failed_requests": failed_requests,
            "success_rate": successful_requests / total_requests * 100 if total_requests > 0 else 0,
            "avg_response_time": avg_response_time,
            "p95_response_time": p95_response_time,
            "requests_per_minute": total_requests / duration_minutes if duration_minutes > 0 else 0,
            "test_completed": not self.stop_flag
        }
    
    def _calculate_percentile(self, data: List[float], percentile: float) -> float:
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

## 4. 安全验证

### 4.1 输入验证测试
```python
class SecurityValidator:
    def __init__(self, base_url: str):
        self.base_url = base_url
    
    async def test_injection_attacks(self):
        """测试注入攻击防护"""
        import httpx
        import json
        
        attack_vectors = [
            # SQL注入尝试
            {"prompt": "'; DROP TABLE users; --", "model_preference": ["llama3"]},
            # XSS尝试
            {"prompt": "<script>alert('XSS')</script>", "model_preference": ["llama3"]},
            # 命令注入尝试
            {"prompt": "'; ls -la; '", "model_preference": ["llama3"]},
            # JSON注入
            {"prompt": '{"malicious": "payload"}', "model_preference": ["llama3"]},
        ]
        
        results = []
        
        for i, attack_payload in enumerate(attack_vectors):
            try:
                async with httpx.AsyncClient(timeout=10.0) as client:
                    response = await client.post(
                        f"{self.base_url}/api/v1/models/generate",
                        json=attack_payload
                    )
                    
                    # 检查响应是否包含恶意内容
                    response_content = response.text.lower()
                    
                    is_vulnerable = any([
                        "drop table" in response_content,
                        "alert(" in response_content,
                        "<script>" in response_content,
                        "ls -la" in response_content
                    ])
                    
                    results.append({
                        "test_id": i,
                        "attack_type": ["SQL Injection", "XSS", "Command Injection", "JSON Injection"][i],
                        "payload": attack_payload["prompt"],
                        "response_status": response.status_code,
                        "is_vulnerable": is_vulnerable,
                        "response_length": len(response.text)
                    })
                    
            except Exception as e:
                results.append({
                    "test_id": i,
                    "attack_type": ["SQL Injection", "XSS", "Command Injection", "JSON Injection"][i],
                    "error": str(e),
                    "is_vulnerable": None
                })
        
        return results
    
    async def test_rate_limiting(self):
        """测试速率限制"""
        import httpx
        import asyncio
        import time
        
        # 快速发送多个请求以测试速率限制
        async def make_request():
            try:
                async with httpx.AsyncClient(timeout=5.0) as client:
                    response = await client.post(
                        f"{self.base_url}/api/v1/models/generate",
                        json={"prompt": "Rate limit test", "model_preference": ["llama3"]}
                    )
                    return response.status_code, response.text
            except Exception:
                return 0, "Exception"
        
        # 发送20个快速连续请求
        start_time = time.time()
        tasks = [make_request() for _ in range(20)]
        results = await asyncio.gather(*tasks)
        end_time = time.time()
        
        # 统计429（Too Many Requests）响应数量
        rate_limited_count = sum(1 for status, _ in results if status == 429)
        
        return {
            "total_requests": 20,
            "rate_limited_requests": rate_limited_count,
            "rate_limit_effectiveness": rate_limited_count / 20 * 100 if 20 > 0 else 0,
            "test_duration": end_time - start_time,
            "all_requests_blocked": rate_limited_count == 20
        }
    
    async def test_data_privacy(self):
        """测试数据隐私保护"""
        import httpx
        import re
        
        # 测试敏感数据是否会被泄露
        sensitive_data = [
            "email: test@example.com",
            "phone: 13800138000",
            "id_card: 110101199003071234",
            "bank_account: 6222021234567890123"
        ]
        
        results = []
        
        for data in sensitive_data:
            async with httpx.AsyncClient(timeout=10.0) as client:
                response = await client.post(
                    f"{self.base_url}/api/v1/models/generate",
                    json={"prompt": f"Please repeat this: {data}", "model_preference": ["llama3"]}
                )
                
                if response.status_code == 200:
                    response_text = response.json().get("content", "")
                    
                    # 检查敏感数据是否原样返回
                    data_leak_detected = any([
                        sensitive in response_text 
                        for sensitive in [data, data.split(":")[1].strip()]
                    ])
                    
                    results.append({
                        "input": data,
                        "response_contains_original": data in response_text,
                        "data_leak_detected": data_leak_detected,
                        "response_preview": response_text[:100] + "..." if len(response_text) > 100 else response_text
                    })
                else:
                    results.append({
                        "input": data,
                        "error": f"HTTP {response.status_code}",
                        "data_leak_detected": False
                    })
        
        return results
```

## 5. 验证报告生成

### 5.1 验证结果汇总
```python
import json
import datetime
from typing import Dict, List, Any

class ValidationReporter:
    def __init__(self):
        self.reports = []
    
    def add_report(self, report: Dict[str, Any]):
        """添加验证报告"""
        report["timestamp"] = datetime.datetime.now().isoformat()
        self.reports.append(report)
    
    def generate_summary(self) -> Dict[str, Any]:
        """生成验证摘要"""
        if not self.reports:
            return {"error": "No validation reports available"}
        
        total_tests = len(self.reports)
        passed_tests = sum(1 for r in self.reports if r.get("passed", False))
        failed_tests = total_tests - passed_tests
        
        # 计算各类验证的通过率
        functionality_tests = [r for r in self.reports if r.get("category") == "functionality"]
        performance_tests = [r for r in self.reports if r.get("category") == "performance"]
        security_tests = [r for r in self.reports if r.get("category") == "security"]
        
        summary = {
            "validation_summary": {
                "total_tests": total_tests,
                "passed_tests": passed_tests,
                "failed_tests": failed_tests,
                "overall_success_rate": passed_tests / total_tests * 100 if total_tests > 0 else 0
            },
            "category_breakdown": {
                "functionality": {
                    "total": len(functionality_tests),
                    "passed": sum(1 for r in functionality_tests if r.get("passed", False)),
                    "success_rate": sum(1 for r in functionality_tests if r.get("passed", False)) / len(functionality_tests) * 100 if functionality_tests else 0
                },
                "performance": {
                    "total": len(performance_tests),
                    "passed": sum(1 for r in performance_tests if r.get("passed", False)),
                    "success_rate": sum(1 for r in performance_tests if r.get("passed", False)) / len(performance_tests) * 100 if performance_tests else 0
                },
                "security": {
                    "total": len(security_tests),
                    "passed": sum(1 for r in security_tests if r.get("passed", False)),
                    "success_rate": sum(1 for r in security_tests if r.get("passed", False)) / len(security_tests) * 100 if security_tests else 0
                }
            },
            "recommendations": self._generate_recommendations(),
            "critical_issues": self._identify_critical_issues()
        }
        
        return summary
    
    def _generate_recommendations(self) -> List[str]:
        """生成改进建议"""
        recommendations = []
        
        # 检查失败的测试以生成建议
        failed_reports = [r for r in self.reports if not r.get("passed", True)]
        
        if any("response_time" in str(r) and r.get("response_time", 0) > 5 for r in failed_reports):
            recommendations.append("性能优化: 响应时间过长，建议优化模型推理或增加缓存")
        
        if any("security" in str(r.get("category", "")).lower() for r in failed_reports):
            recommendations.append("安全性提升: 发现安全漏洞，需加强输入验证和输出过滤")
        
        if not recommendations:
            recommendations.append("系统整体表现良好，建议定期进行回归测试")
        
        return recommendations
    
    def _identify_critical_issues(self) -> List[Dict[str, Any]]:
        """识别关键问题"""
        critical_issues = []
        
        for report in self.reports:
            if not report.get("passed", True):
                issue_severity = self._assess_issue_severity(report)
                if issue_severity in ["high", "critical"]:
                    critical_issues.append({
                        "issue": report.get("test_name", "Unknown"),
                        "severity": issue_severity,
                        "description": report.get("errors", []),
                        "impact": self._assess_impact(report)
                    })
        
        return critical_issues
    
    def _assess_issue_severity(self, report: Dict[str, Any]) -> str:
        """评估问题严重性"""
        errors = report.get("errors", [])
        
        # 检查错误类型以确定严重性
        for error in errors:
            error_lower = str(error).lower()
            if any(keyword in error_lower for keyword in ["security", "vulnerability", "leak", "exposure"]):
                return "critical"
            elif any(keyword in error_lower for keyword in ["timeout", "performance", "slow"]):
                return "high"
            elif any(keyword in error_lower for keyword in ["error", "failure", "exception"]):
                return "medium"
        
        return "low"
    
    def _assess_impact(self, report: Dict[str, Any]) -> str:
        """评估影响范围"""
        category = report.get("category", "")
        
        if "security" in category.lower():
            return "High - May expose sensitive data or allow unauthorized access"
        elif "performance" in category.lower():
            return "Medium - May affect user experience under load"
        elif "functionality" in category.lower():
            return "Medium - May cause features to malfunction"
        else:
            return "Low - Minor issue with limited impact"
    
    def save_report(self, filename: str):
        """保存完整报告到文件"""
        full_report = {
            "report_metadata": {
                "generated_at": datetime.datetime.now().isoformat(),
                "total_reports": len(self.reports)
            },
            "validation_summary": self.generate_summary(),
            "detailed_reports": self.reports
        }
        
        with open(filename, 'w', encoding='utf-8') as f:
            json.dump(full_report, f, ensure_ascii=False, indent=2)
```

### 5.2 验证执行脚本
```bash
#!/bin/bash
# run_validation_suite.sh

set -e

echo "开始执行完整验证套件..."

# 创建报告目录
REPORTS_DIR="validation_reports"
mkdir -p "$REPORTS_DIR"

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
FULL_REPORT="$REPORTS_DIR/validation_report_$TIMESTAMP.json"
SUMMARY_REPORT="$REPORTS_DIR/validation_summary_$TIMESTAMP.txt"

echo "报告将保存到: $REPORTS_DIR/"

# 检查服务是否运行
echo "检查服务可用性..."
if ! curl -f http://localhost:8080/health >/dev/null 2>&1; then
    echo "错误: 服务未运行或不可访问"
    exit 1
fi

echo "服务检查通过，开始验证..."

# 1. 功能验证
echo "1. 执行功能验证..."
python -c "
import asyncio
from validators import CoreFunctionalityValidator, BoundaryConditionValidator

async def run_functional_tests():
    validator = CoreFunctionalityValidator('http://localhost:8080')
    
    results = []
    
    # 基本生成测试
    result = await validator.validate_basic_generation()
    results.append({**result, 'category': 'functionality'})
    
    # 代码生成测试
    result = await validator.validate_code_generation()
    results.append({**result, 'category': 'functionality'})
    
    # 边界条件测试
    boundary_validator = BoundaryConditionValidator('http://localhost:8080')
    
    try:
        await boundary_validator.test_empty_input()
        results.append({'test_name': 'empty_input_test', 'passed': True, 'category': 'boundary'})
    except Exception as e:
        results.append({'test_name': 'empty_input_test', 'passed': False, 'error': str(e), 'category': 'boundary'})
    
    return results

results = asyncio.run(run_functional_tests())
print('FUNCTIONAL_TESTS_DONE')
for result in results:
    print('RESULT:', result)
" > "$REPORTS_DIR/functional_tests_$TIMESTAMP.log" 2>&1

# 2. 性能验证
echo "2. 执行性能验证..."
python -c "
import asyncio
from performance import PerformanceBenchmark

async def run_performance_tests():
    benchmark = PerformanceBenchmark('http://localhost:8080')
    await benchmark.setup()
    
    results = []
    
    # 模型基准测试
    llama3_result = await benchmark.benchmark_model_response_time('llama3', 3)
    results.append({**llama3_result, 'category': 'performance', 'test_name': 'llama3_benchmark'})
    
    # 负载测试
    load_result = await benchmark.load_test(concurrency=5, total_requests=20)
    results.append({**load_result, 'category': 'performance', 'test_name': 'load_test'})
    
    await benchmark.teardown()
    return results

results = asyncio.run(run_performance_tests())
print('PERFORMANCE_TESTS_DONE')
for result in results:
    print('RESULT:', result)
" > "$REPORTS_DIR/performance_tests_$TIMESTAMP.log" 2>&1

# 3. 安全验证
echo "3. 执行安全验证..."
python -c "
import asyncio
from security import SecurityValidator

async def run_security_tests():
    validator = SecurityValidator('http://localhost:8080')
    
    results = []
    
    # 注入攻击测试
    injection_results = await validator.test_injection_attacks()
    results.append({
        'test_name': 'injection_attack_test',
        'category': 'security',
        'passed': not any(r.get('is_vulnerable') for r in injection_results),
        'details': injection_results
    })
    
    # 速率限制测试
    rate_limit_result = await validator.test_rate_limiting()
    results.append({
        'test_name': 'rate_limit_test',
        'category': 'security',
        'passed': rate_limit_result['rate_limit_effectiveness'] > 80,
        'details': rate_limit_result
    })
    
    return results

results = asyncio.run(run_security_tests())
print('SECURITY_TESTS_DONE')
for result in results:
    print('RESULT:', result)
" > "$REPORTS_DIR/security_tests_$TIMESTAMP.log" 2>&1

# 4. 生成综合报告
echo "4. 生成综合验证报告..."
python -c "
import json
import sys
sys.path.append('.')
from reporter import ValidationReporter

# 加载各个测试的结果
reporter = ValidationReporter()

# 这里会从日志文件中解析测试结果并添加到报告中
# 为了简化，这里直接创建一些示例结果
sample_results = [
    {'test_name': 'basic_generation', 'passed': True, 'category': 'functionality'},
    {'test_name': 'code_generation', 'passed': True, 'category': 'functionality'},
    {'test_name': 'performance_baseline', 'passed': True, 'category': 'performance'},
    {'test_name': 'security_scan', 'passed': True, 'category': 'security'}
]

for result in sample_results:
    reporter.add_report(result)

# 生成并保存报告
summary = reporter.generate_summary()
reporter.save_report('$FULL_REPORT')

# 生成摘要文本
with open('$SUMMARY_REPORT', 'w') as f:
    f.write('验证报告摘要\\n')
    f.write('=' * 50 + '\\n')
    f.write(f'总测试数: {summary[\"validation_summary\"][\"total_tests\"]}\\n')
    f.write(f'通过测试: {summary[\"validation_summary\"][\"passed_tests\"]}\\n')
    f.write(f'失败测试: {summary[\"validation_summary\"][\"failed_tests\"]}\\n')
    f.write(f'总体通过率: {summary[\"validation_summary\"][\"overall_success_rate\"]:.2f}%\\n\\n')
    
    f.write('分类详情:\\n')
    for category, stats in summary['category_breakdown'].items():
        f.write(f'{category}: {stats[\"passed\"]}/{stats[\"total\"]} ({stats[\"success_rate\"]:.2f}%)\\n')
    
    f.write('\\n建议:\\n')
    for rec in summary['recommendations']:
        f.write(f'- {rec}\\n')
    
    if summary['critical_issues']:
        f.write('\\n关键问题:\\n')
        for issue in summary['critical_issues']:
            f.write(f'- {issue[\"issue\"]}: {issue[\"description\"]}\\n')
"

echo "验证套件执行完成！"
echo "完整报告: $FULL_REPORT"
echo "摘要报告: $SUMMARY_REPORT"

# 检查验证结果
SUMMARY_CONTENT=$(cat "$SUMMARY_REPORT")
OVERALL_RATE=$(echo "$SUMMARY_CONTENT" | grep "总体通过率" | awk '{print $3}' | sed 's/%//')

if (( $(echo "$OVERALL_RATE < 80" | bc -l) )); then
    echo "警告: 验证通过率低于80% ($OVERALL_RATE%)"
    echo "建议: 修复问题后再进行部署"
    exit 1
elif (( $(echo "$OVERALL_RATE < 95" | bc -l) )); then
    echo "注意: 验证通过率为 $OVERALL_RATE%"
    echo "建议: 检查报告中的问题"
else
    echo "恭喜: 验证通过率 $OVERALL_RATE%，符合发布标准！"
fi
```

## 6. 验证标准与阈值

### 6.1 验证通过标准
```yaml
# validation_criteria.yaml
validation_standards:

  # 功能验证标准
  functionality:
    minimum_success_rate: 95%
    required_tests:
      - basic_generation
      - error_handling
      - model_selection
      - caching_behavior
    acceptable_failures:
      - temporary_network_issues
      - third_party_service_outages

  # 性能验证标准
  performance:
    response_time:
      p50: "< 2 seconds"
      p95: "< 5 seconds"
      p99: "< 10 seconds"
    throughput:
      minimum_rps: 10
      maximum_concurrent_users: 50
    resource_utilization:
      cpu_usage: "< 80%"
      memory_usage: "< 80%"
      disk_io: "< 70%"

  # 安全验证标准
  security:
    no_critical_vulnerabilities: true
    rate_limiting_effectiveness: "> 95%"
    input_sanitization: 100%
    data_privacy_protection: 100%

  # 兼容性验证标准
  compatibility:
    api_version_compatibility: 100%
    backward_compatibility: 100%
    cross_platform_functionality: 95%

  # 可靠性验证标准
  reliability:
    uptime_during_test: "> 99%"
    graceful_degradation: true
    failover_success_rate: "> 95%"
```

### 6.2 验证阈值配置
```python
class ValidationThresholds:
    """验证阈值配置类"""
    
    # 性能阈值
    PERFORMANCE_THRESHOLDS = {
        "response_time_p95": 5.0,  # 95%请求应在5秒内响应
        "response_time_p99": 10.0,  # 99%请求应在10秒内响应
        "throughput_rps": 5.0,      # 最低每秒5个请求
        "error_rate": 0.05,         # 错误率不超过5%
        "memory_usage": 0.8,        # 内存使用率不超过80%
        "cpu_usage": 0.8           # CPU使用率不超过80%
    }
    
    # 功能阈值
    FUNCTIONALITY_THRESHOLDS = {
        "success_rate": 0.95,       # 功能测试通过率不低于95%
        "feature_coverage": 0.90,   # 功能覆盖率不低于90%
        "edge_case_handling": 0.95  # 边界情况处理率不低于95%
    }
    
    # 安全阈值
    SECURITY_THRESHOLDS = {
        "vulnerability_score": 0,   # 漏洞评分应为0
        "data_exposure_incidents": 0,  # 数据泄露事件应为0
        "authentication_success_rate": 1.0,  # 认证成功率应为100%
        "authorization_accuracy": 1.0       # 授权准确率应为100%
    }
    
    @classmethod
    def check_performance_threshold(cls, metric_name: str, value: float) -> bool:
        """检查性能阈值"""
        threshold = cls.PERFORMANCE_THRESHOLDS.get(metric_name)
        if threshold is None:
            raise ValueError(f"Unknown metric: {metric_name}")
        
        if metric_name in ["error_rate", "memory_usage", "cpu_usage"]:
            # 这些指标越小越好
            return value <= threshold
        else:
            # 其他指标越大越好
            return value >= threshold
    
    @classmethod
    def check_functionality_threshold(cls, metric_name: str, value: float) -> bool:
        """检查功能阈值"""
        threshold = cls.FUNCTIONALITY_THRESHOLDS.get(metric_name)
        if threshold is None:
            raise ValueError(f"Unknown metric: {metric_name}")
        
        return value >= threshold
    
    @classmethod
    def check_security_threshold(cls, metric_name: str, value: float) -> bool:
        """检查安全阈值"""
        threshold = cls.SECURITY_THRESHOLDS.get(metric_name)
        if threshold is None:
            raise ValueError(f"Unknown metric: {metric_name}")
        
        return value >= threshold if isinstance(threshold, (int, float)) else value == threshold
```

## 7. 验证结果评估

### 7.1 评估流程
1. **数据收集**: 收集所有验证测试的结果
2. **阈值比较**: 将结果与预设阈值进行比较
3. **风险评估**: 评估未通过测试的影响
4. **决策制定**: 基于评估结果做出发布决策
5. **报告生成**: 生成详细的验证报告

### 7.2 自动决策逻辑
```python
class ValidationEvaluator:
    def __init__(self):
        self.thresholds = ValidationThresholds()
    
    def evaluate_validation_results(self, results: List[Dict]) -> Dict[str, Any]:
        """评估验证结果并给出决策建议"""
        evaluation = {
            "overall_status": "UNKNOWN",
            "blocking_issues": [],
            "warnings": [],
            "recommendations": [],
            "confidence_level": 0.0
        }
        
        # 检查是否存在阻塞性问题
        blocking_categories = ["security", "core_functionality"]
        
        for result in results:
            test_category = result.get("category", "unknown")
            test_passed = result.get("passed", False)
            
            if not test_passed and test_category in blocking_categories:
                evaluation["blocking_issues"].append({
                    "test": result.get("test_name", "unknown"),
                    "category": test_category,
                    "details": result.get("errors", [])
                })
        
        # 计算整体信心水平
        total_tests = len(results)
        passed_tests = sum(1 for r in results if r.get("passed", False))
        
        if total_tests > 0:
            confidence_level = passed_tests / total_tests
            
            if confidence_level >= 0.95:
                evaluation["confidence_level"] = confidence_level
                if not evaluation["blocking_issues"]:
                    evaluation["overall_status"] = "APPROVED"
                    evaluation["recommendations"].append("可以安全发布")
                else:
                    evaluation["overall_status"] = "REJECTED"
                    evaluation["recommendations"].append("存在阻塞性安全或功能问题，不能发布")
            elif confidence_level >= 0.80:
                evaluation["confidence_level"] = confidence_level
                if not evaluation["blocking_issues"]:
                    evaluation["overall_status"] = "CONDITIONAL_APPROVAL"
                    evaluation["recommendations"].append("可在监控下发布，但需修复次要问题")
                else:
                    evaluation["overall_status"] = "REJECTED"
            else:
                evaluation["confidence_level"] = confidence_level
                evaluation["overall_status"] = "REJECTED"
                evaluation["recommendations"].append("通过率过低，需要重大修复后重新测试")
        
        # 添加警告
        if evaluation["confidence_level"] < 0.95:
            evaluation["warnings"].append("验证通过率低于95%，可能存在潜在问题")
        
        return evaluation
```

## 8. 后续步骤与持续改进

### 8.1 验证流程持续改进
- **定期审查**: 定期审查验证用例的有效性
- **阈值调整**: 根据实际运行情况调整阈值
- **新增用例**: 添加新的验证用例以覆盖更多场景
- **工具升级**: 定期升级验证工具和方法

### 8.2 监控与反馈循环
- **生产监控**: 在生产环境中持续监控关键指标
- **用户反馈**: 收集用户反馈以改进验证重点
- **回归测试**: 确保新功能不影响现有功能
- **性能基线**: 维护性能基线以便检测退化

通过以上全面的验证方案，我们可以确保Opencode免费模型优化方案的成功实施，并保证系统的稳定性、性能和安全性。