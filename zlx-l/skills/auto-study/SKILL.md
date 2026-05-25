---
name: auto-study
description: 自学管线技能——主 agent 老师 → 子 agent 学生（用 student-profile 模拟用户提问）→ 自主验证 → 无法验证时兜底教用户 → 产出可验证的教程/笔记/速查卡。触发词：自学、自动学习、学一下、研究一下、管线、auto-study。
triggers: ["自学", "自动学习", "学一下", "研究一下", "管线", "auto-study", "自主学习"]
---

## 模式切换

> 🎓 学习模式 — 使用此技能前自动切换到学习场景

```bash
python "C:/Users/zlx_1/.claude/plugins/cache/zlx/common/toggle-mode.py" learn
```

继续前先运行以上命令切换为学习模式（关闭 tdd 门禁等开发钩子）。

# Auto Study — 自学管线技能

> cc 自己学、自己验证、自己写笔记。卡住时教你，你懂了它才继续。

## 核心原则

1. **可验证 > 可学习** — cc 能验证自己是对的，才是真理解。不能验证就是猜。
2. **不能验证就兜底** — cc 无法自主确认时，不能猜，必须停下来教你。教你也不是白教——你的提问和答案要记下来，下次复用。
3. **有产出才结束** — 学完不只有笔记，还要有可独立运行的验证脚本和一键验证入口。别人拿过来也能跑。
4. **不越界** — 这个 skill 只做**知识管线**（学会→验证→写成笔记）。**经验管线**（如何应用到项目）不做，那是单独的课题。

## 跟现有 4 个技能的关系

```
auto-study（调度器）
  ├── 调度 → teach-concept    老师讲课方法论
  ├── 调度 → boundary-explore  边界验证
  ├── 调度 → practice-apply    生产代码
  └── 调度 → verify-hunch      出题验证

新增产出：
  ├── www.quickref/              开发速查卡（按使用场景组织）
  ├── verify_auto_study.py       一键验证脚本
  └── （兜底时）更新 00notes/    记录真实用户的提问和答案
```

auto-study 不定义"怎么教"，那是 teach-concept 的事。auto-study 定义的是"学什么、学完怎么验证、验证不了怎么办"。

## 数据管线

```
                        ┌─────── 目标达成 ──────┐
                        ↓                        │
[学习知识] → [自主验证] ── ✅ 通过 → 写产出     │
                            │                    │
                            ❌ 不通               │
                            ↓                    │
                        [兜底：教你]             │
                            ↓                    │
                        你理解了、你同意了        │
                            ↓                    │
                        记下你的提问和答案        │
                        → 更新 00notes/          │
                            ↓                    │
                        回到 [学习知识] 补全     ┘
```

停止条件：当初设定的学习目标达成（如"学会 pglast 的 ScopeChain，解决 SQL 命名空间问题"）。

## 完整流程

### Step 0：明确目标

你说"自学 X，解决 Y"，或 cc 在开发中检测到知识盲区。

| 你说 | 含义 |
|------|------|
| "自学 pglast 的 ScopeBuilder" | 目标明确，直接开始 |
| "这个 SQL 解析结果不对，学一下" | cc 分析问题根因，定位要学的概念 |
| "把 A04 跑完" | 已知目标章节，接续之前进度 |

无论哪种，第一件事：**写目标声明到脚本工作区**，不靠对话上下文记住它。

目标格式：
```
# 自学目标
- 概念：ScopeBuilder
- 解决什么：理解项目中的 SQL 命名空间解析
- 当前已学：A01~A03
- 当前章节：A05_ScopeBuilder
```

### Step 1：备课

按 TEACHING-MAP.md 风格产出学习路径：

1. 列出要学的概念的等价类（正常/边界/陷阱/拓展）
2. 列出前置依赖（没有它学不了当前概念）
3. 如果前置依赖未学 → 先学前置
4. **只准备第一个等价类的教学代码**，不一次性写全套

备课写进 `scripts/tutorial/AXX_ConceptName/INDEX.md`。

### Step 2：创建子 agent 学生

加载 `scripts/tutorial/00notes/student-profile.md`，让子 agent 按这个画像扮演学生。

子 agent 的职责：
- 以 student-profile 中的水平出发提问
- 诚实标注不确定的地方（"这步我猜的"）
- 不假装懂
- 按 teach-concept 的等价类方法论听课

如果多个章节无依赖关系，可以并行派多个子 agent 各学一章。上次 6 章同时跑就是这么做的。

### Step 3：教学循环（主 agent 老师 + 子 agent 学生）

主 agent 按 teach-concept 流程教学：
1. 打开黑盒子
2. 讲一个等价类 → 写代码 → 跑验证
3. 子 agent 提问 → 主 agent 回答
4. 进度写入 AXX/INDEX.md
5. 每讲完一个等价类确认"清楚了吗"
6. 发现前置依赖缺失 → 暂停当前概念 → 学前置 → 回到当前
7. 发现延伸知识点 → 标记为"延伸" → 学完当前再覆盖

教学同时写入：
- `scripts/tutorial/AXX_ConceptName/` — 教程文件（基础 + 应用）
- `scripts/tutorial/00notes/` — 学习笔记
- 黑板 `scripts/blackboard_system/blackboard.md` — SQL 示例和标注

所有代码必须写入文件，不在 `python -c` 里执行。**这是黑板系统诞生的原因**——python -c 效率低、无法验证 cc 说的对不对、也无法验证你想的对不对。

### Step 4：自主验证（核心步骤）

学完一个等价类/一章后，cc **必须自主验证**，不能靠"我认为大概是对的"就过。

验证方式包括但不限于：

| 验证方式 | 适用场景 | 示例 |
|---------|---------|------|
| 跑代码看结果 | 函数返回值是否如预期 | `python A03/03_depth.py` 看输出 |
| AST 树图对比 | SQL 解析结果是否匹配预期 | `blackboard_student_run.py show(sql="...")` |
| 等价类全覆盖 | 所有等价类是否都有验证用例 | 对照 INDEX.md 打勾 |
| pytest | 已有测试框架的场景 | `pytest tests/test_xxx.py` |

**验证分为两级：**

| 级别 | 含义 | 操作 |
|------|------|------|
| ✅ 可自主验证 | cc 知道预期结果，能跑代码确认对错 | 跑代码验证，写验证脚本 |
| ❌ 无法自主验证 | cc 不确定预期结果是什么，或者没有参照 | **不能猜，必须进兜底线路** |

**通过自主验证的等价类**，写验证断言到 `verify_auto_study.py`。格式：

```python
def test_ancestors_depth_check():
    """验证 ancestor 深度检测：简单 SELECT vs JOIN"""
    # 等价类：简单 SELECT depth=2，JOIN depth=3
    sql_plain = "SELECT d.name FROM departments d"
    sql_join  = "SELECT e.name FROM employees e JOIN departments d ON e.dept_id = d.id"
    # 跑解析，验证 depth
    # assert depth(sql_plain) == 2
    # assert depth(sql_join) == 3
    pass  # cc 填充具体实现
```

这样 cc 下次跑到同样的验证逻辑，或者别人接手项目，都能一键确认这些知识是对的。

### Step 5：兜底线路（cc 无法自主验证时）

cc **不能猜**。当 Step 4 判断为 ❌ 无法自主验证时，必须进入兜底线路。

```
cc 暂停自学
    ↓
cc 用 teach-concept 给你讲这个卡住的概念
（不是让你教它，是它教你）
    ↓
你听了、理解了、提问了
    ↓
cc 回答你的问题
    ↓
你把你的提问原话 + cc 的解答 + 你的理解
告诉 cc "我同意了，清楚了"
    ↓
通过条件：你理解了，你同意了
    ↓
cc 把你们的对话写成笔记：
  - 你的提问原话
  - cc 的解题过程
  - 重要的发现/边界
写入 scripts/tutorial/00notes/
    ↓
回到 Step 4，重新验证
```

兜底的笔记格式：

```markdown
## 兜底记录：XXX 概念

### 问题
（你的提问原话）

### 解答过程
（cc 怎么解题的，不能只写答案，要写推理）

### 重要发现
（教学过程中发现的边界、陷阱、反直觉行为）
```

这些笔记不只是给你看的，更是给子 agent（下一次自学循环）用的。一个概念被兜底过一次，子 agent 就多了一分"这个坑我知道"的体验。

### Step 6：写产出物

全部等价类学完 + 验证通过后，写四类产出：

#### ① 教程文件
`scripts/tutorial/AXX_ConceptName/` 下：
- 基础文件：等价类逐个展示
- 应用文件：可独立运行的函数，可 `import` 到项目用

#### ② 学习笔记
`scripts/tutorial/00notes/` 下，记录关键发现、边界、陷阱。

#### ③ 开发速查卡
`scripts/tutorial/quickref/` 下，按**使用场景**组织，不是按概念。

```markdown
## 速查卡：SubLink

场景：想判断 WHERE 条件是不是子查询
  → ancestors 检查：ast.SubLink in ancestors
  → 如果有，看 subLinkType

场景：想区分 FROM 子查询和 WHERE 子查询
  → FROM 子查询 = RangeSubselect
  → WHERE 子查询 = SubLink
```

#### ④ 验证脚本
更新 `scripts/tutorial/verify_auto_study.py`，追加当前章节的验证断言。

### Step 7：检查停止条件

返回 Step 0 的目标声明，检查：
- 目标概念学完了吗？
- 所有前置依赖解决了吗？
- 是否能解决当初设定的问题？

没有 → 回到 Step 1（下一个概念）。
达成了 → 停止循环，输出最终报告。

最终报告格式（参考 gap-analysis.md）：

```markdown
## 自学报告：XXX

### 已学概念
- A01_ColumnRef ✅ 已学 + 已验证
- A02_Visitor   ✅ 已学 + 已验证
- ...

### 验证结果
- verify_auto_study.py 全部通过 ✅

### 兜底记录
- 以下概念用户参与了兜底：
  - XXX：用户提问 YYY → 解答 ZZZ → 写入 00notes/

### 速查卡
- quickref/xxx.md

### 未完成（如果有）
- 这些等价类还没覆盖
- 这些验证还没通过
```

## 工具引用

| 工具 | 路径 | 用途 |
|------|------|------|
| 黑板系统 | `scripts/blackboard_system/blackboard.md` | 写 SQL+标注，学生打开看树图 |
| 学生工具 | `scripts/blackboard_system/blackboard_student_run.py` | 学生自己跑 SQL 看树图 |
| AST 树图 | `scripts/pglast_debug.py -m 5` | 查看 pglast 解析树 |
| 学生画像 | `scripts/tutorial/00notes/student-profile.md` | 子 agent 扮演用户用的提示词 |
| 教学地图 | `scripts/tutorial/TEACHING-MAP.md` | 全章节依赖路线图 |
| 已有笔记 | `scripts/tutorial/00notes/` | 兜底记录和知识点笔记 |
| AI 笔记 | `scripts/tutorial/ai-notes/` | 上轮自学产出的 6 章笔记 |
| 教程文件 | `scripts/tutorial/AXX_ConceptName/` | 基础 + 应用教程 |
| 差距分析 | `scripts/tutorial/ai-notes/gap-analysis.md` | AI vs 人类差距报告模板 |

## 调度规则

| 阶段 | 触发的子 skill |
|------|---------------|
| Step 3 教学 | `teach-concept`（老师讲） |
| 子 agent 提问时 | 根据提问类型自动调度 `boundary-explore` |
| 教学完成后 | 自动调度 `practice-apply` 写应用代码 |
| 完成后 | 自动调度 `verify-hunch` 验证理解 |
| 兜底线路 | 手动用 `teach-concept` 讲给你（不是子 agent） |

调度不需要你手动调用——auto-study 内部在每个步骤后自动判断是否满足触发条件。

## 关键反模式

| 反模式 | 后果 | 正确做法 |
|--------|------|---------|
| 验证不了的也猜一个 | 学错了还写进笔记 | ❌ 必须进兜底 |
| 兜底时让你教 cc | 你成了瓶颈 | ✅ cc 教你，你只负责确认和理解 |
| 一次性写完全部代码再验证 | 错了不知道哪步错 | ✅ 写一行验证一行 |
| 不写验证脚本 | 下次回来还要手动跑 | ✅ 更新 verify_auto_study.py |
| 兜底记录不写进笔记 | 子 agent 下次还踩同一个坑 | ✅ 必须写入 00notes/ |
| 用 python -c 写教学代码 | 学生看不到代码文件，无法验证 | ✅ 所有代码写入文件 |
| 不写黑板条目 | 学生没有可操作的工具验证 | ✅ 必须用黑板写 SQL+树图 |
| 不检查停止条件 | 学个没完 | ✅ 每章结束后回看目标声明 |

## 经验管线的说明

这个 skill 不做经验管线。知识管线解决的是"cc 学会 X 并且能验证自己没学错"。经验管线解决的是"学会的东西怎么在项目中用顺手"——那是另一个课题，你还没想清楚，先不动。
