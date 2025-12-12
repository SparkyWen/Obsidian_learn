# copilot本地部署的配置和使用

## 0. 背景 & 目标

- 我本地用 Ollama 下载了模型（例如 `gemma3:12b`、`qwen3-vl:8b`）。
- 但我把模型文件存放在 **E 盘：`E:\Ollama\models`**，不是默认的 `C:\Users\<user>\.ollama\models`。
- 目标：让 Obsidian 的 Copilot 插件能正常 **Test / Add Model** 并使用这些本地模型。

------

## 1. 核心结论（最重要的理解）

✅ Copilot 不是直接读取硬盘模型文件

Copilot 只能通过 **HTTP 请求** 访问一个正在运行的 Ollama 服务（默认 `127.0.0.1:11434`）。

因此：

- ✅ **Ollama 服务在跑**（`ollama serve` 或 Ollama App 后台服务）→ Copilot 才能添加/使用模型
- ❌ **Ollama 服务停止** → Copilot 会报 `Failed to fetch / Model verification failed`

------

## 2. 最终可用的环境变量配置（永久生效）

我在 Windows 环境变量（用户变量）里设置：

- `OLLAMA_HOST = http://127.0.0.1:11434`
- `OLLAMA_MODELS = E:\Ollama\models`
- `OLLAMA_ORIGINS = *`

> 解释：
>
> - **OLLAMA_MODELS** 决定 Ollama 从哪里读取/存储模型（你要永久用 E 盘就靠它）。
> - **OLLAMA_ORIGINS=\*** 是最稳的 CORS 设置，避免因为 `app://` / `file://` 等 scheme 导致 Ollama CORS 配置崩溃（panic）。
> - **OLLAMA_HOST** 固定监听地址和端口，Copilot 就稳定连接它。

### 命令行设置（永久）

```
setx OLLAMA_HOST "http://127.0.0.1:11434"
setx OLLAMA_MODELS "E:\Ollama\models"
setx OLLAMA_ORIGINS "*"
```

⚠️ 注意：`setx` 对**当前 PowerShell 窗口不会立刻生效**，需要：

- 关闭再打开 PowerShell / 或
- 重启 Ollama App / 或
- 重启电脑（最稳）

------

## 3. 临时让当前 PowerShell 会话立刻生效（用于当场验证）

当你不想重开终端时，直接在当前窗口：

```
$env:OLLAMA_HOST="http://127.0.0.1:11434"
$env:OLLAMA_MODELS="E:\Ollama\models"
$env:OLLAMA_ORIGINS="*"
```

------

## 4. 启动 Ollama 服务（必须）

### 方案 A：命令行启动（最直观）

```
ollama serve
```

只要你要用 Copilot，本机就必须有一个 Ollama 服务在跑。

### 方案 B：用 Ollama App 常驻后台（更省事）

- 打开 Ollama App，让它常驻托盘运行。
- **前提：App 必须读到你设置的环境变量**（有时需要重启 App 或重启系统）。

------

## 5. 快速自检（100%定位“服务/模型/路径”问题）

### 5.1 检查服务是否在线

```
curl.exe http://127.0.0.1:11434/api/tags
```

能返回 JSON（包含模型列表）= 服务在线。

### 5.2 检查模型是否真的可用（关键）

```
curl.exe http://127.0.0.1:11434/api/chat `
  -H "Content-Type: application/json" `
  -d "{""model"":""qwen3-vl:8b"",""messages"":[{""role"":""user"",""content"":""hi""}],""stream"":false}"
```

- ✅ 能返回回复 → 模型可用，Copilot 应该能通过 Test
- ❌ `model not found` → 几乎一定是 **OLLAMA_MODELS 指向错目录**（服务在 C 盘仓库，模型在 E 盘仓库）

### 5.3 识别端口冲突（常见于“我又手动 serve 了一次”）

如果出现：
 `bind: Only one usage of each socket address...`
 说明 11434 已经被占用（可能后台 Ollama App 已经开了）。

查占用进程：

```
netstat -ano | findstr :11434
tasklist /FI "PID eq <PID>"
```

------

## 6. Obsidian Copilot 中的正确填写方式（最终配置）

在 Copilot → Add Custom Chat Model：

- **Provider**：Ollama
- **Base URL**：`http://127.0.0.1:11434`（不填也行，但填上最明确）
- **Model Name**：必须严格等于 `ollama list` / `/api/tags` 里的名字
  - `gemma3:12b`
  - `qwen3-vl:8b`
- Vision 勾选：只给 `qwen3-vl:8b` 勾（如果你需要它做图像/多模态）

点击 **Test** → 看到 `Model verification successful!` 即成功。

------

## 7. 这次踩坑的关键原因回顾（帮助以后秒定位）

1. **模型存在 ≠ Copilot 能用**
    Copilot 只认 `http://127.0.0.1:11434` 这个服务是否在线。
2. **模型在 E 盘，但 Ollama 服务跑在 C 盘仓库**
    会出现 `/api/tags` 可能能列出，但 `/api/chat` 报 `model not found`（本质：服务端找不到 blobs）。
3. **CORS 配置写 app:// / file:// 导致 Ollama 崩溃（panic）**
    解决：`OLLAMA_ORIGINS="*"` 最稳。
4. `setx` 不会立即影响当前终端/已运行的 App
    需要重启终端 / 重启 App / 重启电脑。

------

## 8. 我的最终“稳定工作流”（推荐你以后照着做）

1. **永久设置环境变量**
   - `OLLAMA_MODELS=E:\Ollama\models`
   - `OLLAMA_ORIGINS=*`
   - `OLLAMA_HOST=http://127.0.0.1:11434`
2. 每次用 Obsidian 前确保 Ollama 服务在跑（二选一）：
   - 开 Ollama App 常驻托盘；或
   - PowerShell 运行 `ollama serve`
3. 若 Copilot 报错，先跑：
   - `curl http://127.0.0.1:11434/api/tags`
   - `curl ... /api/chat`
      就能立刻知道是“服务没开 / 路径错 / 端口冲突”。

