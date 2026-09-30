# upseem 的 ComfyUI 工作流

这里存放个人日常使用和验证过的工作流。上游官方模板继续保留在仓库原有的 `templates/`、`packages/` 等目录；本目录不加入官方 `templates/index.json` 和 `bundles.json`，避免以后同步 `Comfy-Org/workflow_templates` 时产生大量冲突。

## 使用方法

下载需要的 JSON 后，在 ComfyUI 中选择 **工作流 → 打开**。模型文件应放入工作流节点注明的模型目录。导入后请重新选择本机的视频和图片，不要直接运行示例占位素材。

这些工作流按实际项目环境编写，部分依赖以下自定义节点：

- [upseem/ComfyUI_video_loop](https://github.com/upseem/ComfyUI_video_loop)：视频抽帧、磁盘循环和合成。
- [upseem/Comfyui-video-upscale](https://github.com/upseem/Comfyui-video-upscale)：超分批次读取与写回节点。

MiniMax H3 Character Swap LoRA 工作流只使用 ComfyUI 原生节点，但模型受其各自许可证约束。

## 视频高清

### SeedVR2 3B Int8 · RTX 5090

文件：[`video_upscale/seedvr2_3b_int8_video_upscale_5090.json`](video_upscale/seedvr2_3b_int8_video_upscale_5090.json)

适合32GB显存的RTX 5090。工作流先把源视频抽帧到磁盘，再在Resize和VAE之前读取连续小批次，避免整段4K帧进入内存。每块保留49帧，左右各读取8帧上下文；内部继续使用ComfyUI原生SeedVR2 latent自动分块和时间重叠。

模型：

```text
models/diffusion_models/seedvr2_3b_int8_convrot.safetensors
models/vae/seedvr2_ema_vae_fp16.safetensors
```

长视频运行ComfyUI时必须使用 `--cache-none`，否则原生循环的执行缓存可能保留每轮大张量，导致内存逐块增长。

### SeedVR2 3B Int8 · RTX PRO 6000 96GB

文件：[`video_upscale/seedvr2_3b_int8_video_upscale_6000_96gb.json`](video_upscale/seedvr2_3b_int8_video_upscale_6000_96gb.json)

96GB显存的生产优化版。参数来自1080×1920、30fps、2×输出的实测：

```text
chunk_size = 57
context = 4
单块最多65帧
PNG compress_level = 0（无损、写入更快、临时文件更大）
```

实测VAE阶段显存峰值约64.5GB，SeedVR2主推理约18GB。PNG压缩等级只影响速度和临时文件大小，不影响像素质量。

### SeedVR2 7B FP16 · RTX PRO 6000 96GB

文件：[`video_upscale/seedvr2_7b_fp16_video_upscale_6000_96gb.json`](video_upscale/seedvr2_7b_fp16_video_upscale_6000_96gb.json)

使用7B FP16非量化模型，适合关键镜头和质量对照：

```text
models/diffusion_models/seedvr2_7b_fp16.safetensors
models/vae/seedvr2_ema_vae_fp16.safetensors
```

7B模型更慢、显存更高，不保证每个镜头都明显优于3B Int8。首次建议设置 `duration=2`；如果显存紧张，把 `chunk_size` 从57降为25或33。不要关闭磁盘分块。

## MiniMax H3 换人

### Character Swap LoRA · 视频+人物图

文件：[`character_swap/minimax_h3_character_swap_lora_beach_5s.json`](character_swap/minimax_h3_character_swap_lora_beach_5s.json)

输入：

```text
<Video 1>：原动作、镜头和背景视频
<Picture 1>：替换后的新人物图片
```

使用 [akatz-ai/MiniMax-H3-Character-Swap-LoRA](https://huggingface.co/akatz-ai/MiniMax-H3-Character-Swap-LoRA)：

```text
h3_character_swap_pro4500_1000.safetensors
strength = 1.0
Turbo LoRA = 关闭
steps = 20
sampler = res_multistep
scheduler = simple
```

作者推荐4～5秒、24fps、连续镜头。该LoRA为实验模型：背景保留、动作时序、表情和硬切并不保证稳定。提示词应准确指出源视频中要替换的人；若描述不存在的人物，模型可能直接动画化参考图。

该LoRA不是Apache-2.0，使用MiniMax H3 Community License，并包含地域和商业使用限制。使用前阅读模型仓库的完整许可证。

### Pose Control · 人物图+动作骨架

文件：[`character_swap/minimax_h3_pose_control_character_replacement.json`](character_swap/minimax_h3_pose_control_character_replacement.json)

源视频只用于提取Pose，不把原人物RGB作为H3身份参考；`<Picture 1>`是唯一人物外观参考。适合完整视频参考压过人物图片、导致换人失败的情况。

额外模型：

```text
models/model_patches/minimax_h3_fun_controlnet_union_pruned_int8_convrot.safetensors
models/checkpoints/sdpose_wholebody_fp16.safetensors
models/diffusion_models/rt_detr_v4-x-hgnet_fp16.safetensors
```

Pose控制能保留大动作和节奏，但不会逐像素保留背景。多人场景需要调整检测数量，身份和人数仍可能漂移。

## 同步官方上游

建议一次性添加官方远端：

```bash
git remote add upstream https://github.com/Comfy-Org/workflow_templates.git
git fetch upstream
git switch main
git merge --ff-only upstream/main
# 如果个人仓库已有额外提交，使用普通merge保留历史：
# git merge upstream/main
git push origin main
```

个人工作流只在 `user_workflows/upseem/` 内新增文件；不要为了展示个人工作流修改官方 `templates/index.json`、`bundles.json`、`packages/` 或版本号。这样上游同步通常只会形成独立目录，不与官方模板发生内容冲突。

## 状态说明

- JSON可被ComfyUI导入不等于模型效果已经通过全部素材验证。
- 生成式换人和超分都应先用2～5秒样片验证。
- 工作流引用的模型与外部LoRA许可证独立于本仓库许可证。
