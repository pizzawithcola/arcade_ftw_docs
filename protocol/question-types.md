# 题型契约

题型是**可插拔的处理器**：新增题型 = 加一个模块 + 注册一行，主线流程零改动。

## 接口

定义：`arcade_ftw_be/app/question_types/base.py` 的 `QuestionType`（`Protocol`）。

| 成员 | 签名 | 职责 |
|---|---|---|
| `type_key` | `str` | 注册键（注册表用）|
| `render` | `(question) -> dict` | 发给前端的展示数据 |
| `accept` | `(session, answer) -> bool` | 是否收下这个答案 |
| `score` | `(answers) -> dict` | 计分结果，**写成纯函数**便于单测 |
| `timeout_behavior` | `() -> str` | 超时行为：`auto_submit` / `discard` |

## 注册表

`arcade_ftw_be/app/question_types/__init__.py`：

```python
QUESTION_TYPES: dict[str, QuestionType] = {
    ChoiceType.type_key: ChoiceType(),
}
```

## 现状与规划

| type_key | 状态 |
|---|---|
| `choice` | ✅ 已实现（示例，`choice.py`）|
| `trust_doubt` | ⬜ 待实现（信任 / 怀疑）|
| `interactive` | ⬜ 待实现（互动题）|
