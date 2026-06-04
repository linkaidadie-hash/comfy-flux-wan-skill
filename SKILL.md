---
name: comfy-flux-wan
description: |
  ComfyUI + Flux 图片生成 + Wan2.1 视频生成工作流执行。进入 GPU 后：
  1. 获取官方 workflow JSON（禁止自己设计）
  2. 下载 workflow 对应模型
  3. 修正 workflow 内模型路径
  4. 通过 ComfyUI 本地接口运行
  5. 验证 output 目录出现真实 PNG（FLUX）或 MP4（Wan）

  触发：用户说"进GPU"、"连服务器"、"flux生成"、"wan视频"、"run workflow"、"下载模型"
  禁止触发：本地文件操作、飞书文档、PDF等无关任务。
  skill_id 名称约定：profile里实际目录名是 `comfy-flux-wan-skill`（带-s后缀，不可改），
  调用时必须用 `skill_view(skill_name='comfy-flux-wan-skill')`，
  不要用 `comfy-flux-wan`（会导致 Skill not found，6/3 14:44 已踩过此坑）。
---

# ComfyUI + Flux/Wan 执行技能

> **本文件 GitHub 源（通用版）**：`https://github.com/linkaidadie-hash/comfy-flux-wan-skill`
> **路径假设**：`/root/ai/ComfyUI/`
> **本机环境**：autodl 容器，**路径不一致，详见下方"autodl 环境适配"小节**。适配规则是只读 patch，不修改 GitHub 源，方便 `git pull` 同步。

## autodl 环境适配（重要，worker 必读）

> 本节是 **xiaoyimin-pipeline / openclaw-xiaodai** 容器的环境适配，**与 GitHub 通用版路径不同**。worker 在执行任何命令前必须**先运行"路径探测"**，**不要直接照搬 GitHub 源里的 `/root/ai/...` 路径**。

### 路径映射表

| GitHub 源路径 | 本机实际路径 | 说明 |
|---|---|---|
| `/root/ai/ComfyUI/` | `/root/autodl-tmp/ComfyUIWanvid/` | ComfyUI 主目录 |
| `/root/ai/workflows/` | `/root/autodl-tmp/ComfyUIWanvid/workflows/` | workflow JSON |
| `/root/ai/ComfyUI/models/{checkpoints,vae,...}` | `/root/autodl-tmp/ComfyUIWanvid/models/{checkpoints,vae,...}` | 模型目录 |
| `/root/ai/ComfyUI/output` | `/root/autodl-tmp/ComfyUIWanvid/output` | 产物输出 |
| `/root/ai/ComfyUI/custom_nodes/` | `/root/autodl-tmp/ComfyUIWanvid/custom_nodes/` | 自定义节点 |
| `/root/ai/comfyui.log` | `/root/autodl-tmp/ComfyUIWanvid/comfyui.log` | 日志 |
| 磁盘空间查询 `df -h /root/ai/` | 改为 `df -h /root/autodl-tmp/` | 数据盘100G，**不要写到 `/root` overlay盘** |
| 视频工单产物目录 | `/root/autodl-tmp/wuhai_story/output/` | 业务产物 |

### 路径探测脚本（worker 进门必跑）

```bash
# 真实路径探测（以本机为准）
COMFY_DIR=$(ls -d /root/autodl-tmp/ComfyUIWanvid /root/ai/ComfyUI 2>/dev/null | head -1)
if [ -z "$COMFY_DIR" ]; then
    echo "[FATAL] ComfyUI 目录不存在，本机预期 /root/autodl-tmp/ComfyUIWanvid/"
    echo "        拉仓库并参考 README.md 初始化"
    exit 1
fi
WF_DIR="$COMFY_DIR/workflows"
LOG="$COMFY_DIR/comfyui.log"
echo "COMFY_DIR=$COMFY_DIR"
echo "WF_DIR=$WF_DIR"
echo "LOG=$LOG"
```

### 业务集成点

- **xiaoyimin 工单派发**：主控 `xiaoyimin-pipeline/` 目录下所有 runner 引用本 skill 时，必须用本节路径映射替换 `/root/ai/...`。
- **pilot 闸门**：`/root/.hermes/workflows/xiaoyimin-pipeline/pilot_gate.sh`，字段名 `pilot_required`（不是 `pilot_run_required`，已于 2026-06-04 修复）。
- **GPU 标记**：`/root/.hermes/state/gpu_open.flag` 不存在 = GPU 未开，所有 video 工单 `wan_video_generator.py --pilot` 直接 reject。

## 核心理念

**workflow = 磁盘上真实 `.json` 文件。**
**没有 workflow JSON = 禁止开始任务。**
禁止用文字描述节点结构代替 JSON 文件。

## 阶段负一：环境依赖检查（进门后必须先执行）

> 在做任何操作之前，先系统性地验证环境是否满足要求。检查不通过则停止任务，不允许强行推进。

### 环境检查清单（全部通过才算合格）

```bash
# ═══ 1. GPU 检查 ═══
nvidia-smi --query-gpu=name,memory.total,memory.free,driver_version --format=csv,noheader
# 记录显存：<16GB / 24GB / 40GB / 48GB / 80GB -> 决定下载哪个模型版本

# ═══ 2. CUDA / cuDNN 版本 ═══
python3 -c "import torch; print('CUDA:', torch.version.cuda, '| cuDNN:', torch.backends.cudnn.version(), '| torch:', torch.__version__)"
# 要求：CUDA ≥ 11.8, torch ≥ 2.0（Flux/Wan 必要依赖）

# ═══ 3. Python 版本 ═══
python3 --version
# 要求：Python ≥ 3.8

# ═══ 4. ComfyUI 版本 ═══
curl -s http://127.0.0.1:8188/system_stats | python3 -c "import sys,json; d=json.load(sys.stdin); print('ComfyUI:', d.get('comfyui_version','unknown'), '| Device:', d['devices'][0]['name'], '| VRAM free:', round(d['devices'][0]['vram_free']/1e9,1), 'GB')"
# 记录版本，Flux/Wan 节点可能要求最低版本

# ═══ 5. 自定义节点目录存在性 ═══
echo "=== 自定义节点检查 ==="
ls -la /root/ai/ComfyUI/custom_nodes/ 2>/dev/null | head -20

# 检查必备自定义节点
REQUIRED_NODES=(
    "ComfyUI-WanVideoWrapper"
    "ComfyUI-Manager"
)
for node in "${REQUIRED_NODES[@]}"; do
    if [ -d "/root/ai/ComfyUI/custom_nodes/$node" ]; then
        echo "  [OK] $node"
    else
        echo "  [MISSING] $node - 必须安装"
    fi
done

# ═══ 6. 自定义节点版本（ComfyUI-WanVideoWrapper） ═══
if [ -d "/root/ai/ComfyUI/custom_nodes/ComfyUI-WanVideoWrapper" ]; then
    cd /root/ai/ComfyUI/custom_nodes/ComfyUI-WanVideoWrapper && git log -1 --oneline
fi
# Wan Video 任务必须依赖此节点，版本过旧可能导致 workflow 失败

# ═══ 7. 模型目录结构预检查 ═══
echo "=== 模型目录检查 ==="
COMFY_DIR=$(ls -d /root/ai/ComfyUI /root/ComfyUI 2>/dev/null | head -1)
if [ -z "$COMFY_DIR" ]; then
    echo "[ERROR] ComfyUI 目录不存在"
else
    for subdir in checkpoints vae clip text_encoders diffusion_models; do
        if [ -d "$COMFY_DIR/models/$subdir" ]; then
            echo "  [OK] models/$subdir"
        else
            echo "  [MISSING] models/$subdir - 将自动创建"
            mkdir -p "$COMFY_DIR/models/$subdir"
        fi
    done
fi

# ═══ 8. 磁盘空间检查 ═══
echo "=== 磁盘空间 ==="
df -h /root/ai/ | tail -1
# 要求：模型下载需要 30GB+ 可用空间，不足则停止

# ═══ 9. aria2c 工具 ═══
which aria2c || (apt update && apt install -y aria2)
aria2c --version | head -1

# ═══ 10. 网络连通性（HuggingFace） ═══
curl -sI https://huggingface.co/ | head -1
# 必须能访问 HuggingFace 才能下载模型
```

### 环境检查结果判定

| 检查项 | 最低要求 | 失败处理 |
|--------|----------|----------|
| GPU 显存 | ≥ 12GB | 停止，显存不足无法运行 |
| CUDA 版本 | ≥ 11.8 | 停止，升级驱动/CUDA |
| Python 版本 | ≥ 3.8 | 停止 |
| ComfyUI | 可响应 | 启动或重启 ComfyUI |
| ComfyUI-WanVideoWrapper | 必须存在 | 安装 custom node 才能跑 Wan |
| 磁盘空间 | ≥ 30GB free | 停止，清理空间 |
| HuggingFace 连通性 | 必须可达 | 检查网络/DNS |

**检查通过后再进入阶段零。**

## 进门汇报格式（强制执行）

```
当前状态：
- GPU：<nvidia-smi 型号 + 显存>
- CUDA：<torch.cuda.get_device_name()>
- Python：<python3 --version>
- ComfyUI：<running / not running, 版本>
- ComfyUI-WanVideoWrapper：<存在 / 缺失>
- workflow JSON：<存在路径 / 缺失>
- Flux 模型：<存在 / 缺失>
- Wan 模型：<存在 / 缺失>
- 磁盘空间：<df -h 剩余 GB>
- 环境检查：<通过 / 未通过>
- 当前卡点：<无 / 具体>
- 下一步动作：<具体>
```

## 进门检查清单（按顺序全部执行）

```bash
# 1. GPU 型号 + 显存
nvidia-smi --query-gpu=name,memory.total,memory.free --format=csv,noheader
# 记录显存：24GB / 40GB / 48GB / 80GB -> 决定下载哪个模型版本

# 2. ComfyUI 目录
ls -la /root/ai/ComfyUI/ || ls -la /root/ComfyUI/
# 确认 /root/ai/ 或 /root/ 下哪个有 ComfyUI

# 3. ComfyUI 是否响应
curl -s http://127.0.0.1:8188/system_stats | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['comfyui_version'], '|', d['devices'][0]['name'], '| VRAM', round(d['devices'][0]['vram_free']/1e9,1), 'GB free')"

# 4. aria2c
which aria2c || apt update && apt install -y aria2

# 5. workflow 目录
ls /root/ai/workflows/ 2>/dev/null || mkdir -p /root/ai/workflows && echo "已创建 workflow 目录"
```

## 阶段零：获取 + 验证 workflow JSON

**workflow JSON 是必须项。没有 JSON = 禁止开始任何任务。**

### 0.1 获取 workflow JSON（按优先级）

```bash
# 优先级 A：ComfyUI-WanVideoWrapper 自带
ls /root/ai/ComfyUI/custom_nodes/ComfyUI-WanVideoWrapper/examples/*.json 2>/dev/null
ls /root/ai/ComfyUI/custom_nodes/ComfyUI-WanVideoWrapper/*.json 2>/dev/null
cp /root/ai/ComfyUI/custom_nodes/ComfyUI-WanVideoWrapper/examples/*.json /root/ai/workflows/ 2>/dev/null

# 优先级 B：官方 PNG 示例
curl -s "https://comfyanonymous.github.io/ComfyUI_examples/flux/flux_dev_checkpoint_example.png" \
  -o /root/ai/workflows/flux_fp8.png
curl -s "https://comfyanonymous.github.io/ComfyUI_examples/wan/wan_i2v_example.png" \
  -o /root/ai/workflows/wan_i2v.png 2>/dev/null || true

# 优先级 C：GitHub API
curl -s "https://api.github.com/repos/comfyanonymous/ComfyUI/contents/example_workflows" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); [print(x['name'], x['download_url']) for x in d if isinstance(d,list)]" 2>/dev/null

# 优先级 D：ComfyUI-WanVideoWrapper GitHub
curl -s "https://api.github.com/repos/kijai/ComfyUI-WanVideoWrapper/contents/examples" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); [print(x['name'], x['download_url']) for x in d if isinstance(d,list)]" 2>/dev/null
```

### 0.2 所有来源失败 → 停止任务

```bash
# 检查是否获取到任意 workflow JSON
if [ ! -f /root/ai/workflows/workflow_flux_image.json ] && \
   [ ! -f /root/ai/workflows/workflow_wan_video.json ] && \
   [ -z "$(ls /root/ai/workflows/*.json 2>/dev/null)" ]; then
    echo "=== 致命错误 ==="
    echo "所有 workflow JSON 获取方式失败："
    echo "  1. ComfyUI-WanVideoWrapper examples/ 目录：无 JSON"
    echo "  2. 官方 PNG 示例：下载失败或无 embedded workflow"
    echo "  3. GitHub API：无法访问"
    echo ""
    echo "停止任务。不允许自己设计 workflow。"
    echo "需要人工介入获取官方 workflow JSON。"
    exit 1
fi
```

### 0.3 workflow 验证（获取后必须执行）

**验证失败 = 不允许进入正式生产。**

```python
import urllib.request, json

API = "http://127.0.0.1:8188"

def validate_workflow(wf_path):
    with open(wf_path) as f:
        wf = json.load(f)

    errors = []

    # 验证1：能被 ComfyUI 解析
    try:
        last_node = list(wf.keys())[-1]
    except:
        errors.append("workflow JSON 格式错误，无法解析")
        return errors

    # 验证2：检查 missing nodes（节点类型是否存在）
    with urllib.request.urlopen(f"{API}/object_info") as r:
        available_nodes = set(json.loads(r.read()).keys())

    for node_id, node in wf.items():
        cls = node.get("class_type", "")
        if cls not in available_nodes:
            errors.append(f"missing node: {cls} (node {node_id})")

    # 验证3：检查 missing models（模型文件是否存在）
    model_paths = []
    for node_id, node in wf.items():
        inp = node.get("inputs", {})
        for k, v in inp.items():
            if isinstance(v, str) and any(ext in v for ext in [".safetensors", ".pth", ".pt", ".ckpt"]):
                model_paths.append(v)

    import os
    for path in model_paths:
        if not os.path.exists(path):
            # 尝试检查相对路径
            abs_path = f"/root/ai/ComfyUI/{path}"
            if not os.path.exists(abs_path):
                errors.append(f"missing model: {path}")

    return errors

# 验证 Flux workflow
flux_wf = "/root/ai/workflows/workflow_flux_image.json"
if os.path.exists(flux_wf):
    errs = validate_workflow(flux_wf)
    if errs:
        print(f"Flux workflow 验证失败:")
        for e in errs:
            print(f"  - {e}")
    else:
        print("Flux workflow 验证通过")

# 验证 Wan workflow
wan_wf = "/root/ai/workflows/workflow_wan_video.json"
if os.path.exists(wan_wf):
    errs = validate_workflow(wan_wf)
    if errs:
        print(f"Wan workflow 验证失败:")
        for e in errs:
            print(f"  - {e}")
    else:
        print("Wan workflow 验证通过")
```

### 0.4 验证通过 → 才允许提交正式任务

```bash
# 验证通过后才能进入阶段四/五
# 验证失败：
#   - 修复路径 / 下载模型 / 更换 workflow
#   - 不允许绕过直接进入正式生产
```

## 阶段二：下载模型（基于 workflow 需求）

> 下载前必须先完成阶段负一（环境检查）和阶段零（workflow 验证）。两项都通过后才能下载模型。

下载前先读 workflow JSON 确定需要的模型：

```bash
# 分析 workflow 需要的模型文件
python3 << 'EOF'
import json, sys
try:
    with open("/root/ai/workflows/workflow_flux_image.json") as f:
        wf = json.load(f)
except:
    # 如果是 PNG，从 metadata 提取
    print("workflow JSON not found, will use direct download approach")
    sys.exit(0)

for node_id, node in wf.items():
    cls = node.get("class_type", "")
    inp = node.get("inputs", {})
    for k, v in inp.items():
        if isinstance(v, str) and any(ext in v for ext in [".safetensors", ".pth", ".pt", ".ckpt"]):
            print(f"{node_id} ({cls}).{k} = {v}")
EOF
```

### FLUX 模型下载（两种方案）

**方案 1：FP8 Checkpoint 版（推荐，显存 ≤24GB）**

```bash
# Flux1-dev FP8 量化版（~12GB），可直接用 CheckpointLoaderSimple 加载
ls /root/ai/ComfyUI/models/checkpoints/flux1-dev-fp8.safetensors || \
aria2c -x 16 -s 16 -k 1M \
  -d /root/ai/ComfyUI/models/checkpoints \
  -o flux1-dev-fp8.safetensors \
  "https://huggingface.co/Comfy-Org/flux1-dev/resolve/main/flux1-dev-fp8.safetensors"

# VAE（~300MB）
ls /root/ai/ComfyUI/models/vae/ae.safetensors || \
aria2c -x 16 -s 16 -k 1M \
  -d /root/ai/ComfyUI/models/vae \
  -o ae.safetensors \
  "https://huggingface.co/flux/ae/resolve/main/ae.safetensors"
```

**方案 2：完整 Flux Dev（需 >32GB 显存）**

```bash
# flux1-dev.safetensors（~23GB）
ls /root/ai/ComfyUI/models/diffusion_models/flux1-dev.safetensors || \
aria2c -x 16 -s 16 -k 1M \
  -d /root/ai/ComfyUI/models/diffusion_models \
  -o flux1-dev.safetensors \
  "https://huggingface.co/black-forest-labs/FLUX.1-dev/resolve/main/flux1-dev.safetensors"

# CLIP-L
ls /root/ai/ComfyUI/models/clip/clip_l.safetensors || \
aria2c -x 16 -s 16 -k 1M \
  -d /root/ai/ComfyUI/models/clip \
  -o clip_l.safetensors \
  "https://huggingface.co/comfyanonymous/flux_text_encoders/releases/download/v1.0/clip_l.safetensors"

# T5XXL FP8
ls /root/ai/ComfyUI/models/text_encoders/t5xxl_fp8_e4m3fn.safetensors || \
aria2c -x 16 -s 16 -k 1M \
  -d /root/ai/ComfyUI/models/text_encoders \
  -o t5xxl_fp8_e4m3fn.safetensors \
  "https://huggingface.co/Kijai/flux-fp8/resolve/main/t5xxl_fp8_e4m3fn.safetensors"

# VAE
ls /root/ai/ComfyUI/models/vae/ae.safetensors || \
aria2c -x 16 -s 16 -k 1M \
  -d /root/ai/ComfyUI/models/vae \
  -o ae.safetensors \
  "https://huggingface.co/flux/ae/resolve/main/ae.safetensors"
```

### Wan 模型下载

显存判断：
```bash
MEM_GB=$(nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits | awk '{print $1/1024}')
echo "显存: ${MEM_GB}GB"
```

```bash
# Wan2.1 I2V 模型（fp8 量化版 ~7GB，适合 24GB 显存）
ls /root/ai/ComfyUI/models/diffusion_models/I2V/Wan2.1-I2V-14B-480p_fp8_e4m3fn_scaled_KJ.safetensors || \
aria2c -x 16 -s 16 -k 1M \
  -d /root/ai/ComfyUI/models/diffusion_models/I2V \
  -o Wan2.1-I2V-14B-480p_fp8_e4m3fn_scaled_KJ.safetensors \
  "https://huggingface.co/ByteDance/animateerdream/resolve/main/Wan2.1-I2V-14B-480p_fp8_e4m3fn_scaled_KJ.safetensors"

# Wan VAE
ls /root/ai/ComfyUI/models/vae/Wan2.1_VAE.pth || \
aria2c -x 16 -s 16 -k 1M \
  -d /root/ai/ComfyUI/models/vae \
  -o Wan2.1_VAE.pth \
  "https://huggingface.co/ByteDance/animateerdream/resolve/main/Wan2.1_VAE.pth"

# Wan T5 encoder
ls /root/ai/ComfyUI/models/text_encoders/ || ls /root/ai/ComfyUI/models/clip/ || true
```

## 阶段三：启动 ComfyUI（如未运行）

> 进门时已通过 `curl 127.0.0.1:8188/system_stats` 确认 ComfyUI 状态。未运行时才执行本阶段。

```bash
COMFY_DIR=$(ls -d /root/ai/ComfyUI /root/ComfyUI 2>/dev/null | head -1)
echo "ComfyUI 目录: $COMFY_DIR"

cd "$COMFY_DIR"
nohup python3 main.py --listen 0.0.0.0 --port 8188 > /root/ai/comfyui.log 2>&1 &
sleep 8

# 验证
curl -s http://127.0.0.1:8188/system_stats && echo "ComfyUI OK" || echo "ComfyUI FAIL"
tail -n 30 /root/ai/comfyui.log
```

## 阶段四：执行 FLUX 最小化验证

### 4.1 确认模型存在

```bash
ls -lh /root/ai/ComfyUI/models/checkpoints/ | grep flux
ls -lh /root/ai/ComfyUI/models/vae/
# 必须有 flux1-dev-fp8.safetensors 或 flux1-dev.safetensors 才继续
```

### 4.2 确认 workflow JSON 可用

```bash
# 读取 workflow
cat /root/ai/workflows/workflow_flux_image.json | python3 -c "import sys,json; d=json.load(sys.stdin); print('workflow valid, nodes:', len(d))" 2>/dev/null || \
echo "workflow JSON 不存在，尝试从 PNG metadata 提取或使用 API"

# 如果没有 JSON，用简单 API 调用直接提交（见下方）
```

### 4.3 提交任务（直接通过 API）

> **适用：4090 24GB VRAM，flux1-dev-fp8.safetensors**
> FP8 checkpoint 已整合 CLIP-L + T5XXL，直接用 CheckpointLoaderSimple 加载即可。

```python
import urllib.request, json, time

API = "http://127.0.0.1:8188"

# ============================================================
# 正确的 Flux FP8 工作流节点链（4090 / 24GB 适用）
# CheckpointLoaderSimple → BasicGuider → FluxGuidance
#                              ↑                        ↓
# CLIPTextEncode (positive) ← ClipLoader ← t5xxl (已整合)
#                              ↓
# EmptyLatentImage → KSampler → VAEDecode → SaveImage
# ============================================================

# 第 1 步：检查可用节点（确认 Flux 节点已注册）
with urllib.request.urlopen(f"{API}/object_info") as r:
    nodes = json.loads(r.read()).keys()

# 参考官方 workflow 节点：CheckpointLoaderSimple, CLIPTextEncode(x2),
#   EmptySD3LatentImage, FluxGuidance, KSampler, VAEDecode, SaveImage
required = ["CheckpointLoaderSimple", "CLIPTextEncode", "EmptySD3LatentImage",
            "FluxGuidance", "KSampler", "VAEDecode", "SaveImage"]
missing = [n for n in required if n not in nodes]
if missing:
    print(f"[ERROR] 缺少节点: {missing}")
else:
    print("[OK] 所有必需节点已注册")

# 第 2 步：构建 Flux FP8 workflow（与官方 flux_dev_checkpoint.json 一致）
# 节点连接（官方 workflow 实际结构）：
#   CheckpointLoaderSimple → clip (index 1) → CLIPTextEncode(positive) → CONDITIONING
#   CheckpointLoaderSimple → clip (index 1) → CLIPTextEncode(negative) → CONDITIONING
#   CheckpointLoaderSimple → model (index 0) → KSampler
#   CheckpointLoaderSimple → vae (index 2) → VAEDecode
#   EmptySD3LatentImage → LATENT → KSampler
#   FluxGuidance(conditioning) → KSampler positive
#
# 说明：
#   - clip 来自 checkpoint 的 index 1（FP8 checkpoint 已整合 clip_l）
#   - t5xxl 来自 checkpoint 的 index 2（已整合，无需单独加载）
#   - guidance=3.5 是 Flux 推荐的默认值（范围 1.0~10.0，越高越遵循 prompt）
#   - 注意：Flux 没有 negative prompt 效果，cfg 强制设为 1.0
prompt = {
    "30": {
        "class_type": "CheckpointLoaderSimple",
        "inputs": {"ckpt_name": "flux1-dev-fp8.safetensors"}
    },
    "6": {
        "class_type": "CLIPTextEncode",
        "inputs": {
            "text": "a cute 10-year-old chinese girl in yellow raincoat and red boots, studio ghibli style, watercolor illustration, masterpiece, best quality",
            "clip": ["30", 1]  # index 1 = clip_l from FP8 checkpoint
        }
    },
    "33": {
        "class_type": "CLIPTextEncode",
        "inputs": {
            "text": "blurry, low quality, worst quality, deformed, ugly",  # Flux dev ignored, cfg must be 1.0
            "clip": ["30", 1]
        }
    },
    "27": {
        "class_type": "EmptySD3LatentImage",
        "inputs": {"width": 512, "height": 512, "batch_size": 1}
    },
    "35": {
        "class_type": "FluxGuidance",
        "inputs": {"guidance": 3.5}  # Flux 核心 guidance 值
    },
    "31": {
        "class_type": "KSampler",
        "inputs": {
            "model": ["30", 0],           # index 0 = diffusion model
            "seed": 12345,
            "steps": 25,                  # Flux 推荐 20~50 步
            "cfg": 1.0,                   # ★ Flux 必须 cfg=1.0，negative prompt 被忽略
            "sampler_name": "euler",
            "scheduler": "simple",
            "positive": ["35", 0],        # FluxGuidance output → positive
            "negative": ["6", 0],         # CLIPTextEncode negative → negative
            "latent_image": ["27", 0]     # EmptySD3LatentImage → latent
        }
    },
    "8": {
        "class_type": "VAEDecode",
        "inputs": {"samples": ["31", 0], "vae": ["30", 2]}  # index 2 = vae from FP8 checkpoint
    },
    "9": {
        "class_type": "SaveImage",
        "inputs": {"images": ["8", 0], "filename_prefix": "flux_fp8_test"}
    }
}

# 第 3 步：提交（使用官方同款节点 ID，与 flux_dev_checkpoint.json 保持一致）
data = json.dumps({"prompt": prompt, "last_node_id": "9"}).encode()
req = urllib.request.Request(f"{API}/prompt", data=data, headers={"Content-Type": "application/json"})
resp = json.loads(req.read().decode())
prompt_id = resp["prompt_id"]
print(f"提交成功: {prompt_id}")
```

**节点引脚对应关系（Flux FP8 checkpoint，来自官方 workflow）：**

| 节点 ID | 类型 | 说明 |
|---------|------|------|
| 30 | CheckpointLoaderSimple | index 0=model, index 1=clip, index 2=vae |
| 6 | CLIPTextEncode | positive prompt conditioning |
| 33 | CLIPTextEncode | negative prompt conditioning（Flux 不生效，cfg=1.0） |
| 27 | EmptySD3LatentImage | 生成 latent 空图 |
| 35 | FluxGuidance | guidance=3.5 → KSampler positive |
| 31 | KSampler | 核心采样器，cfg 强制 1.0 |
| 8 | VAEDecode | latent → image |
| 9 | SaveImage | 输出 PNG |

> **★ 关键：Flux 必须 cfg=1.0** — 官方 Note 说得很清楚：Flux dev/schnell 没有 negative prompt 效果，cfg 设任何值都没用，必须为 1.0。guidance 控制由 FluxGuidance 节点负责（dev 版），schnell 版则不需要 FluxGuidance 节点。

**Schnell 工作流（无 FluxGuidance，节点更少）**

Flux Schnell 是蒸馏版，速度更快，不需要 FluxGuidance 节点：

```python
prompt = {
    "30": {
        "class_type": "CheckpointLoaderSimple",
        "inputs": {"ckpt_name": "flux1-schnell-fp8.safetensors"}
    },
    "6": {
        "class_type": "CLIPTextEncode",
        "inputs": {"text": "a cute 10-year-old chinese girl in yellow raincoat, studio ghibli style", "clip": ["30", 1]}
    },
    "27": {
        "class_type": "EmptySD3LatentImage",
        "inputs": {"width": 512, "height": 512, "batch_size": 1}
    },
    "31": {
        "class_type": "KSampler",
        "inputs": {
            "model": ["30", 0], "seed": 12345, "steps": 4,  # Schnell 只需 4 步
            "cfg": 1.0, "sampler_name": "euler", "scheduler": "simple",
            "positive": ["6", 0], "negative": [], "latent_image": ["27", 0]
        }
    },
    "8": {
        "class_type": "VAEDecode",
        "inputs": {"samples": ["31", 0], "vae": ["30", 2]}
    },
    "9": {
        "class_type": "SaveImage",
        "inputs": {"images": ["8", 0], "filename_prefix": "flux_schnell_test"}
    }
}
```

> **Dev vs Schnell 对比**：Dev 需要 FluxGuidance + 25 步；Schnell 无需 guidance + 只需 4 步。两者 cfg 都必须为 1.0。

>
> **显存优化**：如果 24GB 不够，将 width/height 从 512 降到 256，或 batch_size 降到 1。

### 4.4 监控（强制）

```python
import time
for i in range(40):  # 最多 10 分钟
    with urllib.request.urlopen(f"{API}/queue") as r:
        q = json.loads(r.read())
    running = len(q.get("queue_running", []))
    pending = len(q.get("queue_pending", []))
    print(f"[{i*15}s] running={running} pending={pending}")
    if running == 0 and pending == 0:
        break
    time.sleep(15)

# 查结果
with urllib.request.urlopen(f"{API}/history") as r:
    h = json.loads(r.read())
last_id = list(h.keys())[-1]
status = h[last_id]["status"]["status_str"]
msgs = h[last_id].get("status", {}).get("messages", [])
for msg in msgs:
    if isinstance(msg, list) and msg[0] == "execution_error":
        print(f"错误: {msg[1].get('node_type')} - {msg[1].get('exception_message')}")
print(f"最终状态: {status}")
```

### 4.5 验证 PNG

```bash
# 必须看到这个才叫完成
find /root/ai/ComfyUI/output -name "*.png" -mmin -10 | head -5

# 如果有输出 -> FLUX 验证成功
# 如果无输出 -> 继续排查，10 分钟内必须解决
```

## 阶段五：Wan 视频（先 FLUX 成功再进入）

### 5.1 准备输入图片

```bash
# 用 FLUX 生成的图片
LATEST_PNG=$(find /root/ai/ComfyUI/output -name "*.png" -mmin -10 | sort -t mmin -r | head -1)
echo "使用图片: $LATEST_PNG"
cp "$LATEST_PNG" /root/ai/input/wan_ref.png
```

### 5.2 提交 Wan I2V 任务

```python
# 使用 ComfyUI-WanVideoWrapper 的官方 workflow
# 从 examples/ 目录找到 JSON
```

### 5.3 验证 MP4

```bash
find /root/ai/ComfyUI/output -name "*.mp4" -mmin -10 | head -5
# 必须有 MP4 才叫完成
```

## 禁止行为

1. **禁止没有 workflow JSON 就开始任务** — JSON 必须项，获取失败必须停止并汇报
2. **禁止自己设计 / 拼接节点** — 不允许发明 fallback workflow，必须用官方/已验证 workflow
3. **禁止跳过 workflow 验证** — 获取 JSON 后必须通过阶段零全部验证才能进入正式生产
4. **禁止重复下载** — `ls` 确认不存在才下载
5. **禁止混 FLUX 和 Wan** — 先 FLUX → PNG → 再 Wan → MP4
6. **禁止 GPU 0% 停止监控** — 每 15 秒查一次，10 分钟内必须出结果
7. **禁止"理论上可以运行"** — 必须看到真实文件才算完成
8. **禁止显存不够还下载完整模型** — 24GB 用 fp8 版，不要下 fp16 完整版

## 停止汇报模板（workflow JSON 获取失败时）

```
=== 致命：workflow JSON 获取失败 ===

尝试过的方式：
  [A] ComfyUI-WanVideoWrapper examples/ 目录：无 JSON 文件
  [B] 官方 PNG 示例：下载失败 / 无 embedded workflow
  [C] GitHub API：无法访问
  [D] ComfyUI-WanVideoWrapper GitHub：无法访问

结论：无法获取 workflow JSON。

操作：停止任务，等待人工介入。

需要的帮助：请提供以下任一：
  1. workflow_flux_image.json 文件
  2. workflow_wan_video.json 文件
  3. ComfyUI 操作权限（用于导出已有 workflow）
  4. 官方 workflow 下载链接
```

## 验收标准

| 阶段 | 命令 | 达标条件 |
|---|---|---|
| FLUX | `find /root/ai/ComfyUI/output -name "*.png" -mmin -10` | 有输出 |
| Wan | `find /root/ai/ComfyUI/output -name "*.mp4" -mmin -10` | 有输出 |

没有真实文件 = 任务未完成 = 继续排查，直到有文件。