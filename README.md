# comfy-flux-wan-skill

ComfyUI + Flux 图片生成 + Wan2.1 视频生成工作流 skill。

## 📖 项目简介

这是一个完整的 ComfyUI 技能包，整合了：
- **Flux** 高效能文生图模型
- **Wan2.1** 先进视频生成能力
- 预构建的工作流配置

支持快速生成高质量图片和视频内容。

## 📁 文件结构

```
comfy-flux-wan-skill/
├── README.md                 # 本文件
├── SKILL.md                  # 完整执行技能说明
└── workflows/                # ComfyUI 工作流文件
    └── (官方示例工作流 JSON)
```

### 主要文件说明

| 文件 | 说明 |
|-----|------|
| `SKILL.md` | 技能完整使用指南和执行流程 |
| `workflows/` | 官方 ComfyUI 工作流 JSON 配置文件（来源：[comfyanonymous/ComfyUI_examples](https://github.com/comfyanonymous/ComfyUI_examples)） |

## 🚀 快速开始

### 前置要求

- ComfyUI 环境已安装
- Python 3.8+
- CUDA/GPU 支持（推荐用于加速）

### 安装步骤

1. **克隆或下载此项目**
   ```bash
   git clone https://github.com/linkaidadie-hash/comfy-flux-wan-skill.git
   cd comfy-flux-wan-skill
   ```

2. **查看完整指南**
   
   详细的使用说明请参考 [`SKILL.md`](./SKILL.md)

## 💡 功能特性

- ✅ **Flux 图片生成** - 快速生成高质量图片
- ✅ **Wan2.1 视频生成** - 创建专业视频内容
- ✅ **预置工作流** - 即插即用的配置
- ✅ **灵活扩展** - 支持自定义工作流

## 📝 使用方法

完整的使用流程和参数说明见 [`SKILL.md`](./SKILL.md)

### 基础工作流

1. 加载相应的工作流 JSON 文件
2. 配置输入参数
3. 执行生成任务
4. 导出结果

## 🔗 相关资源

- [ComfyUI 官方文档](https://github.com/comfyanonymous/ComfyUI)
- [ComfyUI 示例工作流](https://github.com/comfyanonymous/ComfyUI_examples)
- [Flux 模型文档](https://huggingface.co/black-forest-labs)

## 📄 许可证

请查看项目中的许可证文件（如有）

## 🤝 贡献

欢迎提交 Issue 或 Pull Request！

## 📧 反馈与支持

如有问题，请在 GitHub Issues 中反馈。

---

**最后更新**: 2026-05-18
