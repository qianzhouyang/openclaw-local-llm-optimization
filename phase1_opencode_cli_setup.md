# 第一阶段：配置Opencode CLI 详细实施方案

## 任务目标
设置并配置Opencode命令行界面，为后续模型测试和集成做好准备。

## 1. 环境检查与准备

### 1.1 系统依赖检查
在开始安装之前，需要确认系统具备以下依赖：
- Python 3.8 或更高版本
- pip 包管理器
- Git（用于从仓库克隆）
- 网络连接（用于下载包）

```bash
# 检查Python版本
python3 --version

# 检查pip
pip --version

# 检查Git
git --version
```

### 1.2 创建虚拟环境（推荐）
为避免包冲突，建议创建独立的虚拟环境：

```bash
# 创建虚拟环境
python3 -m venv opencode_env

# 激活虚拟环境
source opencode_env/bin/activate  # Linux/Mac
# 或
opencode_env\Scripts\activate  # Windows
```

## 2. Opencode CLI 安装

### 2.1 通过pip安装
```bash
# 安装Opencode CLI
pip install opencode-cli

# 如果是特定开发版本
pip install git+https://github.com/username/opencode-cli.git
```

### 2.2 从源码安装
```bash
# 克隆仓库
git clone https://github.com/username/opencode-cli.git
cd opencode-cli

# 安装开发版本
pip install -e .
```

## 3. 基本配置

### 3.1 初始化配置
```bash
# 初始化配置
opencode init

# 或者直接设置API密钥
opencode config set api_key YOUR_API_KEY_HERE
```

### 3.2 配置文件位置
通常配置文件位于：
- Linux/Mac: ~/.config/opencode/config.json
- Windows: C:\Users\{username}\.config\opencode\config.json

## 4. 验证安装

### 4.1 版本检查
```bash
opencode --version
```

### 4.2 基础命令测试
```bash
# 查看可用命令
opencode --help

# 测试连接
opencode models list
```

## 5. 常见问题排查

### 5.1 权限问题
如果遇到权限问题，尝试使用用户安装：
```bash
pip install --user opencode-cli
```

### 5.2 网络问题
如果无法访问外部仓库，可能需要配置代理：
```bash
pip install --proxy http://proxy.company.com:port opencode-cli
```

## 6. 预期输出

成功安装后应能看到类似输出：
```
$ opencode --version
opencode-cli, version 1.0.0

$ opencode models list
Available models:
- model1
- model2
- model3
```

## 7. 安装脚本示例

创建一个自动化安装脚本：

```bash
#!/bin/bash

echo "开始安装 Opencode CLI..."

# 检查Python
if ! command -v python3 &> /dev/null; then
    echo "错误: 未找到Python3，请先安装Python3"
    exit 1
fi

# 检查pip
if ! command -v pip &> /dev/null; then
    echo "错误: 未找到pip，请先安装pip"
    exit 1
fi

# 创建虚拟环境
python3 -m venv opencode_env
source opencode_env/bin/activate

# 升级pip
pip install --upgrade pip

# 安装Opencode CLI
pip install opencode-cli

# 验证安装
if command -v opencode &> /dev/null; then
    echo "Opencode CLI 安装成功!"
    opencode --version
else
    echo "Opencode CLI 安装失败!"
    exit 1
fi

echo "安装完成。请运行 'source opencode_env/bin/activate' 激活环境。"
```

## 8. 后续步骤

安装完成后，下一步是：
1. 获取API密钥（如果是付费服务）
2. 配置默认参数
3. 测试基本功能
4. 记录配置信息以供后续使用