# Workstream B：视频提取与转 Transcript Tools

负责人：徐翊宸 · 分支：`feat/media-transcript-tools`

本模块让 agent / 前端把**用户本地的视频或音频**转成带时间戳的 transcript，可选保存到资料库。所有长任务走后台 job，不阻塞 HTTP。

---

## 1. 运行依赖

真实转写需要两样东西（仅整合/部署机器需要，写前端不需要）：

```bash
# 1) Python 包（可选 extra）
uv sync --extra media          # 安装 faster-whisper

# 2) 系统二进制：ffmpeg / ffprobe，需在 PATH 中
#   macOS:  brew install ffmpeg
#   Ubuntu: sudo apt install ffmpeg
#   Windows: choco install ffmpeg  或官网下载后加入 PATH
```

跑测试**不需要** ffmpeg / whisper：测试已在边界 mock 掉外部调用。

```bash
uv run pytest tests/test_media.py -q     # 17 passed
```

---

## 2. HTTP 接口

所有接口前缀 `/api`，本机 `127.0.0.1:8765`。

### 2.1 探测媒体信息

```
POST /api/media/probe
{ "path": "~/SJTUFlowData/lecture.mp4" }
```

返回（ffprobe 解析）：

```json
{
  "path": "...", "filename": "lecture.mp4", "extension": ".mp4",
  "size_bytes": 12345678, "duration_seconds": 3560.0,
  "has_audio": true, "has_video": true,
  "audio_codec": "aac", "video_codec": "h264",
  "sample_rate": "44100", "channels": 2
}
```

### 2.2 提取音频（写操作，需确认）

```
POST /api/media/extract-audio
{ "path": "...", "out_dir": null, "sync": false }
```

提取为 mono 16kHz WAV（Whisper 友好），默认输出到 `~/SJTUFlowData/extracted/`。
`sync=false`（默认）返回 **job**；`sync=true` 同步返回结果。

### 2.3 转写（核心，返回 job）

```
POST /api/media/transcribe
{ "path": "...", "provider": "local-whisper", "language": "zh", "sync": false }
```

- `provider`：`local-whisper`（默认，用 faster-whisper）/ `openai-whisper`。
- `language`：可选语言提示，如 `"zh"` / `"en"`，留空自动检测。
- 非 WAV 输入会**自动先用 ffmpeg 抽音频**，再转写。
- 默认返回 **job**，前端轮询 `/api/jobs/{id}` 看进度；`sync=true` 适合小文件/测试。

job 成功后 `result` 形如：

```json
{
  "language": "zh", "source": "...", "provider": "local-whisper",
  "duration_seconds": 65.0,
  "text": "全文……",
  "segments": [
    { "start": 0.0, "end": 5.0, "text": "第一句" },
    { "start": 5.0, "end": 10.0, "text": "第二句" }
  ]
}
```

> 注意：转写结果**只在内存中**，不会自动保存。用户选择「保存到资料库」时再调 2.4；选择「只用于本轮会话」就把 `segments` 直接喂给会话。

### 2.4 保存 transcript（写操作，需确认）

```
POST /api/media/save-transcript
{
  "title": "Lecture 03",
  "source": "~/SJTUFlowData/lecture.mp4",
  "description": "可选摘要",
  "segments": [ { "start":0, "end":5, "text":"..." } ],
  "language": "zh",
  "overwrite": false
}
```

同时写两份到 `~/SJTUFlowData/transcripts/`：

- `<slug>.json`：权威记录，含 `metadata` + `segments`（start/end/text）。
- `<slug>.md`：带时间戳的可读版。

返回该 transcript 的 metadata（含 `path` / `markdown_path` / `segment_count`）。
之后即可通过现有 `GET /api/transcripts`（只列标题/说明）和 `GET /api/transcripts/{id}`（读全文）访问。

### 2.5 任务状态

```
GET /api/jobs/{job_id}      # 单个 job
GET /api/jobs               # 全部 job（按创建时间倒序）
```

job 结构：

```json
{
  "id": "...", "kind": "media.transcribe",
  "status": "pending|running|succeeded|failed",
  "progress": 0.0,            // 0.0 ~ 1.0
  "message": "Transcribing with local-whisper",
  "result": { ... },          // 成功时
  "error": "",                // 失败时
  "created_at": "...", "updated_at": "..."
}
```

**前端轮询建议**：发起 transcribe → 拿到 `job.id` → 每 1~2s 拉一次 `/api/jobs/{id}`，
用 `progress` 画进度条，`status` 进入 `succeeded`/`failed` 时停止。

---

## 3. Agent 工具（供模型调用）

| 工具 | 风险等级 | 说明 |
|---|---|---|
| `media.probe` | read | 探测时长/编码/流信息 |
| `media.extract_audio` | write（需确认） | 抽 mono 16kHz WAV |
| `media.transcribe` | read | 转写为 segments，**不保存** |
| `media.save_transcript` | write（需确认） | 存 JSON + Markdown |

模型典型流程：`probe` → `transcribe` → 给用户看 → 用户确认后 `save_transcript`。

---

## 4. 安全与约束（已实现）

- 只处理用户提供的**本地文件**；不绕过平台 DRM / 登录 / 验证码 / 权限。
- 所有读写经过 `Workspace` 路径校验，只能落在项目目录 / `data_dir` / `state_dir`。
- 写操作（提取音频、保存 transcript）`requires_confirmation=True`，并写审计日志。
- transcript **metadata-first**：列表接口不返回全文，全文走 `transcripts.read` / `GET /api/transcripts/{id}`。

---

## 5. 给整合同学的合并要点

新增文件：

- `apps/backend/src/sjtuflow/tools/media.py`
- `apps/backend/src/sjtuflow/services/jobs.py`
- `tests/test_media.py`
- `docs/workstream-b-media.md`（本文档）

改动文件（均为追加，不影响他人逻辑）：

- `tools/__init__.py`：注册 media 工具
- `services/local_app.py`：`media_*` / `get_job` / `list_jobs` 方法 + `JobManager` 实例
- `web/app.py`：media / jobs 路由 + 请求模型
- `tools/transcripts.py`：列表去重（JSON+MD 同名只显示一条，保留 JSON）
- `pyproject.toml`：可选依赖 `[media]`
