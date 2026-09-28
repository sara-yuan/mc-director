---
name: mochuan-director-agent
description: 通过本地 HTTP Agent API（支持本地/云端推理）操作墨川导演台，完成剧本写入、项目创建、单镜/批量视频生成、AI 制图、状态查询和日志读取。适用于 Codex、OpenClaw、Claude Code、DeepSeek Harness 等任何能读文件并发 HTTP 请求的 Agent。
---

# 墨川导演台 Agent API

本文件让外部 Agent 不需要理解导演台页面，就能安全地写剧本、创建项目、单镜/批量生成视频、AI 制图和查询状态。

生成视频前，必须先读取并遵守 H3 官方 Prompt 硬规则：`$ROOT\agent_files\h3_skills\SKILL.md` 第 6 节（`$ROOT\base\docs\h3_skills\SKILL.md` 是同内容副本）。无配乐必须写 `non_diegetic_music: N/A`，不能写 “No background music, no melody, no score.” 这类规则描述；普通无音乐不是静音，`overall_soundscape` 仍写环境声。

## 0. 定位项目

项目根目录是编译交付版形态：**没有** `base\web_frontend\` 目录，也没有 `.bat`/`.ps1` 启动脚本；前端是 `base\web_frontend_compiled\director.html.enc`（AES 加密），后端是 `base\web_frontend_compiled\server.cp312-win_amd64.pyd`（Cython 编译 + 签名），启动入口是 `exe\墨川导演台-授权启动器\墨川导演台-授权启动器.exe`。

不要把盘符写死。新机器可这样搜索：

```powershell
foreach ($d in (Get-PSDrive -PSProvider FileSystem)) {
  where.exe /R "$($d.Root)" "server.cp312-win_amd64.pyd" 2>$null
}
```

找到后记为 `$ROOT`。

## 1. 固定端口与鉴权

- 后端固定监听：`http://127.0.0.1:8199`
- 导演台页面：`http://127.0.0.1:8199/director`
- 机器可读规范：`http://127.0.0.1:8199/api/openapi.json`
- 本地 token：`$ROOT\agent_token.json`

读取 token：

```powershell
$token = (Get-Content "$ROOT\agent_token.json" -Raw | ConvertFrom-Json).token
```

所有 `/api/agent/v1/*` 请求必须带：

```text
X-Agent-Token: <token>
```

也可以用 `Authorization: Bearer <token>`。

## 2. 稳定入口总览

| 能力 | 方法与路径 |
| --- | --- |
| 健康检查 | `GET /api/agent/v1/health` |
| Agent OpenAPI | `GET /api/agent/v1/openapi.json` |
| 授权状态 | `GET /api/agent/v1/auth/status` |
| 可用模型/管线 | `GET /api/agent/v1/models/pipelines` |
| 读取全量状态 | `GET /api/agent/v1/state` |
| 写入全量状态 | `POST /api/agent/v1/state` |
| 剧本列表 | `GET /api/agent/v1/scripts` |
| 剧本详情 | `GET /api/agent/v1/scripts/{script_id}` |
| 创建/覆盖剧本 | `POST /api/agent/v1/scripts/apply` |
| 导入剧本 | `POST /api/agent/v1/scripts/import` |
| 项目列表 | `GET /api/agent/v1/projects` |
| 项目详情 | `GET /api/agent/v1/projects/{project_id}` |
| 从剧本创建项目 | `POST /api/agent/v1/projects` |
| 单镜生成 | `POST /api/agent/v1/generate/single` |
| 批量生成 | `POST /api/agent/v1/generate/batch` |
| 生成状态 | `GET /api/agent/v1/generate/status?scope=single` |
| 停止生成 | `POST /api/agent/v1/generate/stop` |
| AI 编剧对话 | `POST /api/agent/v1/ai/chat` |
| AI 编剧生成草稿（推荐，非流式稳定） | `POST /api/agent/v1/ai/draft` |
| AI 编剧应用草稿 | `POST /api/agent/v1/ai/apply` |
| AI 制图状态 | `GET /api/agent/v1/image/status` |
| AI 制图 | `POST /api/agent/v1/image/generate` |
| 卸载制图模型 | `POST /api/agent/v1/image/unload` |
| 删除制图记录 | `DELETE /api/agent/v1/image/delete` |
| AI 配音生成 | `POST /api/agent/v1/tts/generate` |
| AI 配音状态 | `GET /api/agent/v1/tts/status` |
| 资产列表 | `GET /api/agent/v1/assets` |
| 日志 | `GET /api/agent/v1/logs?kind=comfyui` |
| 通知 | `GET /api/agent/v1/notifications` |


## 2.1 本地 / 云端推理模式（执行生成前必读）

导演台支持 `local`（本机 ComfyUI）与 `cloud`（SSH 隧道到云端 ComfyUI）两种推理位置。不要默认只调 `127.0.0.1:8188`。

先检测可用性：

```powershell
$m = Invoke-RestMethod -Headers $headers -Uri "http://127.0.0.1:8199/api/clouddeploy/mode"
```

关键字段：

- `local_available`：本地 ComfyUI 是否可用。
- `cloud_deployed`：云端是否已完成部署。
- `cloud_comfy_ready`：云端 ComfyUI 是否可推理。
- `cloud_tunnel_active`：云端隧道是否已建立。

三种情况：

1. 仅本地：`local_available=true` 且 `cloud_comfy_ready=false`，全部使用 `runtime="local"`。
2. 仅云端：`local_available=false` 且 `cloud_comfy_ready=true`，全部使用 `runtime="cloud"`，不要尝试本地 8188。
3. 本地和云端都有：两者均可用时，先读 `GET /api/agent/v1/state` 的 `defaultRuntime`；如果用户对某个分镜/任务明确指定了 `runtime`，以用户指定为准。两者都没有才报告“本地/云端 ComfyUI 均不可用”。

设置方式：

- 单镜视频：`POST /api/agent/v1/generate/single`，body 可加 `"runtime":"local|cloud"`。
- 批量视频：先读全量 state，只改目标 `shots[].runtime`（必要时改 `defaultRuntime`），原样写回 `POST /api/agent/v1/state`，再调用批量接口。禁止用 state 写 `shotCards` 或剧本字段。
- AI 制图：`POST /api/agent/v1/image/generate`，body 可加 `"runtime":"local|cloud"`。
- AI 配音：`POST /api/agent/v1/tts/generate`，body 可加 `"runtime":"local|cloud"`。

批量生成仍按项目时间轴串行；本地与云端队列由后端调度，Agent 侧不要重复并发提交同一批分镜。

## 2.2 概念与命名澄清（Agent 必读）

- `shots` 数组长度 = 实际“分镜”数量。一个 `shot` 对应一个分镜；`shotCards` 是单个分镜内部的“镜头段/子镜头”，不是新分镜。
- 用户说“1 个分镜 + 多个镜头段”时，必须生成 `len(shots) == 1`，并把多个 `shotCards` 放在该分镜内；禁止把镜头段数量当成分镜数量。用户说“3 个分镜”时才是 3 个 `shots`。这不是写死一个分镜，最新明确指令优先。
- 目标分镜数解析：后端会从最近一条用户消息向前扫描，取用户最后一次明确表达的分镜数量。旧消息里早先的“12 到 16 个镜头”不会覆盖后来说的“只要一个分镜/单分镜/只用一个分镜”。
- 云端生成回拉后的最终本地文件名规范：`<项目slug>_<分镜序号:05d>_<版本:05d>_.mp4`。首版 `印度壮汉决斗_00001_00001_.mp4`，后续 `印度壮汉决斗_00001_00002_.mp4`。云端 ComfyUI 首次输出可能带自己的保存计数（如 `印度壮汉决斗_00001_00001_.mp4`），不要用末尾保存计数判断分镜序号。
- `/api/director/shot_versions` 按解析后的真实分镜序号归组“过往分镜”，不是取文件名末尾任意数字；因此分镜 2 的视频不会出现在分镜 1 下。
- 页面子窗口/浮窗点击外部区域不会自动关闭，必须点“保存/确认/取消”等明确按钮才会退出。
- 导航页的 ComfyUI 网页托管默认已启用：`DIRECTOR_MANAGE_COMFY` 默认为 `1`。若显示“当前启动模式不支持网页托管”，说明当前启动脚本禁用了托管模式，应改用内嵌日志启动脚本或显式设置该环境变量。
- 资产自动关联：剧本正文、剧情、台词里出现与当前剧本资产库完全一致的角色/场景/道具名称时，后端按精确名称自动关联；Agent 写 JSON 仍应显式填写 `assets: ["@张三", "@场景"]`，不要依赖页面顶部手动添加。
- 参考图不是首尾帧：资产参考图/参考视频只作为身份与画面锚点；只有分镜显式设置了首帧/尾帧或上一镜 22 帧/下一镜 22 帧时才进入关键帧链路。
- 如果只分发本文件而没有完整项目目录，需同时带上 `base\docs\h3_skills\` 及其 `references\`（`交付文档\agent_files\h3_skills\` 是同一份镜像副本，必须与它逐字节一致；不要保留任何旧版副本，否则外置 Agent 可能读到过期规则），否则本文件引用的 H3 硬规则会被视为缺失/截断。

## 3. 最小工作流

### 3.1 健康检查

```powershell
$headers = @{ "X-Agent-Token" = $token }
Invoke-RestMethod -Headers $headers -Uri "http://127.0.0.1:8199/api/agent/v1/health"
```

### 3.2 写剧本

推荐先让 AI 编剧通过非流式草稿端点生成草稿，再应用：

```powershell
$body = @{
  messages = @(
    @{ role = "user"; content = "写一个5分钟港风武打剧本，20个分镜，每个15秒，每个分镜拆成多个镜头段" }
  )
} | ConvertTo-Json

$draftResponse = Invoke-RestMethod -Method Post -Headers $headers -ContentType "application/json" -Body $body -Uri "http://127.0.0.1:8199/api/agent/v1/ai/draft"
$draft = $draftResponse.draft
```

然后把 AI 返回的草稿对象写入：

```powershell
$body = @{
  draft = $draft
  scriptId = $scriptId
} | ConvertTo-Json -Depth 100

Invoke-RestMethod -Method Post -Headers $headers -ContentType "application/json" -Body $body -Uri "http://127.0.0.1:8199/api/agent/v1/scripts/apply"
```

### 3.3 创建项目

```powershell
$body = @{
  scriptId = $scriptId
  title = "我的项目"
  resolution = "720p"
  aspectRatio = "16:9 (Widescreen)"
  steps = 4
} | ConvertTo-Json

$project = Invoke-RestMethod -Method Post -Headers $headers -ContentType "application/json" -Body $body -Uri "http://127.0.0.1:8199/api/agent/v1/projects"
```

### 3.4 生成视频

单镜：

```powershell
$body = @{ shot_id = "sh_xxxx" } | ConvertTo-Json
Invoke-RestMethod -Method Post -Headers $headers -ContentType "application/json" -Body $body -Uri "http://127.0.0.1:8199/api/agent/v1/generate/single"
```

批量：

```powershell
$body = @{ projectId = "p_xxxx"; shotIds = @("sh_1", "sh_2") } | ConvertTo-Json -Depth 10
Invoke-RestMethod -Method Post -Headers $headers -ContentType "application/json" -Body $body -Uri "http://127.0.0.1:8199/api/agent/v1/generate/batch"
```

不传 `shotIds` 表示生成该项目全部未完成分镜。

### 3.5 查询状态

```powershell
Invoke-RestMethod -Headers $headers -Uri "http://127.0.0.1:8199/api/agent/v1/generate/status?scope=batch"
```

### 3.6 AI 制图

```powershell
$body = @{
  prompt = "一只机械鹤，雪夜，写实电影质感"
  width = 1024
  height = 1024
} | ConvertTo-Json

Invoke-RestMethod -Method Post -Headers $headers -ContentType "application/json" -Body $body -Uri "http://127.0.0.1:8199/api/agent/v1/image/generate"
```

### 3.7 AI 配音（IndexTTS 2.5）

语言支持 `ZH / EN / JA / ES / AR`；八种情绪键名固定为 `Happy/Angry/Sad/Fear/Hate/Love/Surprise/Neutral`，值建议 0~1；`durationFactor` 越大语速越快。

```powershell
$body = @{
  text = "你好，欢迎使用墨川导演台"
  referenceAudioUrl = "/director_uploads/your_ref_audio.wav"
  language = "ZH"
  durationFactor = 1.0
  emotions = @{ Happy = 0.2; Angry = 0; Sad = 0; Fear = 0; Hate = 0; Love = 0.1; Surprise = 0; Neutral = 0.7 }
} | ConvertTo-Json

Invoke-RestMethod -Method Post -Headers $headers -ContentType "application/json" -Body $body -Uri "http://127.0.0.1:8199/api/agent/v1/tts/generate"
Invoke-RestMethod -Headers $headers -Uri "http://127.0.0.1:8199/api/agent/v1/tts/status"
```


## 4. 剧本 JSON 强制规范（AI 编剧 / Agent 写剧本必读）

**写剧本禁止用 `POST /api/agent/v1/state` 整份覆盖。**

**剧本库中的历史剧本只能作为剧情参考，不能作为字段完整性的模板；旧剧本可能因为早期写入方式而缺少机位/运镜/站位/动作。写新剧本一律以本规范为准，所有 shotCard 关键字段必须写满。**
 必须使用 `POST /api/agent/v1/scripts/apply`（或让 AI 编剧先 `POST /api/agent/v1/ai/draft` 再 `POST /api/agent/v1/ai/apply`）。`/state` 只用于读取和修改非剧本数据，直接写全量 state 会绕过镜头段补全，导致机位、运镜、站位、动作等大量字段为空。

草稿结构必须是：

```json
{
  "title": "剧本标题",
  "genre": "类型",
  "desc": "一句话梗概",
  "world": "世界观",
  "conflict": "核心冲突",
  "summary": "故事摘要",
  "shots": [
    {
      "scene": "第1场 · 晨雾山路",
      "num": "1.1",
      "title": "阿毛回头等人",
      "type": "全景",
      "dur": 8,
      "assets": ["@流浪汉", "@阿毛"],
      "audioRefs": [],
      "negative": "",
      "style": "现代写实风格，贴近现实生活，不要影视效果",
      "sceneEnv": "清晨薄雾笼罩的荒野土路，两侧碎石枯草，远处山脊在雾中若隐若现，地面露水反光",
      "audio": "山风低沉呼啸、脚步踩碎石声、远处野鸟鸣叫",
      "plot": "流浪汉沿碎石土路缓步前行，阿毛在前方三四步小跑并回头等待。",
      "dialogue": "流浪汉说：阿毛，慢点。",
      "directorNote": "略低机位全景；固定镜头缓慢前推；主光为清晨冷光，辅光为天光，轮廓光勾边；色温偏冷灰蓝；色调低饱和青灰；剪辑叠化到下一镜中景。",
      "shotCards": [
        {
          "startSec": 0,
          "endSec": 3,
          "shotSize": "全景",
          "camera": "略低机位平拍",
          "movement": "缓慢推镜头",
          "staging": "流浪汉在前景偏左，阿毛在前方偏右，两人视线方向与机位形成纵深感",
          "plot": "流浪汉背着帆布包袱沿碎石路前行，阿毛在前面小跑，尾巴轻摆。",
          "action": "流浪汉脚步稳定，阿毛偶尔回头，耳朵竖起。",
          "audioRefs": [],
          "videoRef": "",
          "dialogue": {"speaker": "@流浪汉", "line": "阿毛，慢点。", "lang": "zh", "position": "during", "anchor": "阿毛回头", "voiceId": ""}
        }
      ]
    }
  ]
}
```

硬性规则：

- 每个 `shot` 必须有 `shotCards`，且所有卡片的时间段首尾相接、总和等于 `dur`，禁止空缺、重叠或只给一个空卡片。
- 单段不得超过 4 秒：8 秒至少拆 3 段，10-15 秒至少拆 4-5 段，20 秒至少拆 5-6 段。
- 每张 `shotCard` 的 `shotSize / camera / movement / staging / plot / action` 都必须非空；只有 `audioRefs / videoRef / dialogue.line / voiceId` 允许为空。
- `shotSize` 优先用：大远景、远景、全景、中景、近景、特写、大特写。
- `camera` 写机位高度与角度，如：平拍、俯拍、仰拍、低机位、高机位、过肩机位、侧面机位。
- `movement` 写运镜，如：固定、推、拉、摇、移、跟、升降、环绕、手持、甩镜、缓慢推进。
- `staging` 写站位角度与构图关系，不能只复制 `plot`。
- `plot` 只写画面事件；`action` 写外部可见的人物动作、表情、视线、肢体表演。
- `plot / staging / action` 禁止写“谁都不肯、心想、决定、觉得、似乎、仿佛、几乎无法分辨、咬牙坚持、不甘、旁白、解说”等小说式叙述句，必须写“谁 + 身体部位 + 动作 + 对象 + 结果”。
- `dialogue.speaker` 必须统一用 `@角色名`，与该镜 `assets` 一致；`dialogue.line` 没有台词就写空字符串；`position` 只能是 `before / during / after`；`anchor` 写对应动作锚点。
- `negative` 一律留空字符串 `""`，AI 编剧与 Agent 都不要写任何“禁止 / 不要 / 无 / 避免”负面内容；无字幕、无旁白、无背景音乐通过 `audio` 只写环境声、无对白时 `dialogue.line` 留空实现。
- `directorNote` 必须写全 6 项：机位、运镜、主光/辅光/轮廓光、色温明暗、色调参考、剪辑衔接。
- `assets` 只填该镜真正出现的角色/场景 `@名称`；`audioRefs` 有参考音色才填 `@音频名`，没有就留 `[]`。


### 4.1 两种提示词模式：卡片式 / 六段式（写剧本前必须先问用户）

剧本级字段 `promptMode`：`cards`（卡片式，默认）/ `six`（六段式）。

**强制反问**：Agent 在写第一个分镜之前，必须先问用户「这部用卡片式还是六段式？」；
用户没有明确回答之前，不要写 `shotCards`，也不要写 `h3PromptRaw`。

- `cards`：只写 `shotCards`（字段规范见上方 §4）；提示词与 `<Picture N>` 编号由导演台生成，不要手写。
- `six`：只写 `h3PromptRaw`（可选 `h3PromptRawEn` / `h3PromptSource` / `h3PromptNotes`）。
  用户直接给 H3 官方全参考六段：
  `subject_definitions / summary / retention_analysis / detailed_description / overall_soundscape / non_diegetic_music`；
  没写的段由导演台用该镜 `assets` 自动补全（`<Subject N>` 定义、`retention_analysis`、首行关键帧指令、task-type 前缀）。
- **不得擅自改模式**：已有剧本的 `promptMode` 只能由用户改；draft 不写 `promptMode` 即沿用原值（不会退回 `cards`）。
- **不要混写**：同一份 draft 里不要 `shotCards` 与 `h3PromptRaw` 并存；提交前说明覆盖/沿用哪些字段。
- **六段式必须绑定参考图**：`<Subject 1> 是张三，身份参考 <Picture 1>，只锁脸、发型、五官比例与衣着。`
  H3 只认 `<Picture N>` 的挂图顺序，不写绑定就会出现「衣服像、脸不像 / 换人 / 双胞胎」；`<Picture N>` 编号必须与实际挂图顺序一致。
- **六段式台词**：`<d>[Chinese] 台词</d>`；说话人写在 `<d>` 外面（`<Subject 1> (S1) says, <d>…</d>`）；
  电话音 / 画外音 / 旁白必须写发声源与稳定 `(Sx)`，同一通电话跨镜用同一个编号。
- 六段式完整规则、草稿示例与翻译接口见 `$ROOT\agent_files\h3_skills\SKILL.md` 第 6.2 节（`$ROOT\base\docs\h3_skills\SKILL.md` 同内容）。

## 5. 重要规则

- 不要直接改 `director.db`，通过 API 写状态。
- 剧本核心结构是 `shotCards`：每个分镜由多个镜头段组成，字段为 `startSec/endSec/shotSize/camera/movement/plot/staging/action/dialogue`。
- `plot`/`staging`/`action` 只能写镜头能拍到的物理动作：谁 + 身体部位 + 动作 + 对象 + 结果；禁止“谁都不肯/不愿/心想/决定/觉得/似乎/仿佛/几乎无法分辨/咬牙坚持/不甘/旁白/解说”等小说式叙述句。这些文字会被 MiniMax H3 本地版念成旁白；无台词镜头的动作/表演文字尤其要物理化。
- “无字幕 / 无旁白 / 无背景音乐”不靠 `negative` 实现：`audio` 只写真实环境声与拟音，无对白时 `dialogue.line` 留空，`plot` 不写叙述句；后端会生成 `non_diegetic_music: N/A` 与“仅非语言环境声”的 `overall_soundscape`。
- 批量生成必须串行，不要并发提交多个分镜。
- 生成中不要重启服务；如服务已重启，先查状态再决定是否重新提交。
- 有 `shotCards` 时以镜头段为准；旧八行提示词只作为无 `shotCards` 时的回退。
- 新装或 exe 启动后，服务应监听 `127.0.0.1:8199`；如果端口不通，看 `$ROOT\director_launcher.log` 与 `$ROOT\base\web_frontend_compiled\_app_update.log`。

更多数据模型、音频、首尾帧、导入导出与排障：见 `$ROOT\agent_files\SKILL.md`。

