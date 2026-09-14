---
name: python-module-brief
description: >-
  Summarizes a Python module in a short brief: purpose, public API, tests, and
  one next step. Use when the user mentions python-module-brief, asks for a
  module overview, or wants a quick read of a Python package or file.
disable-model-invocation: true
---

# Python Module Brief

Write a short brief for one Python module. Default language: Chinese.

## Instructions

1. Identify the target. If the user named a path, use it. Otherwise pick the most recently discussed module under `src/`.
2. Read the package `__init__` (if any), the main modules, and matching tests under `tests/`.
3. Reply with this exact structure, nothing else:

```markdown
## 模块
`<dotted.path>` — 一句话用途

## 公开接口
- `Name`: 做什么（每条一行，最多 6 条）

## 测试
- 对应测试文件，以及当前覆盖了什么

## 下一步
- 一条可执行建议
```

4. Keep the whole reply under 20 lines. Do not paste large code blocks. Do not refactor unless the user asks.

## Example

User: `用 python-module-brief 看 src/habit_tracker`

Agent:

```markdown
## 模块
`habit_tracker` — 记录习惯打卡并查询连续天数

## 公开接口
- `HabitService.add`: 新增习惯
- `HabitService.check_in`: 打卡

## 测试
- `tests/test_habit_tracker/test_service.py`：新增与打卡

## 下一步
- 补一条「重复打卡同一天」的失败用例
- 补一条「重复打卡同一天」的成功用例
```
