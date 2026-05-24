---
name: verify-hunch
description: 验证猜测技能——通过出题/树图对比/等价类抽查，验证学生对 AST 概念的理解程度。触发词：验证猜测、考题、考一下、出题、还原 SQL、预测树图、考试。
triggers: ["验证猜测", "考题", "考一下", "出题", "还原 SQL", "预测树图", "考试"]
---

## 模式切换

> 🎓 学习模式 — 使用此技能前自动切换到学习场景

```bash
python "C:/Users/zlx_1/.claude/plugins/cache/zlx/common/toggle-mode.py" learn
```

继续前先运行以上命令切换为学习模式（关闭 tdd 门禁等开发钩子）。

# Verify Hunch — 验证猜测技能

> 验证学生是否真的理解了，还是只是听懂了。通过三种考题形式检验理解程度。

## 触发时机

- 学生说"考考我"、"出个题"
- 学完一个概念后想确认理解程度
- INDEX.md 里某个知识点打完勾后

## 三种考题形式

### ① 从 AST 还原 SQL

我给 AST 树图，学生不看原 SQL，尝试还原。

```
输入（树图）:
SelectStmt
├── targetList: ColumnRef(cty_cd), FuncCall(min)
├── fromClause: RangeVar(ref_city)
└── whereClause: A_Expr(<>) → A_Const('CN')

学生还原:
SELECT cty_cd, MIN(...) FROM ref_city WHERE ... <> 'CN'
```

验证方式：用 `show(sql_text=学生写的SQL)` 看树图是否匹配。

### ② 预测树图

我给 SQL，学生预测树图结构，然后跑 `pglast_debug.py -m 5` 验证。

```
输入（SQL）:
SELECT d.name FROM departments d WHERE d.id > 1

学生预测:
SelectStmt
  ├── targetList: ...
  ├── fromClause: ...
  └── whereClause: ...

验证:
python scripts/pglast_debug.py -m 5 "SELECT d.name FROM departments d WHERE d.id > 1"
```

### ③ 等价类抽查

给一个概念，让学生列出正常/边界/陷阱：

```
概念：ColumnRef.fields
→ 正常: name, d.name
→ 边界: pub.d.name, *, d.*
→ 陷阱: A_Star 没 .sval，需要 hasattr 过滤
```

考点来源：`scripts/tutorial/INDEX.md` 里的 checkmark 知识点。

## 工具

| 工具 | 用途 |
|------|------|
| `blackboard_student_run.py` | 学生验证 SQL 树图（三种考试都能用） |
| `pglast_debug.py -m 5` | 出题时生成树图 / 验证预测 |
| `00notes/` | 记录考题和常见错误 |

## 评分标准

- ✅ 全部正确：理解到位，可进下一概念
- ⚠️ 部分正确：查漏补缺，回 teach-concept 补对应的等价类
- ❌ 大部分错误：回到 teach-concept 重讲，换一个等价类编排顺序
