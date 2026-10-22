# 实用工具分享

## markitdown：各种文件转为markdown

[项目地址](https://github.com/microsoft/markitdown)

微软出的 **Python 小工具/库**，把各种文件转成 Markdown，**主要给 LLM 和大模型文本管道用**，不是给人看的精美排版工具。支持格式很广：

PDF、Word、PowerPoint、Excel、图片（EXIF + OCR）、音频（EXIF + 语音转写）、HTML、CSV/JSON/XML、ZIP（遍历内部）、EPub、YouTube 链接、Outlook 邮件等。

**前提**：需要 **Python 3.10–3.14**，建议装在虚拟环境里。它没有 GUI，就是个命令行 + Python 库。

### 安装

**1. 建虚拟环境**（强烈建议，避免污染全局）：

```powershell
python -m venv .venv
```

如遇到下载缓慢没有科学上网可使用如下命令：

```powershell
pip install "markitdown[all]" -i https://pypi.tuna.tsinghua.edu.cn/simple
```

> `[all]` 是装全部格式的依赖；只想要部分可以按需装，比如 `pip install "markitdown[pdf,docx,pptx]" -i ...`。

**可选依赖对照**（按需装，不用全装）：

| 选项                                               | 用途                     |
| -------------------------------------------------- | ------------------------ |
| `[all]`                                            | 全部                     |
| `[pdf]` / `[docx]` / `[pptx]` / `[xlsx]` / `[xls]` | 对应办公格式             |
| `[outlook]`                                        | Outlook 邮件             |
| `[audio-transcription]`                            | wav/mp3 语音转写         |
| `[youtube-transcription]`                          | YouTube 字幕抓取         |
| `[az-doc-intel]` / `[az-content-understanding]`    | Azure 云端增强（要付费） |

### 用法一：命令行（最常用）

```powershell
markitdown 文件.pdf > document.md
```

指定输出文件：

```powershell
markitdown 文件.pdf -o document.md
```

管道方式：

```powershell
Get-Content 文件.pdf -Raw | markitdown
```

### 用法二：Python 代码

```python
from markitdown import MarkItDown
```

**用大模型给图片/PPT 生成描述**（需要 OpenAI 等客户端）：

```python
from markitdown import MarkItDown
```

### 用法三：Docker

```powershell
docker build -t markitdown:latest .
```

### 几个要点

- **它输出是给机器读的**，格式够用但不保证"高保真排版"。要精美人工阅读的转换，它不是最优选择。
- **安全提醒（官方强调）**：它以当前进程权限做 I/O，`convert()` 能读本地文件、远程 URI、字节流。处理**不可信输入**时别直接丢给它，用更窄的接口：`convert_local()`（只读本地）、`convert_stream()`、`convert_response()`。
- **性能**：图片 OCR、音频转写依赖可选包，装 `[all]` 体积会比较大。
- **插件**：默认关闭，`markitdown --list-plugins` / `--use-plugins 文件.pdf` 启用。

## Bugzilla：提Bug平台
