# 大昀精选视频 Skill

将脚本、口播、录屏或旧视频制作成信息清楚、语义连贯、手机上容易读懂的中文信息型视频。

Skill 名称：`dayun-featured-video`。适合流程讲解、精选风格重剪、解释性手绘插画、渐进字幕和音画同步；不用于纯真人口播清剪或只配音频。“精选”描述制作与验收标准，不保证平台推荐结果。

## 内容

- [SKILL.md](SKILL.md)：入口、制作约束、工作流程与默认审美。
- [制作蓝图](references/production-blueprint.md)：叙事、分镜、证据与关键词同步表。
- [GSAP 动效配方](references/gsap-recipes.md)：双页打开片头、重叠交接与确定性 seek。
- [手绘文字联动](references/illustration-text-sync.md)：预留文字区、独立文字层与真实口播同步。
- [验收表](references/qa-checklist.md)：画面、声音、编码、抽帧与人工观看。
- [agents/openai.yaml](agents/openai.yaml)：Codex 展示信息。
- [抽离记录](edit/project.md)：来源、独立化改动与验证边界。

## 安装到 Codex

需要 Git。仓库已公开，可直接克隆，无需 GitHub 身份认证。

当目标目录不存在时：

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
git clone https://github.com/cxcxy/dayun-featured-video.git   "${CODEX_HOME:-$HOME/.codex}/skills/dayun-featured-video"
```

如果已有同名目录或符号链接，先检查其来源；不要覆盖。也可克隆到其他位置，再将整个目录接入宿主支持的 Skill 目录。重新打开会话，确认可发现 `dayun-featured-video`。

更新独立克隆时，在仓库内运行 `git pull --ff-only`；若有本地修改或分叉，先处理差异。

## 使用

在支持 Skill 的 AI 编程助手中调用：

> 使用 $dayun-featured-video，把这份脚本和已确认口播制作成 1920×1080、30fps 的横版信息视频。按真实发音建立字幕和关键词时间锚点，交付源工程、MP4、字幕与验收记录。

也可以要求局部修改，并明确受影响时间段；本 Skill 会按实际改动范围验收。

## 制作前提

这是制作规范与动效参考包，未附带一键生成脚本、成片素材或完整视频工程。新制作默认使用 HyperFrames + GSAP，实际渲染需要 Node.js/npm、浏览器渲染环境、FFmpeg 与 ffprobe；沿用目标工程的版本、安装方式和构建入口。

动效配方中的 `mountScene`、`registerScene` 等为工程适配示意，须接入实际项目。外部 TTS、图片生成、字体、音乐及素材服务需自行配置并核对授权；本仓库不携带声音复刻、API Key 或个人素材。

入口和参考文档可独立读取，不要求克隆 DY-AI；可选的 HyperFrames、GSAP、图片和音频 Skill 由使用环境提供。

## 来源与使用边界

从 `cxcxy/DY-AI` 的 `skills/video/dayun-featured-video/` 抽离，保留四份参考文档及原界面元数据，补齐原仓库默认值与独立环境说明。原仓库中的 Skill 和本机安装入口保持原状，二者不会自动双向同步。

本次未新增开源许可证；对外分发前需另行确定许可。视频中的第三方素材、字体、音乐和声音的许可不随 Skill 一并授予。
