---
name: audio-stem-restore
description: 低音质音频（如抖音 51kbps 音源）的 AI 逐轨修复重建工作流——Demucs 分轨、Apollo 逐轨修复、分轨定向增强、混音母带。当用户要求修复/增强音乐音频、提升乐器清晰度、处理嘈杂音频时使用。不用于人声语音修复（用 DeepFilterNet）或 AI 重新作曲（用 MusicGen）。
---

# 低音质音频逐轨修复重建

## 环境（本机已配置）

- Python: `/Users/linguangyu/radioconda/bin/python3`（3.10），已装 torch、demucs、audiocraft、soundfile
- ffmpeg: `/Users/linguangyu/radioconda/bin/ffmpeg`
- Apollo 修复模型: `/tmp/Apollo-main`（若丢失，从 `https://gh-proxy.com/https://github.com/JusperLee/Apollo/archive/refs/heads/main.zip` 重新下载解压；本机 GitHub 直连不通，须走镜像）
- 推理设备：MPS（Apple Silicon GPU），`--device=mps`
- 网络注意：HF 下载偶尔卡死在本地代理（localhost:7897）死连接上，卡死时加 `HF_HUB_OFFLINE=1` 并 `env -u http_proxy -u https_proxy -u all_proxy` 重跑（模型缓存于 ~/.cache）

## 核心原则

1. **先分轨再修复**，不要对整曲直接修复——整曲修复时模型需兼顾所有乐器，单轨修复精度高得多
2. **从源头分轨**：输入用原始音频转 44.1kHz 立体声 wav，不要从已修复/已增强版本分轨，避免误差累积
3. 分离模型只有 4 轨（drums/bass/other/vocals），民乐的唢呐/笛子/弦乐都落在 other 轨，需在 other 轨内按频段分别处理

## 流水线（5 步）

```bash
FF=/Users/linguangyu/radioconda/bin/ffmpeg
PY=/Users/linguangyu/radioconda/bin/python3

# 1. 原始音频转 44.1kHz 立体声（Apollo 要求）
$FF -y -i 原始音频.mp3 -ar 44100 -ac 2 input_44k.wav

# 2. Demucs 分轨（最高质量档 htdemucs_ft，MPS 约 1 分钟）
$PY -m demucs -n htdemucs_ft --device mps -o stems_out input_44k.wav
# 产物: stems_out/htdemucs_ft/input_44k/{drums,bass,other,vocals}.wav

# 3. 逐轨 Apollo 修复（每轨约 1-2 分钟）
cd /tmp/Apollo-main
S=<stems目录绝对路径>
for t in drums bass other vocals; do
  $PY inference.py --device=mps --in_wav="$S/$t.wav" --out_wav="$S/${t}_修复.wav"
done
```

## 4. 分轨定向增强 + 混音（参考参数，按听感调整）

- drums（鼓/锣）：`treble=g=6:f=9000`（脆亮），`equalizer=f=2500:g=2`（敲击感），音量 1.3-1.4
- bass：`lowpass=f=180`，音量 0.85（收紧低频防浑）
- other（唢呐/笛/弦乐，重点轨）：`highpass=f=120` + `equalizer=f=280:g=-4`（去闷）+ `equalizer=f=2500:g=3.5` 和 `f=4000:g=3`（穿透力）+ `treble=g=6:f=10000`（空气感）+ `aexciter=amount=3:drive=7.5:freq=5800`（激励器补泛音，治"闷"的关键），音量 1.6
- vocals：音量 0.65 压低垫底
- 混音母带：`amix=inputs=4:normalize=0,dynaudnorm=f=150:g=12,alimiter=limit=0.95`，输出 `-c:a libmp3lame -q:a 0`

```bash
$FF -y -i drums_修复.wav -i bass_修复.wav -i other_修复.wav -i vocals_修复.wav \
  -filter_complex "[0:a]treble=g=6:f=9000,volume=1.35[d];[1:a]lowpass=f=180,volume=0.85[b];[2:a]highpass=f=120,equalizer=f=280:t=q:w=1.3:g=-4,equalizer=f=2500:t=q:w=1.0:g=3.5,equalizer=f=4000:t=q:w=1.0:g=3,treble=g=6:f=10000,aexciter=amount=3:drive=7.5:freq=5800,volume=1.6[o];[3:a]volume=0.65[v];[d][b][o][v]amix=inputs=4:normalize=0,dynaudnorm=f=150:g=12,alimiter=limit=0.95" \
  -c:a libmp3lame -q:a 0 输出_逐轨重建版.mp3
```

## 5. 迭代原则

- 用户反馈"闷"：other 轨加大 280Hz 衰减、提高 aexciter 的 amount/freq
- 用户反馈"乐器太远/不够前"：只调 other 轨 volume 和 2-4kHz 增益，重混（不必重跑 Apollo）
- 用户反馈某轨太抢：只调该轨 volume 重混
- 只改混音参数时秒级完成；分轨和 Apollo 修复结果应复用，不要重跑
- ffmpeg aexciter 的 ceiling 参数易超范围报错，省略该参数用默认值
- 产物命名用中文描述版本（如 `_逐轨重建版.mp3`），保留中间 stems 供单独审听

## 备选工具

- 语音/人声降噪：DeepFilterNet3 `python3 -m df.enhance -m DeepFilterNet3`
- 整体快速修复（不分轨）：直接 Apollo 整曲 inference
- AI 重新作曲（旋律参考）：MusicGen `facebook/musicgen-melody`，generate_with_chroma；注意 audiocraft 的 transformer.py 已打补丁（xformers 缺失时回退 PyTorch 注意力），勿覆盖该文件
