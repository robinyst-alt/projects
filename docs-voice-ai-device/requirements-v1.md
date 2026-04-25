# Voice AI Device - 基础版需求文档

> 整理时间：2026-04-22
> 状态：调试完成，API 可用，UI 待优化

---

## 一、系统架构

```
┌─────────────────────────────────────────────────────────┐
│                     UI 层 (Tkinter)                     │
│  - Hold-to-Talk 录音按钮                                 │
│  - 文字输入 + Send 按钮                                  │
│  - 聊天记录显示                                          │
│  - 状态指示器                                            │
└──────────────────┬──────────────────────────────────────┘
                   │ HTTP API
┌──────────────────▼──────────────────────────────────────┐
│                  API Server (FastAPI)                    │
│  端口：8000                                              │
│  - /health  ── Ollama 连接状态                           │
│  - /chat    ── AI 对话（phi3:3.8b via Ollama）            │
│  - /tts     ── 文字转语音（gTTS）                        │
│  - /stt     ── 语音转文字（whisper CLI）                 │
└─────────────────────────────────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                      ▼
  ┌──────────┐          ┌──────────┐
  │  Ollama  │          │  gTTS /  │
  │ phi3:3.8b│          │  whisper │
  └──────────┘          └──────────┘
```

---

## 二、API 端点

### GET /health
检查 Ollama 连接状态。

**响应：**
```json
{"status": "ok", "ollama": "connected"}
```

### POST /chat
发送文字，获取 AI 回复。

**请求：**
```json
{"message": "你好"}
```

**响应：**
```json
{"response": "AI 回复内容", "model": "phi3:3.8b"}
```

### POST /tts
文字转语音。

**请求：** `?text=Hello`（query parameter）

**响应：** MP3 音频文件

### POST /stt
语音转文字。

**请求：** `multipart/form-data`，字段名 `file`，支持 wav/mp3/webm

**响应：**
```json
{"text": "识别出的文字", "filename": "xxx.wav"}
```

---

## 三、环境依赖

### Python 环境
- Python 3.12
- venv：`voice-ai-device/.venv`

### 外部依赖
| 组件 | 版本 | 用途 | 安装方式 |
|------|------|------|---------|
| Ollama | 最新 | LLM 推理 | `ollama serve` |
| ffmpeg | 最新 | 音频格式转换 | `brew install ffmpeg` |
| whisper | base 模型 | 语音识别 | 手动下载模型（见下方） |

### 模型下载（STT 用）
```bash
mkdir -p ~/.cache/whisper
curl -k -x http://127.0.0.1:7897 -L "https://openaipublic.azureedge.net/main/whisper/models/ed3a0b6b1c0edf879ad9b11b1af5a0e6ab5db9205f891f668f8b0e6c6326e34e/base.pt" -o ~/.cache/whisper/base.pt
```

### Python 包
```
fastapi
uvicorn
httpx
edge-tts
openai-whisper
pyaudio
tkinter（内置）
```

---

## 四、启动方式

### 仅 API（服务器模式，推荐）
```bash
cd /Users/wongliliana/voice-ai-device
source .venv/bin/activate
python run.py --api
```

### 完整模式（API + UI）
```bash
cd /Users/wongliliana/voice-ai-device
source .venv/bin/activate
python run.py
```
> ⚠️ Mac 上 Tkinter UI 有 segfault 风险，建议用 API 模式

### 测试命令
```bash
# 健康检查
curl http://localhost:8000/health

# TTS 测试
curl -X POST 'http://localhost:8000/tts?text=hello' --output /tmp/test.mp3

# Chat 测试
curl -X POST http://localhost:8000/chat -H "Content-Type: application/json" -d '{"message": "hello"}'

# STT 测试（需要 wav/mp3 文件）
curl -X POST http://localhost:8000/stt -F "file=@/path/to/audio.wav"
```

---

## 五、已知问题

| 问题 | 原因 | 状态 |
|------|------|------|
| Tkinter UI seg fault | Mac 音频驱动冲突 | 待优化 |
| edge_tts 403 | WebSocket 被代理阻断 | 已换 gTTS |
| whisper 模型下载 SSL 失败 | 代理 SSL 拦截 | 已手动下载 |
| gTTS 中文 deprecated warning | zh-cn → zh-CN | 已知，不影响 |

---

## 六、待办

- [ ] 解决 Tkinter UI 在 Mac 上的稳定性问题
- [ ] UI 支持中文字符输入
- [ ] 录音超时保护
- [ ] 错误提示优化
- [ ] 考虑换用更轻量的 UI 框架（PyQt / web UI）
