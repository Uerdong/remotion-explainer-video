# Remotion 工程约定

## 目录

1. 工程骨架
2. theme.ts（设计 token）
3. components.tsx（四种基础件）
4. timeline.ts（唯一事实源）
5. 图层顺序
6. 卡点与动效写法
7. 常见坑

---

## 1. 工程骨架

```
src/
├── index.ts          # registerRoot
├── Root.tsx          # Composition 注册（正片 + 封面 still）
├── Main.tsx          # 六层拼装
├── timeline.ts       # 唯一事实源
├── theme.ts          # 颜色/字体/动效曲线
├── components.tsx    # Figure / Pill / Eyebrow / Subtitle / 换行工具
├── scenes/S1..S6.tsx # 逐段独立组件
public/
├── audio/bgm.wav
├── audio/narr/s1..s6.wav
├── audio/sfx/*.wav
└── subtitles.srt
assets/               # 合成/分析脚本，不放进渲染依赖
out/                  # 成片、字幕、qa/ 静帧
```

一段一个组件、一段一个 `Sequence`。段内动画只用本地 `frame`（Sequence 已把时间归零），不要用全局帧号算局部进度。

## 2. theme.ts

只放 token，不放逻辑。两类用法：

- **同一套设计沿用**：只改值不改键名（`bg`/`ink`/`accent`/`panel`…），换肤时可以一次改完、组件零改动。
- **需要新语义**：追加新键（`panel`/`gold`/`red`/`green`/`rim`），旧键保留继续工作。

```ts
export const COLORS = { bg:'#FFFFFF', accent:'#1677ff', ink:'#444444', … };
export const FONT_SANS = '"PingFang SC","Microsoft YaHei",system-ui,sans-serif';
export const FONT_SERIF = '"SimSun","STSong","Noto Serif SC",serif';
export const pop = (f:number, d=0) => spring({frame:f-d, fps:30, config:{damping:200}});
```

字体永远给字族栈而不是单一字体名，否则渲染机缺字直接回退。

## 3. components.tsx

四个基础件撑起全部画面，其余都是它们的组合：

| 组件 | 定位约定 | 关键 prop |
|---|---|---|
| `Figure` | 以**底边中心**为锚点（`translate(-50%,-100%)`，`transformOrigin: bottom center`） | `x,y`（脚位置）、`scale`、`color`、`bodyRot`、`headRot`、`lookDown`、`opacity` |
| `Pill` | 以中心为锚点（`translate(-50%,-50%)`） | `x,y,w,h`、`bg`、`color`、`border`、`radius`、`fontSize` |
| `Eyebrow` | 左上角定位的段标题 | `x,y`、`text` |
| `Subtitle` | 固定 `width` + `bottom`，`whiteSpace:"pre"` | `lines`、`color`、`bg` |

`Figure` 用脚位置当锚点，是为了"站在某条基线上"这类构图不用做心算；改 scale 时脚不动，只缩身体。

## 4. timeline.ts

```ts
export const FPS = 30, BPM = 104;
export const BEAT_FRAMES = (60 / BPM) * FPS;      // 17.3077
export const beatF = (n:number) => Math.round(n * BEAT_FRAMES);
export const TOTAL_FRAMES = 1610;
export const TOTAL_SEC = TOTAL_FRAMES / FPS;

export const SEGMENTS = [
  { id:'S1', from:0, duration:260, narrFrom:5, narrFrames:231, src:'s1.wav', text:'…' },
  …
];
export const SFX = [ { at: beatF(15), file:'transition.wav', vol:0.9, note:'转场→矩阵' }, … ];
```

`Main.tsx` 只消费它：`SEGMENTS.map(s => <Sequence from={s.from} durationInFrames={s.duration}> …)`。新增段落只改这一处。

## 5. 图层顺序

自下而上：BGM `Audio` → 画面 Sequence → 配音 Sequence → 音效 Sequence → 视觉叠层（暗角/颗粒）→ 字幕 Sequence。

叠层必须在字幕之下，否则字幕被压暗；必须在画面之上，否则没有氛围。

## 6. 卡点与动效写法

- 入场：`pop(frame, localDelay)`；位移用 `interpolate(frame, [a,b], [x0,x1], {extrapolateRight:'clamp'})`。
- 摆动：`Math.sin(frame/18) * amp`，禁 `Math.random()`。
- 转场：段起点做 8–12 帧的透明度 + 缩放交叉，确保落在 `beatF(n)` 而不是任意帧。
- 高亮一格矩阵：高亮底色 `rgba(accent, 0.14)` + 边框 3px `accent` + 其余格 `opacity 0.55`，一个高亮只解释一句话。

## 7. 常见坑

| 现象 | 原因 | 处理 |
|---|---|---|
| 静帧是旧样式 | Remotion bundler 缓存 | 删 `node_modules\.cache` 重跑 still |
| 字幕被压成三行、"。"甩单行 | 容器 `maxWidth` 被内容反推 | 固定 `width: 1460` + `whiteSpace:"pre"` |
| 小人腿被字幕框切 | 字幕条 z 层高于人物且位置冲突 | 人物 `top` 上移、scale 缩小、标签移到矩阵下方 |
| 标题压图形顶弧 | 标题与圆环 y 区间重叠 | 标题上移，或让绘制提前（避免右半边空 2s） |
| 3D 全黑 | 原生 `@react-three/fiber` Canvas 在 Remotion 下无输出 | 用 `@remotion/three` 的 `ThreeCanvas` 并显式传 `width/height` |
| 输出峰值 1.000 | 多层电平相加超 1 | 人声/BGM/SFX 分别降档，重渲后复测 |
| 体积暴涨 8 倍 | 胶片颗粒拉高熵 | 降 `opacity` 或减 `numOctaves`，或干脆去掉颗粒 |
