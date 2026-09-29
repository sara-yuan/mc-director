# MiniMax H3 导演台 Agent 入口

本目录是 MiniMax H3 导演台项目。任何 Agent（Codex、OpenClaw、DeepSeek Harness、Claude Code 等）进入本仓库时，先读本文件，再按需读取详细 skill。

## 定位项目

项目根目录有两种形态：源码版含 `启动导演台.bat`、`启动导演台.ps1`、`base\web_frontend\director.html`、`base\web_frontend\server.py`；编译交付版**没有** `base\web_frontend\` 目录和 `.bat`/`.ps1` 脚本，只有 `base\web_frontend_compiled\`（`server.cp312-win_amd64.pyd` + `director.html.enc`），启动入口是 `exe\墨川导演台-授权启动器\墨川导演台-授权启动器.exe`。

如果不知道项目在哪个盘符，请先搜索：

```powershell
foreach ($d in (Get-PSDrive -PSProvider FileSystem)) {
  where.exe /R "$($d.Root)" "server.cp312-win_amd64.pyd" 2>$null
  where.exe /R "$($d.Root)" "启动导演台.bat" 2>$null
}
```

找到后，把项目根目录记为 `$ROOT`。

## 关键文件

- 跨 harness 快速入口：`$ROOT\SKILL.md`
- 详细 Agent 指南：源码环境读 `$ROOT\.codex\skills\director-shot-filler\SKILL.md`；编译交付版读 `$ROOT\agent_files\SKILL.md`（自带 `h3_skills\`）。
- H3 官方 Prompt 硬规则（生成视频前必读）：源码版 `$ROOT\base\docs\h3_skills\SKILL.md` 第 6 节；编译交付版 `$ROOT\agent_files\h3_skills\SKILL.md` 第 6 节（两份内容一致）。
- 导演台中文提示词工程与官方映射：`$ROOT\base\docs\director_prompt_engineering.md`
- 前端：源码版 `$ROOT\base\web_frontend\director.html`；交付版 `$ROOT\base\web_frontend_compiled\director.html.enc`（AES 加密，运行时由后端解密下发；同目录另有 `director_logpopup.html.enc`）
- 后端：源码版 `$ROOT\base\web_frontend\server.py`；交付版 `$ROOT\base\web_frontend_compiled\server.cp312-win_amd64.pyd`（Cython 编译 + Ed25519 签名，配套 `server_pyd_manifest.json` / `server_pyd_manifest.sig`）
- 启动入口（交付版）：`$ROOT\exe\墨川导演台-授权启动器\墨川导演台-授权启动器.exe`（交付包不含 `.bat` 启动脚本）
- 页面：`http://127.0.0.1:8199/director`
- 更新：系统通知面板 → 「一键更新」（仅 Windows 本地安装；会自动停后端/ComfyUI、逐文件校验覆盖、重启启动器并自动续跑。云端部署没有这个按钮，走更新包覆盖）
- ComfyUI：`http://127.0.0.1:8188`
- Agent token：`$ROOT\agent_token.json`（请求头 `X-Agent-Token: <token>`）
- Agent OpenAPI：`$ROOT\openapi.json`（运行时也可 `GET /api/openapi.json`）
- 稳定 Agent API：所有 `/api/agent/v1/*` 端点都必须带 `X-Agent-Token` 或 `Authorization: Bearer`。
- 剧本新结构以 `shotCards`（多镜头段数组）为核心；Excel 导入按 `@资产名` 精确匹配现有资产。
- 资产自动关联：剧本正文、剧情、台词里出现与当前剧本资产库完全一致的角色/场景/道具名称时，后端按精确名称自动关联；Agent 写 JSON 仍应显式填写 `assets: ["@张三", "@场景"]`，不要依赖页面顶部手动添加。
- 上传资产：用 `POST /api/agent/v1/assets`（传 `name` / `type` / `scriptId` / `images[].path`）一次完成“复制文件 + 写名称 + 绑定剧本”；批量绑定多剧本用 `POST /api/agent/v1/assets/batch_associate`，改名用 `POST /api/agent/v1/assets/rename`。不要把图片 URL 直接写进分镜。详见 `SKILL.md` §3.8。
- 参考图不是首尾帧：资产参考图/参考视频只作为身份与画面锚点；只有分镜显式设置了首帧/尾帧或上一镜 22 帧/下一镜 22 帧时才进入关键帧链路。
- 如果只分发 `交付文档\agent_files`，该目录已自带 `.\h3_skills\`，不要再到 `$ROOT\base\docs\h3_skills\` 找缺失文件。

## 当前命名与模式

- 左侧“AI 导演”已改名为 **AI 编剧**。
- 顶部显存开关为 **内存卸载**：默认 `highvram`（纯 GPU）；勾选后由后端按 `lowvram` 启动 ComfyUI。
- 音频库分两组：`group="script"`（剧本组，供分镜做音色克隆/对口型/音效参考）与 `group="public"`（公用组，只用于成片背景乐音轨）。
- 音频资产新增 `usage`（`voice_clone` 音色克隆 / `lip_sync` 对口型 / `bgm` 背景乐 / `sfx` 音效）与 `scriptIds`（绑定剧本；音频库左侧分组单选一个剧本或公用组）。
- 分镜右列“参考音频”只显示剧本组音频并支持多选；成片音乐轨只显示公用组音频。
- AI 编剧可把 `@音频名` 写入分镜 `audioRefs`（多选或留空）；生成时按音频用途自动执行克隆/对口型/音效，背景乐不进分镜。
- 数据读写、生成、日志、模式切换都通过 `127.0.0.1:8199` 的 HTTP API 完成。
- 写剧本/分镜必须使用 `POST /api/agent/v1/scripts/apply` 或 `POST /api/agent/v1/ai/apply`；禁止用 `POST /api/agent/v1/state` 整份覆盖剧本数据，否则 `shotCards` 的机位/运镜/站位/动作等字段会大量缺失。完整 JSON 规范见 `SKILL.md` 第 4 节。
- Agent 生成剧本草稿优先使用 `POST /api/agent/v1/ai/draft`（非流式稳定），再调用 `POST /api/agent/v1/ai/apply`；`ai/chat` 仅用于简短讨论，不建议依赖它直接产出结构化草稿。


## 剧本模式：卡片式 / 六段式（写分镜前必须先问用户）

剧本有 `promptMode` 字段：`cards`（卡片式，默认）或 `six`（六段式）。

- **强制反问**：外部 Agent 在写第一个分镜之前，必须先问用户「这部用卡片式还是六段式？」。用户没有明确回答之前，只可以做剧情梗概、场次、资产清单等不依赖模式的前期工作，不要写 `shotCards`，也不要写 `h3PromptRaw`。
- **卡片式 `cards`**：只写 `shotCards`（每镜按时间轴拆镜头段，写满 `shotSize / camera / movement / staging / plot / action`，台词写 `dialogue`）。导演台会自动拼成 H3 官方英文提示词，参考图编号 `<Picture N>` 也由后端生成，Agent 不要手写。
- **六段式 `six`**：只写 `h3PromptRaw`（可另写 `h3PromptRawEn` / `h3PromptSource` / `h3PromptNotes`）。用户直接给 H3 官方全参考（Ref2VA）六段：`subject_definitions / summary / retention_analysis / detailed_description / overall_soundscape / non_diegetic_music`；没写的段由导演台按 `assets` 合并补全，写了就按用户的来。
- **不得擅自改模式**：对已有剧本 `apply` 时，draft 里不写 `promptMode` 就沿用原值（不会退回 `cards`）；用户没要求换模式就不要写 `promptMode`，也不要清空手写正文。
- **模式与字段配套**：同一份 draft 里 `cards` 与 `six` 不要混写；提交前先说明本次覆盖哪些字段、哪些沿用原值。
- **六段式必须绑定参考图**：H3 不认识人名，只认标签，图片按挂载顺序就是 `<Picture 1..N>`。`subject_definitions` 每一行都要写清身份来自哪张图，例如 `<Subject 1> 是张三，身份参考 <Picture 1>，只锁脸、发型、五官比例与衣着。`；只写「`<Subject 1>` 是张三」会退化成靠挂图顺序猜人，典型症状是「衣服像、脸不像 / 换人 / 双胞胎」。
- **六段式台词**：台词固定写在 `<d>[语种] 台词</d>`（如 `<d>[Chinese] 让开！</d>`）；说话人写在 `<d>` 外面，例如 `<Subject 1> (S1) says, <d>…</d>`。电话音 / 画外音 / 旁白必须写出发声源与稳定的 `(Sx)` 编号（如 `a serious male voice coming through the phone (S2) says: <d>…</d>`），漏写时 H3 会把这句台词派给画面里那个人。
- 两种模式的完整规范（含示例与翻译接口）见 H3 官方规则第 6.2 节：源码版 `$ROOT\base\docs\h3_skills\SKILL.md`；编译交付版 `$ROOT\agent_files\h3_skills\SKILL.md`（内容一致）。

## 本地 / 云端推理模式

- 导演台支持 `local` 与 `cloud` 两种推理位置，`local` 走本机 `127.0.0.1:8188`，`cloud` 走 SSH 隧道到已部署云端 ComfyUI。
- 执行视频生成、AI 制图、AI 配音前，先用 `GET /api/clouddeploy/mode` 判断可用性：`local_available`、`cloud_deployed`、`cloud_comfy_ready`、`cloud_tunnel_active`。
- 仅本地可用：`runtime="local"`；仅云端可用：`runtime="cloud"`；两者都可用：读取 state 的 `defaultRuntime`，用户明确指定某分镜/任务的 `runtime` 时以指定为准。
- 单镜视频、AI 制图、AI 配音都可在请求 body 中带 `"runtime":"local|cloud"`。
- 批量视频必须保证目标 `shots[].runtime` 已正确写入；可读全量 state 后只修改 `runtime` 再写回，但禁止用全量 state 写 `shotCards` 或剧本内容。
- 不要默认只连本地 8188；纯云端用户的 8188 可能不存在或未启动，错误地连本地会导致任务失败。

## 概念与命名澄清（Agent 必读）

- `shots` 数组长度 = 实际“分镜”数量。一个 `shot` 对应一个分镜；`shotCards` 是单个分镜内部的“镜头段/子镜头”，不是新分镜。
- 用户说“1 个分镜 + 多个镜头段”时，结果是 `len(shots) == 1`，多个 `shotCards` 都放在该分镜内；禁止把镜头段数量当成分镜数量。用户说“3 个分镜”时才是 3 个 `shots`。这不是写死一个分镜，最新明确指令优先。
- 目标分镜数以后端最新意图解析：从最近一条用户消息向前扫描，取用户最后一次明确表达的分镜数量；旧消息里的“12 到 16 个镜头”不会覆盖后来说的“只要一个分镜”。
- 云端生成回拉后的最终本地文件名规范：`<项目slug>_<分镜序号:05d>_<版本:05d>_.mp4`，统一版本号：首版 `项目名_00001_00001_.mp4`，后续 `项目名_00001_00002_.mp4`。云端 ComfyUI 首次输出可能带自己的保存计数，不要用末尾保存计数判断分镜序号。
- `/api/director/shot_versions` 按解析后的真实分镜序号归组“过往分镜”；因此分镜 2 的视频不会显示到分镜 1 下。
- 页面子窗口/浮窗点击外部区域不会自动关闭，必须点“保存/确认/取消”等明确按钮才会退出。
- 导航页 ComfyUI 网页托管默认已启用：`DIRECTOR_MANAGE_COMFY` 默认为 `1`。若显示“当前启动模式不支持网页托管”，说明当前启动脚本禁用了托管模式，应改用内嵌日志启动脚本或显式设置该环境变量。

## 工作原则

- 数据读写优先使用 `/api/director/state`，不要直接改 HTML 默认数据。
- 剧本与项目必须联动；提示词按固定八行顺序：负面、视频风格、场景环境、环境音效、镜头景别、角色、剧情、台词。
- MiniMax H3 没有负面词通道：AI 编剧与 Agent 必须让 `negative` 保持空字符串 `""`，不要写任何“禁止 / 不要 / 无 / 避免”负面内容；“无字幕/无旁白/无背景音乐”由 `audio` 只写环境声、无对白时 `dialogue.line` 留空、`non_diegetic_music: N/A` 实现。
- `plot`/`staging`/`action` 只能写可拍摄物理动作，禁止“谁都不肯、心想、决定、不甘、几乎无法分辨、旁白、解说”等叙述句；H3 本地版会把这类句子念成旁白，无对白镜头必须改写成身体部位 + 动作 + 对象 + 结果。
- 批量生成必须严格串行，禁止并发。
- 不要修改 ComfyUI 工作流节点，只填导演台数据。

详细字段、音频策略、首尾帧、导入导出、排障方法见 skill 文件。

如果系统打包成 exe，Agent 仍通过 `127.0.0.1:8199` 与 `127.0.0.1:8188` 的 HTTP 接口工作；exe 必须继续暴露这些接口，且 `SKILL.md`/`AGENTS.md`/`openapi.json` 应保留在 exe 外可读。
后端首次启动会自动生成/复用 `$ROOT\agent_token.json`；EXE 启动器写同一路径，确保 token 稳定、跨 harness 通用。

