# 子领域分析：配置系统

## 1. 功能概述
配置系统管理用户设置、项目配置和运行时参数，支持多层配置覆盖。

## 2. 核心组件

### 2.1 配置层次
```
配置优先级（高到低）：

1. 命令行参数
   └─ --model claude-3-opus
   
2. 环境变量
   └─ CLAUDE_CODE_MODEL=claude-3-opus
   
3. 项目配置
   └─ .claude/config.json
   
4. 用户配置
   └─ ~/.claude/config.json
   
5. 系统默认
   └─ 代码中硬编码
```

### 2.2 配置管理器
```python
class ConfigManager:
    """配置管理器"""
    
    def __init__(self):
        self.layers = [
            CommandLineConfig(),
            EnvironmentConfig(),
            ProjectConfig(),
            UserConfig(),
            DefaultConfig(),
        ]
    
    def get(self, key: str, default=None):
        """按优先级获取配置"""
        for layer in self.layers:
            if key in layer:
                return layer[key]
        return default
    
    def set(self, key: str, value):
        """设置用户配置"""
        self.layers[3][key] = value
```

### 2.3 配置验证
```python
from dataclasses import dataclass

@dataclass
class Config:
    """配置结构"""
    model: str = 'claude-3-sonnet'
    max_tokens: int = 8192
    temperature: float = 0.7
    
    def validate(self):
        """验证配置"""
        assert self.max_tokens > 0
        assert 0 <= self.temperature <= 1

class ConfigValidator:
    """配置验证器"""
    
    def validate(self, config: Config) -> bool:
        """验证配置有效性"""
        try:
            config.validate()
            return True
        except AssertionError as e:
            print(f"Invalid config: {e}")
            return False
```

## 3. 配置流程
```
程序启动
  │
  ├─ 1. 加载默认配置
  │     └─ 代码中定义
  │
  ├─ 2. 加载用户配置
  │     └─ ~/.claude/config.json
  │
  ├─ 3. 加载项目配置
  │     └─ .claude/config.json
  │
  ├─ 4. 加载环境变量
  │     └─ CLAUDE_CODE_*
  │
  ├─ 5. 解析命令行
  │     └─ argparse
  │
  └─ 6. 验证配置
        └─ validate()
```

## 4. 关键设计
### 4.1 为什么多层覆盖？
| 层 | 用途 | 示例 |
|----|------|------|
| 默认 | 代码定义 | model='sonnet' |
| 用户 | 个人偏好 | model='opus' |
| 项目 | 团队统一 | temperature=0.5 |
| 环境 | CI/CD | api_key=*** |
| 命令 | 临时 | --max-tokens 16K |

**优点**:
- 灵活性
- 可覆盖
- 渐进配置
### 4.2 为什么 dataclass?
```python
@dataclass
class Config:
    model: str
    max_tokens: int
```
**优点**:
- 类型安全
- 自动验证
- 可序列化
- IDE 支持
### 4.3 配置 vs 硬编码？
| 场景 | 配置 | 硬编码 |
|------|------|--------|
| API 密钥 | ✓ 环境 | ✗ |
| 模型名称 | ✓ 用户 | ✗ |
| 超时秒数 | ✓ 项目 | ✗ |
| 重试次数 | ✓ 默认 | 可 |
| 版本号 | ✗ | ✓ |
## 5. 评价
| 维度 | 评分 | 说明 |
|------|--------|------|
| 灵活性 | ★★★★★ | 多层 |
| 安全性 | ★★★☆☆ | 需验证 |
| 可维护 | ★★★★☆ | 清晰 |
| 用户体验 | ★★★★☆ | 渐进 |
| 复杂性 | ★★★☆☆ | 适中 |
## 6. 关键代码
```python
# 加载配置
def load_config():
    config = {}
    
    # 默认
    config.update(DEFAULT_CONFIG)
    
    # 用户
    user_file = Path.home() / '.claude' / 'config.json'
    if user_file.exists():
        config.update(json.loads(user_file.read_text()))
    
    # 项目
    project_file = Path('.claude/config.json')
    if project_file.exists():
        config.update(json.loads(project_file.read_text()))
    
    return config

# 保存配置
def save_config(config):
    config_file = Path.home() / '.claude' / 'config.json'
    config_file.parent.mkdir(parents=True, exist_ok=True)
    config_file.write_text(json.dumps(config, indent=2))
```
## 7. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|------|-----------|-------------|
| 层次 | 5 层 | 概念 |
| 格式 | JSON + 环境 | 概念 |
| 验证 | Zod | dataclass |
| 动态 | 支持 | 概念 |
| 文档 | 自动生成 | 手动 |
## 8. 代码片段
```python
# 配置类
@dataclass
class ClaudeConfig:
    model: str = 'claude-3-sonnet'
    max_tokens: int = 8192
    temperature: float = 0.7
    
    @classmethod
    def from_env(cls):
        return cls(
            model=os.getenv('CLAUDE_MODEL', 'claude-3-sonnet'),
            max_tokens=int(os.getenv('CLAUDE_MAX_TOKENS', '8192')),
        )
```
