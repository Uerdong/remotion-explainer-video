# remotion-explainer-video

用 Remotion 生产中文科普动画的一站式 Skill：从选题、解说文案、TTS 配音、SRT 字幕，到 BGM 节拍实测、画面切点吸附重拍、代码合成音效、tsx 工程搭建、静帧自检、成片渲染，再到封面与发布文案。

它把"卡点"这件事沉淀成可复现的方法：**音乐定拍 → 配音定长 → 吸附重拍 → 反写时间轴**，而不是凭感觉对齐。

## 目录

```
remotion-explainer-video/
├── SKILL.md                          # 主入口：十条铁律 + 十步流水线 + 冲突决策
└── references/
    ├── models.md                     # 每个环节适配的模型与选型理由
    ├── pipeline.md                   # 十步的命令、参数、验证方法
    ├── remotion-conventions.md       # 工程骨架、theme/components/timeline、常见坑
    └── restyle-playbook.md           # 已有成片换肤重制（白底扁平 → 暗色电影感）
```

## 安装

把 `remotion-explainer-video/` 整个目录拷到目标环境的 skill 目录即可：

```bash
cp -r remotion-explainer-video <workspace>/.user_skills/
```

无需注册，靠 `SKILL.md` 里的 `name` 与 `description` 自动触发。

## 触发场景

- 做一支科普 / 知识 / 原理讲解动画
- 用 Remotion 做横屏 MG 视频，按背景音乐重拍卡点做分镜
- 把解说词配成带字幕的成片（TTS + SRT + 内嵌字幕）
- 给已有成片换一套美术重制一版
- 为成片配封面、标题、简介

## 适配的模型

核心判断是"能确定性渲染的交给代码，只有真实感素材才交给生成模型"：

| 环节 | 首选 |
|---|---|
| 文案 / 分镜 | 文本大模型 |
| 配音 | 豆包语音合成（逐段独立出 wav） |
| BGM / 音效 | 代码合成（已知 BPM，可钉帧） |
| 画面本体 | Remotion（interpolate / spring） |
| 封面 / KV | seedream 5.0 flash（文字精细用 pro） |
| 真实感空镜 | seedance 2.0 fast / 2.5 |

细节与"不要做的事"见 [references/models.md](remotion-explainer-video/references/models.md)。

## 实战来源

本 Skill 提炼自两个完整项目：

- 《博弈论：囚徒困境》科普动画，1920×1080 / 30fps / 104 BPM / 53.65s / 六段结构，21 处画面切点全部吸附重拍
- 同片暗色电影感重制版：只换美术不换内容，帧级对齐旧版，留存双版本交付

## 许可

内容为通用方法论，可自由取用与修改。
