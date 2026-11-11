# 十步执行细节

## 目录

1. 工程初始化
2. BGM：合成或实测
3. 文案压缩与字数预算
4. 配音与 SRT
5. 时间轴生成（吸附重拍）
6. 音效合成与钉帧
7. 静帧自检
8. 渲染与双版本
9. 音频峰值实测
10. 物料与交付说明

---

## 1. 工程初始化

```bash
npm init -y
npm i remotion @remotion/cli react react-dom
```

`src/index.ts` 用 `registerRoot(RemotionRoot)`；`src/Root.tsx` 注册 Composition：

```tsx
<Composition id="MyVideo" component={Main}
  durationInFrames={1610} fps={30} width={1920} height={1080} />
```

`package.json` 至少两个脚本：`dev` = `remotion dev src/index.ts`，`render` = `remotion render src/index.ts MyVideo out/<name>.mp4`。

**新建工程，不在旧工程里改。** 用户说"新建一个项目"时不要复用同目录，哪怕画面元素能抄。

## 2. BGM：合成或实测

曲库/用户给的曲子先实测，别直接信文件名或描述：

```python
import librosa
y, sr = librosa.load(path, sr=None, mono=True)
tempo, beats = librosa.beat.beat_track(y=y, sr=sr, units='time')
centroid = librosa.feature.spectral_centroid(y=y, sr=sr).mean()
rms = librosa.feature.rms(y=y).mean()
```

判读：
- `tempo` 若与听感差一倍（常见 51.7 报而实为 103.4），用自相关或 `onset_envelope` 参数复核，取听感一致的半/倍值。
- `spectral_centroid > 1200` 且 `rms > 0.35` 通常意味着强鼓点/高能量，配"理性科普"会打架。
- 判为不合适就自合成：给定和弦走向（如 Am–F–C–Em）、极简钢琴层 + pad 层 + 低频脉冲 + 软混响，用 `numpy` 生成、`soundfile` 写 44100Hz PCM_16，并把种子写进脚本保证可复现。

重拍表直接由 BPM 推：`beat_n_sec = n * 60 / BPM`，帧号 `round(beat_n_sec * FPS)`。把整张表存成 `bgm.json` / `timeline.json`，别留在对话里。

## 3. 文案压缩与字数预算

- 字数预算 = 目标秒数 × 3.3（中文 TTS 经验值）。目标 53s → 约 175–200 字。
- 结构：钩子（1 句）→ 模型设定（1 段）→ 个体推导（1 段）→ 均衡与结论（1 段）→ 条件（1 段）→ 启示（1 句）。
- 压缩手法：删解释性从句、把"两个人"改"双方"、去掉重复主语；**知识点一个都不删**。
- 定稿后**不再改字数**。改字数必须重跑第 5 步脚本并同步 `timeline.ts`。

## 4. 配音与 SRT

逐段合成 → 逐段量时长 → 生成 SRT。时长表形如：

| 段 | 文本字数 | 实测时长 |
|---|---|---|
| S1 | 42 | 7.68s |
| S2 | 58 | 10.38s |

SRT 由第 5 步的边界统一生成，保证画面、内嵌字幕、外挂字幕三者同源。

## 5. 时间轴生成（吸附重拍）

核心算法（写成 `assets/build_timeline.py`，输出 `timeline.json` 与人读的 `timeline.ts`）：

```python
BEAT = 60 / BPM            # 秒/拍
FRAME_PER_BEAT = BEAT * FPS

def snap_up(sec):          # 配音结束时刻向上吸附到最近重拍
    return math.ceil(sec / BEAT - 1e-9)

def beat_frame(n):
    return round(n * FRAME_PER_BEAT)

# 段起点 = 上一段终点的吸附拍；段时长 = beat_frame(next) - beat_frame(cur)
# 配音从段起点 + narrFrom 帧开始（留 4–8 帧呼吸）
```

字幕断行（子函数 `wrap()`）：

```python
# 1) 先按 ，；：。 切 clause，标点留在 clause 末尾
# 2) 再按 n=24 字装箱，单 clause 超长才硬断
# 3) 每行写进 SRT 时保留原标点（whiteSpace: "pre" 才不会丢空格）
```

产出后立刻核对：总帧数 = 末段终点；每段时长 = 配音时长的 1.0–1.25 倍（太紧会压字，太松会拖节奏）。

## 6. 音效合成与钉帧

- 每个音色一个生成函数：短促单音钢琴、木琴、低音钢琴、颤音合成器、和弦、转场。峰值 0.16–0.34，时长 0.26–1.4s。
- 触发点表写成 `(beat, 动作说明)` 列表，代码里用 `beatF(n)` 转帧，**不写绝对秒**。
- 音效数 = 关键动作数（示例 19 条，其中 5 条转场）；每条注释写清"这一拍画面上发生了什么"。

## 7. 静帧自检

```bash
npx remotion still src/index.ts MyVideo out/qa/f180.png --frame=180
```

- 每段至少 1 张；转折帧（高亮落定、标题落定）单独出图。
- 检查项：文字是否出框、图形是否压字幕条、主体是否被裁、留白是否失衡。
- 改 TSX 后出图仍是旧样式 → 删 `node_modules\.cache` 重跑。

## 8. 渲染与双版本

```bash
npx remotion render src/index.ts MyVideo out/<name>.mp4
npx remotion render src/index.ts MyVideo out/<name>-nobgm.mp4 --props=props-nobgm.json
```

`props-nobgm.json` 内容 `{"bgm":false}`，组件里用 `bgm?: boolean` 控制 `Audio` 挂载。**Windows 必须传文件**，内联 JSON 会被剥掉引号。

渲染完 `ffprobe` 核一遍：`width/height/nb_frames/r_frame_rate/duration`。

## 9. 音频峰值实测

```bash
ffmpeg -i out/<name>.mp4 -vn -ac 1 -ar 16000 -f f32le tmp.raw
```

用 Python `numpy.fromfile` 读后算 `abs(peak)`/RMS。峰值 > 0.95 视为削波风险；调电平后重渲，再看一次。示例：人声 1.0→0.80、BGM 0.32→0.24、转场 0.55→0.90，峰值从 1.000（削平）回到 0.890。

## 10. 物料与交付说明

- 封面：先试代码渲染（`remotion still` 出 1920×1080 与 1080×1440 两版），字不会糊且与成片同语言；需要更强视觉再由设计工具出图。
- 标题：给 3–7 条，分「主推 / 搜索向 / 口语向」三档；简介给「通用 120 字 / 短版 60 字 / 一句话钩子」三版；再补话题标签。
- `交付说明.md` 必须含：成片表、BPM 与节拍口径、段落边界表、重拍落点表、六段文案、音效表、混音电平、工程结构、运行命令、确定性渲染说明。
