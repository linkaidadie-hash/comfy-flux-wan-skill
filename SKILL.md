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
---

# ComfyUI + Flux/Wan 执行技能

## 核心理念

**workflow = 磁盘上真实 `.json` 文件。**
**没有 workflow JSON = 禁止开始任务。**
禁止用文字描述节点结构代替 JSON 文件。

## 进门汇报格式（强制执行）

```
当前状态：
- GPU：<nvidia-smi 型号 + 显存>
- ComfyUI：<running / not running>
- workflow JSON：<存在路径 / 缺失>
- Flux 模型：<存在 / 缺失>
- Wan 模型：<存在 / 缺失>
- 自定义节点：<wan wrapper 存在 / 缺失>
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

## 阶段一：下载模型（基于 workflow 需求）

> 下载前必须先完成阶段零的 workflow 验证。验证通过后才能下载模型。

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

```python
import urllib.request, json, time

API = "http://127.0.0.1:8188"

# 简单 Flux workflow（基于 FP8 checkpoint 方案）
# CheckpointLoaderSimple -> CLIPTextEncode -> KSampler -> VAEDecode -> SaveImage
# （FP8 checkpoint 版用标准 SD 节点，无需 Flux 专用节点）

prompt = {
    "3": {
        "class_type": "CheckpointLoaderSimple",
        "inputs": {"ckpt_name": "flux1-dev-fp8.safetensors"}
    },
    "4": {
        "class_type": "CLIPTextEncode",
        "inputs": {
            "text": "a cute 10-year-old chinese girl in yellow raincoat and red boots, studio ghibli style, watercolor illustration, masterpiece",
            "clip": ["3", 1]
        }
    },
    "5": {
        "class_type": "KSampler",
        "inputs": {
            "model": ["3", 0],
            "seed": 12345,
            "steps": 20,
            "cfg": 1.0,
            "sampler_name": "euler",
            "scheduler": "normal",
            "positive": ["4", 0],
            "negative": [],
            "latent_image": [{"class_type": "EmptyLatentImage", "inputs": {"width": 512, "height": 512, "batch_size": 1}}],
            "denoise": 1.0
        }
    }
}

# 正确格式：latent_image 应该是 node reference，不是嵌套 dict
prompt = {
    "3": {
        "class_type": "CheckpointLoaderSimple",
        "inputs": {"ckpt_name": "flux1-dev-fp8.safetensors"}
    },
    "4": {
        "class_type": "CLIPTextEncode",
        "inputs": {"text": "a cute 10-year-old chinese girl in yellow raincoat and red boots, studio ghibli style, watercolor illustration", "clip": ["3", 1]}
    },
    "5": {
        "class_type": "EmptyLatentImage",
        "inputs": {"width": 512, "height": 512, "batch_size": 1}
    },
    "6": {
        "class_type": "KSampler",
        "inputs": {"model": ["3", 0], "seed": 12345, "steps": 20, "cfg": 1.0, "sampler_name": "euler", "scheduler": "normal", "positive": ["4", 0], "negative": [], "latent_image": ["5", 0], "denoise": 1.0}
    },
    "7": {
        "class_type": "VAEDecode",
        "inputs": {"samples": ["6", 0], "vae": ["3", 2]}
    },
    "8": {
        "class_type": "SaveImage",
        "inputs": {"images": ["7", 0], "filename_prefix": "flux_test"}
    }
}

data = json.dumps({"prompt": prompt, "last_node_id": "8"}).encode()
req = urllib.request.Request(f"{API}/prompt", data=data, headers={"Content-Type": "application/json"})
resp = json.loads(req.read().decode())
prompt_id = resp["prompt_id"]
print(f"提交: {prompt_id}")
```

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