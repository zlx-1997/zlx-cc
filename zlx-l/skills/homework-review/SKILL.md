---
name: homework-review
description: 作业审查技能——主 agent 对子 agent 的 homework.py 做严格代码审查，确保无静默错误、无猜值、无手写 AST 遍历，然后写行为测试 verifier.py。
triggers: ["审查作业", "批改", "review homework", "check homework"]
---

# Homework Review — 作业审查技能

> 主 agent 在 auto-study Step 2 子 agent 产出 homework 后调用此技能。
> 严格审查代码质量 + 写 behavior test（verifier.py），不依赖 verifier 的绿色勾勾。

## 和 verifier 的区别

| | verifier.py | homework-review |
|--|-----------|----------------|
| 测什么 | 外部行为（输入→输出契约） | 内部实现（代码质量、错误处理） |
| 谁写 | 主 agent（审查后写） | 主 agent（审查时读） |
| 自动化 | `run_verifiers.py` 自动跑 | 逐行手动审查 |
| 发现的问题 | 功能 bug | 静默吞错、猜值、重复造轮 |

## 审查流程

### Step 1：路径与全局状态

```python
# ❌ 禁止——硬编码层级
sys.path.insert(0, str(Path(__file__).resolve().parents[3]))

# ❌ 禁止——改工作目录
os.chdir("../../..")

# ✅ 正确——用项目标准方法
from src.common.find_anchor import find_anchor
PROJECT_ROOT = find_anchor(__file__, "CLAUDE.md")
```

- `sys.path.insert(0, ...)` → 用完 pop 了吗？没有就是污染全局 path
- 有 `os.chdir` 或 `Path.cwd` 依赖？→ 重构掉

### Step 2：AST 遍历方式

**任何手动遍历 pglast AST 的行为都是红色警告**。先问：能不能用 pglast Visitor 替代？

```python
# ❌ 禁止——手写 __slots__ 遍历
slots = getattr(type(node), '__slots__', {})
for attr in slot_names:
    val = getattr(node, attr)

# ❌ 禁止——手写 BFS 替代 Visitor
queue = [tree]
while queue:
    node = queue.pop(0)
    ...

# ✅ 正确——用 pglast Visitor
class MyVisitor(Visitor):
    def visit_ColumnRef(self, ancestors, node):
        ...
```

如果确实必须手动遍历（原因极少），子 agent 必须在 `reasoning.md` 中写明**为什么 Visitor 不够用**，否则打回重写。

### Step 3：异常处理

**`except Exception` 直接判定为红色**。

```python
# ❌ 禁止——静默吞所有异常
try:
    result = analyze(sql)
except Exception:
    pass  # 吞了之后没人知道错了

# ❌ 也不行——记到 error 字段不抛
try:
    ...
except Exception as e:
    result["error"] = str(e)  # 调用方可能不看 error 字段

# ✅ 正确——只捕获预期的异常
try:
    result = analyze(sql)
except (IndexError, KeyError):
    pass  # 边界情况已知
```

**做不来就说不会**。子 agent 不确定的等价类，必须写进 `reasoning.md`：

```
❓ 不确定：关联子查询的别名映射没做
  尝试了：用 Visitor + ancestors 检测外层表
  卡住了：子查询 WHERE 中的 ColumnRef 有别名 a，
          但外层 Scope 存的表名是 t1 不是 a
  猜测：可能需要 ScopeChain 的 resolve_ref 才能正确解析
```

不允许静默返回 None 或空字典让主 agent 误以为"功能实现了"。

### Step 4：verifier 必须测行为，不测实现

```python
# ❌ 脆弱——依赖内部 dict 结构（重构字段名就炸）
assert r["columns_by_context"]["top_level"][0] == "e.name"

# ❌ 脆弱——依赖类型或嵌套层级
assert isinstance(r["layers"][0], dict)
assert r["layers"][0]["sources"][0]["is_subquery"] == False

# ✅ 健壮——测输入→输出的行为
assert analyze_join_type("SELECT * FROM a LEFT JOIN b ON a.id = b.id").get("jointype") == "JOIN_LEFT"

# ✅ 健壮——测可观察的外部结果
result = resolve_in_chain("SELECT x FROM (SELECT a AS x FROM t) sub", "x")
assert result is not None  # 外部可观察，不管内部结构
```

判断标准：**如果 homework.py 重构了但功能不变，verifier 应该照样全绿。** 如果 verifier 红了，说明测的是实现不是行为。

### Step 5：审查清单（逐项打勾）

```
□ 路径：用了 find_anchor 还是手写？            ✅/❌
□ AST 遍历：用了 Visitor 还是手动？             ✅/❌
  若手动 → reasoning.md 解释了为什么吗？          ✅/❌
□ except Exception：有裸捕获吗？               ✅/❌
□ 猜值：有兜底返回值吗？                        ✅/❌
  有 → 子 agent 标注了"不确定"吗？               ✅/❌
□ verifier 测行为还是测实现？                    ✅/❌
□ reasoning.md 卡点 → 主 agent 逐条回答了？     ✅/❌
```

**任意 ❌ 必须修正后才能写 verifier.py。**
