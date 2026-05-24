---
name: practice-apply
description: 应用实践技能——概念学完后写生产级代码（_applied.py），将知识转化为可复用的函数/类。触发词：应用实践、写代码、生产代码、实战、怎么用、applied、自己写一个。
triggers: ["应用实践", "写代码", "生产代码", "实战", "怎么用", "applied", "自己写一个"]
---

## 模式切换

> 🎓 学习模式 — 使用此技能前自动切换到学习场景

```bash
python "C:/Users/zlx_1/.claude/plugins/cache/zlx/common/toggle-mode.py" learn
```

继续前先运行以上命令切换为学习模式（关闭 tdd 门禁等开发钩子）。

# Practice Apply — 应用实践技能

> 从"懂了"到"会写"的转换。概念讲完、边界试完，必须写一份能在项目里直接用的代码。

## 触发时机

- 学生学完一个概念和边界后，说"那具体怎么用？"
- 学生说"自己写一个试试"
- teach-concept 基础文件完成后

## 流程

```
teach-concept 完成基础文件（原子讲解）
    ↓
practice-apply 触发
    ↓ 基于基础文件写 _applied.py
    1. 设计一个可复用的函数/类
    2. 用真实场景的组合 SQL 验证
    3. 标注关键设计决策和取舍
    ↓
验收标准：函数能 import 到项目直接用
```

## 判断标准

学生学完这个文件，能把里面的函数直接 import 到项目里用，不需要改。如果不行，说明写的是流水账不是生产代码。

## 示例输出格式

`AXX_ConceptName/ConceptName_applied.py`，结构：

```python
"""一句话说明这个模块做什么"""
import ...

def reusable_function(...) -> dict:
    """文档说明参数和返回值"""
    ...

# 验证：真实场景 SQL
sql = "SELECT ..."
result = reusable_function(parse_sql(sql))
print(result)
```

## 关键原则

1. **暴露可复用接口** — 函数/类必须在文件顶层定义，不是写在 `if __name__` 里
2. **用真实 SQL 验证** — 用项目里可能出现的 SQL，不是教学玩具 SQL
3. **标记设计决策** — 注释说明为什么这么选（如"选手动取表而不是两遍遍历，因为性能差 1.8 倍"）
4. **可独立运行** — `python xxx_applied.py` 直接出结果
