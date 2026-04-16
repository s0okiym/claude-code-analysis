# 子领域分析：测试策略系统

## 1. 功能概述
测试策略定义了如何验证代码正确性，包括单元测试、集成测试和端到端测试。

## 2. 核心组件

### 2.1 测试结构
```
tests/
├── __init__.py
├── test_models.py          # 数据类测试
├── test_port_manifest.py   # 清单生成测试
├── test_query_engine.py    # 查询引擎测试
├── test_commands.py        # 命令测试
└── test_integration.py     # 集成测试
```

### 2.2 单元测试
```python
import unittest
from src.models import Subsystem, PortingModule

class TestModels(unittest.TestCase):
    """数据模型测试"""
    
    def test_subsystem_creation(self):
        """测试 Subsystem 创建"""
        s = Subsystem('main', 'src/main', 1, 'entry')
        self.assertEqual(s.name, 'main')
        self.assertEqual(s.file_count, 1)
    
    def test_subsystem_immutable(self):
        """测试不可变性"""
        s = Subsystem('main', 'src/main', 1, 'entry')
        with self.assertRaises(FrozenInstanceError):
            s.name = 'changed'
    
    def test_porting_module_defaults(self):
        """测试默认值"""
        m = PortingModule('test', 'desc', 'src/test.py')
        self.assertEqual(m.status, 'planned')
```

### 2.3 集成测试
```python
class TestIntegration(unittest.TestCase):
    """集成测试"""
    
    def test_full_workflow(self):
        """测试完整工作流"""
        # 1. 构建清单
        manifest = build_port_manifest()
        
        # 2. 创建引擎
        engine = QueryEnginePort(manifest)
        
        # 3. 生成摘要
        summary = engine.render_summary()
        
        # 4. 验证
        self.assertIn('# Python Porting', summary)
        self.assertIn('Command surface:', summary)
```

### 2.4 Mock 测试
```python
from unittest.mock import patch, MagicMock

class TestWithMock(unittest.TestCase):
    """使用 Mock 的测试"""
    
    @patch('src.port_manifest.Path.rglob')
    def test_scan_with_mock(self, mock_rglob):
        """模拟文件扫描"""
        # 设置 mock
        mock_rglob.return_value = [
            MagicMock(is_file=lambda: True, name='main.py'),
            MagicMock(is_file=lambda: True, name='models.py'),
        ]
        
        # 执行
        manifest = build_port_manifest()
        
        # 验证
        self.assertEqual(manifest.total_python_files, 2)
```

## 3. 测试流程
```
代码变更
  │
  ▼
运行测试
  │
  ├─ 单元测试
  │   ├─ 快速
  │   └─ 隔离
  │
  ├─ 集成测试
  │   ├─ 较慢
  │   └─ 验证交互
  │
  └─ 端到端
      ├─ 最慢
      └─ 完整场景
```

## 4. 关键设计
### 4.1 为什么 unittest?
| 框架 | 优点 | 缺点 |
|------|------|------|
| unittest | 标准库 | 稍繁琐 |
| pytest | 简洁 | 需安装 |
| nose | 插件多 | 过时 |

**选择理由**:
- 零依赖
- 标准库
- 足够当前需求
### 4.2 为什么分层测试？
| 类型 | 范围 | 速度 | 目的 |
|------|------|------|------|
| 单元 | 函数/类 | 快 | 验证逻辑 |
| 集成 | 模块间 | 中 | 验证交互 |
| E2E | 完整流程 | 慢 | 验证场景 |

**金字塔原则**: 单元 > 集成 > E2E
### 4.3 为什么 Mock?
```python
@patch('src.port_manifest.Path.rglob')
def test(self, mock):
    ...
```
**优点**:
- 隔离依赖
- 控制输入
- 快速执行
- 确定性
## 5. 评价
| 维度 | 评分 | 说明 |
|------|--------|------|
| 覆盖 | ★★★☆☆ | 需补充 |
| 速度 | ★★★★☆ | 快速 |
| 可靠 | ★★★★☆ | 确定性 |
| 维护 | ★★★★☆ | 清晰 |
| 完整 | ★★☆☆☆ | 基础 |
## 6. 关键代码
```python
# 运行测试
python -m unittest discover -s tests -v

# 覆盖率
python -m coverage run -m unittest discover
python -m coverage report
```
## 7. 与原始对比
| 特性 | 原始 (TS) | 本实现 (Py) |
|------|-----------|-------------|
| 框架 | Jest | unittest |
| 覆盖 | 高 | 低 |
| Mock | jest.mock | unittest.mock |
| E2E | 有 | 无 |
| CI | 集成 | 概念 |
## 8. 代码片段
```python
# 测试示例
class TestPortManifest(unittest.TestCase):
    def test_empty_dir(self):
        with tempfile.TemporaryDirectory() as tmp:
            manifest = build_port_manifest(Path(tmp))
            self.assertEqual(manifest.total_python_files, 0)
    
    def test_with_files(self):
        with tempfile.TemporaryDirectory() as tmp:
            Path(tmp, 'a.py').write_text('')
            Path(tmp, 'b.py').write_text('')
            
            manifest = build_port_manifest(Path(tmp))
            self.assertEqual(manifest.total_python_files, 2)
```
