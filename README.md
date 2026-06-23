# 👁️ bailian-eyes

给没有视觉能力的 LLM（如 DeepSeek）装上眼睛。

通过 [bailian-cli](https://github.com/modelstudioai/cli) 调用 Qwen-VL 视觉模型，将图片转为文字描述，再交给你的主力模型推理。

## 安装

```bash
# 1. 安装 bailian-cli
npm install -g bailian-cli

# 2. 配置 API Key
bl auth login --api-key sk-你的百炼key

# 3. 安装 skill
npx skills add emanon312/bailian-eyes -g
```

## 工作原理

```
你的图片 → bl vision describe (Qwen-VL) → 文字描述 → DeepSeek/其他文本模型 → 回答
```

## 触发条件

当使用没有视觉能力的模型（如 deepseek）且任务涉及图片时，skill 自动激活，强制走视觉委托流程。

## 支持的场景

- 截图理解（UI 分析、报错截图）
- 图表解读（学习曲线、数据可视化）
- 照片描述
- 文档扫描件
- 视频理解（通过 `bl omni`）

## 许可证

MIT
