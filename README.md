<div align="center">

# Skill Router

统一技能路由 v2.0 — 使用微调中文嵌入模型预筛选相关技能，在每次 LLM 调用前自动注入相关技能推荐

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License">
  <img src="https://img.shields.io/github/last-commit/weksbwrx62862/skill-router?color=blue" alt="Last Commit">
  <img src="https://img.shields.io/badge/Hermes-%3E%3D2.0.0-orange.svg" alt="Hermes">
  <img src="https://img.shields.io/badge/version-2.0.0-blue.svg" alt="Version">
</p>

</div>

core/pool 分层 + pre_llm_call 自动注入 + 反馈学习。

## 功能矩阵

| 能力 | 描述 | 状态 |
|------|------|------|
| 微调嵌入模型 | 中文 BGE 嵌入模型，302 条训练数据微调 | ✅ 已完成 |
| pre_llm_call 自动注入 | LLM 调用前自动预筛选并注入技能上下文 | ✅ 已完成 |
| 上下文注入 | 预筛选结果注入用户消息，不破坏系统提示缓存 | ✅ 已完成 |
| 预计算嵌入 | 首次加载预计算所有技能嵌入，后续检索 <1s | ✅ 已完成 |
| 反馈学习 | 用户反馈持续优化匹配精度 | ✅ 已完成 |
| 语义搜索工具 | skill_search 工具支持 Top-K 语义检索 | ✅ 已完成 |
| 模型微调脚本 | fine_tune_embedding.py 支持增量训练 | ✅ 已完成 |

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

## 技术栈

```
+------------------------+----------------------------------+
| 类别       | 技术                             |
+------------------------+----------------------------------+
| 语言       | Python 3.10+                     |
| 嵌入模型   | BGE (中文) + 微调                |
| 向量检索   | FAISS (CPU)                      |
| 配置管理   | PyYAML                           |
| 插件框架   | Hermes Agent >= 2.0.0            |
| 模型训练   | sentence-transformers            |
| 许可证     | MIT                              |
+------------------------+----------------------------------+
```

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

## 路线图

- [ ] **多语言嵌入支持**：扩展至英文 BGE 嵌入模型，支持中英双语技能检索
- [ ] **GPU 加速**：支持 FAISS-GPU，大规模技能池检索性能提升 10x+
- [ ] **增量索引**：技能变更时仅更新差异嵌入，无需全量重算
- [ ] **匹配可解释性**：输出相似度分数与匹配依据，便于调试与优化
- [ ] **技能分类体系**：自动聚类技能，支持按类别筛选与浏览
- [ ] **A/B 测试框架**：对比不同嵌入模型 / 参数的匹配效果

## 常见问题

**Q: 首次加载为什么需要 ~30s？**

A: 首次加载需要将微调嵌入模型读入内存，并预计算所有技能的嵌入向量。后续检索直接使用预计算结果，响应时间 <1s。

**Q: 如何添加新的技能到路由池？**

A: 将技能 YAML 文件放入技能池目录，重启 Hermes 或调用 `skill_search` 触发重新索引即可自动纳入。

**Q: 匹配准确率不够高怎么办？**

A: 使用 `skill_feedback` 工具提交反馈，积累训练数据后运行 `fine_tune_embedding.py` 重新微调模型。详见[持续改进](#持续改进)章节。

**Q: 可以不依赖 Hermes 独立使用吗？**

A: 可以。`skill_search` 工具可独立调用，但 `pre_llm_call` 自动注入功能需要 Hermes 插件框架支持。

**Q: 支持哪些嵌入模型？**

A: 当前使用中文 BGE 嵌入模型。可通过替换 `fine-tuned-model/` 目录下的模型文件切换到其他 sentence-transformers 兼容模型。

## Contributing

欢迎贡献！请遵循以下流程：

1. **Fork** 本仓库到你的 GitHub 账户
2. **Branch** 创建功能分支：`git checkout -b feat/your-feature`
3. **Commit** 提交变更，遵循 [Conventional Commits](https://www.conventionalcommits.org/) 规范
4. **PR** 提交 Pull Request 到 `main` 分支，填写完整的 PR 描述模板

提交前请确保：
- 代码通过 lint 检查
- 新功能包含必要的测试
- 不包含敏感信息或构建产物

## License

MIT

## Security

如发现安全漏洞，请**不要**通过 GitHub Issue 公开报告。请发送邮件至仓库维护者，我们将在确认后尽快修复并发布安全公告。

安全最佳实践：
- 不要将 API Key 或 Token 硬编码到配置文件中
- 使用环境变量管理敏感配置
- 定期更新依赖以修复已知漏洞

## 致谢

- [BGE Embedding](https://huggingface.co/BAAI/bge-large-zh) — 中文嵌入模型
- [sentence-transformers](https://www.sbert.net/) — 模型微调与推理框架
- [FAISS](https://github.com/facebookresearch/faiss) — 高效向量相似度检索
- [Hermes Agent](https://github.com/weksbwrx62862/hermes) — 插件框架支持

<div align="center">

**让每个 LLM 调用都拥有最合适的技能**

</div>