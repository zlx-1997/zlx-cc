---
name: code-reasoning
description: 代码推理技能——面对一段看不懂的源码时，按三步流程拆解：输入输出契约→决策树拆解→自动trace验证。触发词：看懂、这段代码、推理、分析代码、怎么理解、拆解、trace、trace一下、走一遍。
triggers: ["看懂", "这段代码", "推理", "分析代码", "怎么理解", "拆解", "trace", "trace一下", "走一遍"]
---

> 🎓 学习模式 — 使用此技能前自动切换到学习场景

```bash
python "C:/Users/zlx_1/.claude/plugins/cache/zlx/common/toggle-mode.py" learn
```

# Code Reasoning — 代码推理

> 面对一段看不懂的源码时，按三步流程拆解。不是教你怎么写代码，是教你怎么**读**代码。

## 触发方式

- **自动**：teach-concept 教学过程中，你说"这段代码看不懂" / "这写的啥" 时自动介入
- **手动**：任何时候说 `分析 resolve_ref 的源码` 或 `帮我走一遍 scope.py`

## 和 diagnose 的区别

| | code-reasoning | diagnose |
|--|---------------|----------|
| 起点 | "这段代码我看不懂" | "这段代码有 bug" |
| 目标 | 理解代码的正确行为 | 找到 bug 的根因 |
| 输出 | 契约 + 拆解 + trace 验证 | 根因 + 修复方案 |

## 三步推理流程

### Step 1：输入输出契约

只关注函数/类的**边界**，不关心内部实现。

**三句话回答：**
1. 输入：接收什么参数？什么类型？传 None 会怎样？
2. 输出：返回什么类型？None 代表什么？
3. 边界：哪些情况会提前返回 / 抛异常？

产出：函数签名 + 三句话描述边界。

### Step 2：逻辑拆解

把函数体拆成**决策树**——每层一个判断 + 分支，不写长步骤列表。

```
[问] 有没有传 table_alias？
  ├── 有 → [问] 在当前作用域找到了吗？
  │       ├── 找到了 → 返回值池引用
  │       └── 没找到 → 向外回溯（递归）
  └── 没有 → 模糊匹配
```

产出：3-5 行的决策树注释，标注每个判断点的输入和输出。

### Step 3：自动 trace 验证

**不动手插 print。** 用 `sys.settrace()` 自动录每行代码的变量变化。

```python
import sys

def _make_tracer():
    """创建跟踪器：每执行一行代码，打印行号和局部变量"""
    def trace_func(frame, event, arg):
        if event == 'line':
            vals = {k: v for k, v in frame.f_locals.items()
                    if not k.startswith('_')}
            print(f"  [L{frame.f_lineno}] {vals}")
        return trace_func
    return trace_func

# 用跟踪器包裹目标函数
sys.settrace(_make_tracer())
result = resolve_ref("a", table_alias="t1")
sys.settrace(None)
```

输出：
```
>> resolve_ref({'col_name': 'a', 'table_alias': 't1'})
  [L12] {'col_name': 'a', 'table_alias': 't1'}
  [L14] {'col_name': 'a', 'table_alias': 't1', 'alias': 't1'}
  [L15] {'col_name': 'a', 'table_alias': 't1', 'alias': 't1'}
  ...
<< resolve_ref = ${pool_pick(t_a_pool, $row_idx)}
```

每一行都能看到当前变量值——等价于 IDE 里一步一步 debug，但不需要人点下一步。

## 三不原则

| 不做什么 | 原因 |
|---------|------|
| 不解释语法（if/for/列表推导式） | 那是 teach-concept 的职责 |
| 不批量读整个文件 | 一次只聚焦一个函数/类 |
| 不猜替代方案 | 只分析当前代码，不提重构建议 |

## 产出物

每次 code-reasoning 完成后，将发现记录到进度地图：
- 分析的函数名
- 拆解后的决策树
- trace 验证结果（每个分支是否走到）

## 和 teach-concept 的集成

```
teach-concept 讲课时
  ↓ 学生说"这段源码看不懂"
code-reasoning 自动触发
  ↓ 完成 Step 1→2→3（不插一行 print）
回到 teach-concept 继续
```
