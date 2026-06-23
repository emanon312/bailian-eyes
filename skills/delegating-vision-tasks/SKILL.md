---
name: delegating-vision-tasks
description: Use when the current model has no vision capability and images, screenshots, diagrams, photos, videos, or any visual content is part of the task — the user pasted an image, referenced a screenshot, or the task requires understanding what something looks like
---

# 视觉任务委托

## 概述

当前模型无视觉能力时，用 `bl vision describe` 将图片转为文字描述，再基于文字推理。本质是把视觉编码+理解外包给 Qwen-VL，推理留在本地模型。

## 何时使用

```
收到任务 → 涉及图片/视频/视觉内容？
    ├─ 否 → 正常处理
    └─ 是 → 当前模型有视觉能力？
              ├─ 是 → 直接处理
              └─ 否 → 委托 bl vision describe
```

**触发信号：**
- 用户消息中包含图片附件或截图路径
- 用户要求"看图"、"分析这张图"、"描述截图内容"
- 任务需要理解 UI 界面、图表、照片、手绘图、文档扫描件
- 需要对比两张图的差异

**不使用的情况：**
- 纯文本任务
- 当前模型本身具备视觉能力（如 Claude Sonnet/Opus）——直接用模型视觉，不要绕路
- 需要精确像素级分析（OCR 坐标定位、像素颜色采样、精确尺寸测量）——如实告知限制

## 警惕信号 — STOP 并检查

- "我没有视觉能力所以我做不到" → **错**，你有 `bl vision describe`
- "用户可能想让我换个有视觉的模型" → **错**，先用 `bl` 处理，不要猜测用户意图
- "这图太复杂了文字描述不够" → 先试试，描述不够再加 `--prompt` 定向追问
- "我可以直接读图片文件" → **错**，deepseek 看不懂二进制图片数据

## 快速参考

| 场景 | 命令 |
|------|------|
| 单张图片/截图 | `bl vision describe --image "<path>"` |
| 需要定向描述 | `bl vision describe --image "<path>" --prompt "重点关注..."` |
| 多张图片 | `bl vision describe --image "<path1>" --image "<path2>"` |
| 视频理解 | `bl omni --video "<path>" --text-only` |
| 检查认证 | `bl auth status` |
| 切换 API Key | `bl auth login --api-key sk-...` |

## 铁律

**遇到视觉内容 + 当前模型无视觉能力 → 必须委托。** 不允许只说"我没有视觉能力"然后停下。`bl vision describe` 就是为此存在的。

## 标准流程

### 第〇步：确认文件存在

在调用 `bl` 之前，先用 `ls` 或 Read 确认用户给的文件路径真实存在。如果不存在，立即告知用户并要求提供正确路径——不要调用 `bl`。

### 第一步：准备工作

```bash
# 确认 CLI 可用
bl --version

# 确认认证有效
bl auth status
```

如果 `bl` 不可用：提示用户 `npm install -g bailian-cli`。
如果未认证：提示用户提供 API Key，运行 `bl auth login --api-key sk-...`。
如果配额耗尽：提示用户换 Key 或在百炼控制台充值。

### 第二步：获取描述

```bash
bl vision describe --image "<本地路径>" --prompt "请详细描述图片内容"
```

**关键技巧：**
- `--prompt` 用中文描述你关注的重点，如"重点看右侧图表的数据趋势"
- 本地路径直接传入，无需上传到图床
- 输出是 JSON，提取 `choices[0].message.content` 即可得到文字描述

### 第三步：基于描述推理

拿到文字描述后，用当前模型完成推理任务——回答问题、分析数据、给出建议等。

## 输出提取

`bl vision describe` 返回 JSON 格式，有效内容在：

```
choices[0].message.content   → 文字描述
usage                         → token 消耗信息
model                         → 实际使用的视觉模型
```

描述通常是详细的自然语言文本，可以直接作为推理的上下文。

## 边界情况处理

### 配额耗尽

```
错误: AllocationQuota.FreeTierOnly
```

→ 告知用户免费额度用完，需换 Key 或充值。用户给新 Key 则运行 `bl auth login --api-key <新Key>`。

### 图片不存在

```
错误: 文件路径不存在
```

→ 确认用户给的路径是否正确，用 `ls` 检查文件是否存在。

### 描述不够详细

→ 用 `--prompt` 追加定向要求，如"请更详细地描述右上角的数值和颜色"。

## 常见错误

| 错误 | 正确做法 |
|------|---------|
| 直接用 deepseek 处理图片 | 无视觉能力，必须先委托 `bl vision describe` |
| 只描述不推理 | 描述是中间产物，最终要对用户的问题给出答案 |
| 忽略 JSON 外层结构 | 描述内容在 `choices[0].message.content`，不是整个 JSON |
| 让用户手动上传图片 | 本地路径直传即可 |
| 用 `bl omni` 处理静态图片 | 图片用 `bl vision describe`，`omni` 用于视频/音频 |

## 示例

用户给你一张 UI 截图问"这个按钮为什么是灰色的？"

```bash
# 1. 获取描述
bl vision describe --image "./screenshot.png" \
  --prompt "请详细描述这个UI界面，重点是按钮的状态、颜色和周围上下文"

# 2. 拿到描述后推理
# 描述: "界面中有一个'提交'按钮，颜色为浅灰色(#CCC)，
#        上方表单中必填字段'邮箱'为空..."
```

→ 回答："提交按钮是灰色的因为必填字段'邮箱'未填写，这是禁用状态的视觉反馈。"
