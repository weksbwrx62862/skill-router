# Skill Router

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License">
  <img src="https://img.shields.io/badge/Hermes-%3E%3D2.0.0-orange.svg" alt="Hermes">
  <img src="https://img.shields.io/badge/version-2.0.0-blue.svg" alt="Version">
</p>

统一技能路由 v2.0 — 使用微调中文嵌入模型预筛选相关技能，在每次 LLM 调用前自动注入相关技能推荐。core/pool 分层 + pre_llm_call 自动注入 + 反馈学习。

## 核心能力

- **微调嵌入模型**：使用中文 BGE 嵌入模型，在 302 条训练数据上微调
- **pre_llm_call 自动注入**：在 LLM 调用前自动预筛选并注入相关技能上下文
- **上下文注入**：将预筛选结果注入用户消息，不破坏系统提示缓存
- **预计算嵌入**：首次加载时预计算所有技能嵌入，后续检索 <1s
- **反馈学习**：用户反馈持续优化匹配精度

## 安装

### 前置条件

- Python 3.10+
- [Hermes Agent](https://github.com/weksbwrx62862/hermes) >= 2.0.0

### 从源码安装

```bash
git clone https://github.com/weksbwrx62862/skill-router.git
cd skill-router
pip install -e .
```

### 依赖

```bash
pip install sentence-transformers faiss-cpu pyyaml
```

## 使用

### Hermes 插件模式

```yaml
# hermes_config.yaml
plugins:
  - name: skill-router
    path: ./skill-router
    config:
      model_path: ./fine-tuned-model
      db_path: ./skill_index.db
      top_k: 5
```

### 自动注入（推荐）

启用插件后，每次用户消息都会自动触发预筛选：

```
用户: 帮我调试 Python 脚本

[Skill Router: Related skills found]
  - python-debugpy: Debug Python: pdb REPL + debugpy remote (DAP)
  - jupyter-live-kernel: Iterative Python via live Jupyter kernel
  - codebase-inspection: Inspect codebases w/ pygount
[Use skill_view(name) to load a skill]
```

### 工具调用

```json
{
  "name": "skill_search",
  "arguments": {
    "query": "帮我调试 Python 脚本",
    "top_k": 5
  }
}
```

## 架构

```
用户消息 → pre_llm_call钩子 → skill_router.search_skills()
                                  ↓
                       预计算嵌入 + 查询嵌入
                                  ↓
                        余弦相似度 Top-K
                                  ↓
                     注入上下文到用户消息 → LLM
```

## 性能

- **首次加载**：~30s（加载模型 + 预计算嵌入）
- **后续检索**：<1s（使用预计算嵌入）
- **准确率**：Top-1: 59.3%，Top-3: 78.1%（302 条测试集）

## 提供的工具

| 工具 | 功能 |
|------|------|
| `skill_search` | 语义搜索最相关技能 |
| `skill_feedback` | 提交搜索结果反馈 |

## 提供的钩子

| 钩子 | 说明 |
|------|------|
| `pre_llm_call` | LLM 调用前自动注入技能上下文 |

## 项目结构

```
skill-router/
├── plugin.yaml              # 插件声明
├── __init__.py              # 主入口 + 路由引擎
├── README.md                # 本文档
├── IMPROVEMENT_GUIDE.md     # 持续改进指南
├── fine-tuned-model/        # 微调嵌入模型
│   ├── config.json
│   ├── model.safetensors
│   └── tokenizer.json
└── scripts/
    └── fine_tune_embedding.py  # 模型微调脚本
```

## 持续改进

### 收集训练数据

当查询未命中正确技能时，将查询加入训练数据：

```python
{
  "query": "用户查询",
  "positive": "正确技能名",
  "negatives": ["相似但错误的技能1", "相似但错误的技能2"]
}
```

### 重新微调模型

```bash
cd skill-router
python scripts/fine_tune_embedding.py --train
```

## 开发

```bash
git clone https://github.com/weksbwrx62862/skill-router.git
cd skill-router
pip install -e .
# 通过 Hermes 运行时测试
```

## License

MIT