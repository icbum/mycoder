# mycoder

[![Python](https://img.shields.io/badge/python-3.10+-blue)](https://python.org)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

mycoder 是一个我从claude code源码解析得到的极简、可读、可运行的中文 AI 编程 Agent。它保留了 AI 编程助手最核心的运行机制：命令行交互、模型调用、工具调用、文件读写、代码编辑、Shell 执行、上下文压缩和会话保存。

这个项目的目标不是把所有功能都堆进去，而是提供一个足够小、足够清晰、可以直接阅读和二次开发的 Agent 核心实现。

---

## 你能得到什么

mycoder 提供了一个最小但完整的 AI 编程 Agent 骨架：

| 能力 | 实现位置 | 说明 |
|---|---|---|
| 命令行交互 | `mycoder/cli.py` | REPL、单次 prompt、内置命令 |
| Agent 主循环 | `mycoder/agent.py` | 用户输入 → 模型 → 工具调用 → 工具结果 → 最终回答 |
| 模型调用 | `mycoder/llm.py` | OpenAI 兼容接口，支持流式输出和重试 |
| 环境配置 | `mycoder/config.py` | 从环境变量或 `.env` 读取模型、API Key、Base URL |
| 上下文压缩 | `mycoder/context.py` | 控制长对话和大工具输出带来的上下文膨胀 |
| 会话保存 | `mycoder/session.py` | 保存和恢复历史会话 |
| 文件读取 | `mycoder/tools/read.py` | 读取指定文件内容 |
| 文件写入 | `mycoder/tools/write.py` | 写入或创建文件 |
| 文件编辑 | `mycoder/tools/edit.py` | 搜索替换编辑，并输出 diff |
| Shell 执行 | `mycoder/tools/bash.py` | 执行命令，并拦截部分危险命令 |
| 文件搜索 | `mycoder/tools/glob_tool.py` | 按 glob 模式查找文件 |
| 内容搜索 | `mycoder/tools/grep.py` | 按正则搜索文件内容 |
| 子 Agent | `mycoder/tools/agent.py` | 用隔离上下文处理子任务 |

---

## 安装

建议在项目目录内使用虚拟环境，避免影响系统 Python 环境：

```bash
cd /home/userroot/mycoder
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -U pip
python -m pip install -e .
```

如果需要运行测试：

```bash
python -m pip install -e ".[dev]"
python -m pytest tests/ -v
```

---

## 模型配置

mycoder 使用 OpenAI 兼容接口。可以通过环境变量配置，也可以在项目根目录创建 `.env` 文件。

`.env` 示例：

```bash
OPENAI_BASE_URL=http://183.214.120.234:10021/v1
OPENAI_API_KEY=your-api-key
MYCODER_MODEL=gpt-5.4-codex
```

也可以直接在 shell 中 export：

```bash
export OPENAI_BASE_URL=http://183.214.120.234:10021/v1
export OPENAI_API_KEY=your-api-key
export MYCODER_MODEL=gpt-5.4-codex
```

---

## 运行
这是在我自己的服务器的运行方法，可以根据自己的需求：
进入项目目录并启用虚拟环境：

```bash
cd /home/userroot/mycoder
source .venv/bin/activate
mycoder
```

也可以不激活虚拟环境，直接运行：

```bash
cd /home/userroot/mycoder
.venv/bin/mycoder
```

单次模式：

```bash
.venv/bin/mycoder -p "帮我解释这个项目的结构"
```

指定模型：

```bash
.venv/bin/mycoder -m gpt-5.4-codex
```

---

## 架构

项目核心结构：

```text
mycoder/
├── cli.py            命令行入口、REPL、内置命令
├── agent.py          Agent 循环、工具调用、并行工具执行
├── llm.py            LLM 客户端、流式输出、重试、LiteLLM 可选后端
├── context.py        上下文估算与压缩
├── session.py        会话保存与恢复
├── prompt.py         系统提示词
├── config.py         环境变量与 .env 配置
└── tools/
    ├── base.py       工具基类和 schema 生成
    ├── bash.py       Shell 执行与危险命令拦截
    ├── edit.py       搜索替换编辑与 diff 输出
    ├── read.py       文件读取
    ├── write.py      文件写入
    ├── glob_tool.py  文件搜索
    ├── grep.py       内容搜索
    └── agent.py      子 Agent 工具
```

核心流程：

```text
用户输入
  ↓
Agent 组装 system prompt 和历史消息
  ↓
调用 LLM
  ↓
如果 LLM 返回工具调用，则执行工具
  ↓
把工具结果追加回上下文
  ↓
继续调用 LLM
  ↓
直到 LLM 返回普通文本作为最终回答
```

---

## 内置命令

在交互模式中可以使用：

```text
/help           显示帮助
/reset          清空当前对话历史
/model          查看当前模型
/model <名称>   切换模型
/tokens         查看 token 用量和费用估算
/compact        手动压缩上下文
/diff           查看本次会话修改过的文件
/save           保存当前会话
/sessions       列出已保存会话
quit            退出
```

保存的会话位于：

```text
~/.mycoder/sessions
```

---

## 作为库使用

```python
from mycoder import Agent, LLM

llm = LLM(
    model="gpt-5.4-codex",
    api_key="your-api-key",
    base_url="http://183.214.120.234:10021/v1",
)

agent = Agent(llm=llm)
response = agent.chat("找出项目里所有 TODO 注释并列出来")
print(response)
```

---

## 添加自定义工具

工具只需要继承 `Tool`，声明名称、描述、参数 schema，并实现 `execute()`：

```python
from mycoder.tools.base import Tool

class HttpTool(Tool):
    name = "http"
    description = "请求一个 URL。"
    parameters = {
        "type": "object",
        "properties": {
            "url": {"type": "string"},
        },
        "required": ["url"],
    }

    def execute(self, url: str) -> str:
        import urllib.request
        return urllib.request.urlopen(url).read().decode()[:5000]
```

---

## 适合做什么

mycoder 适合：

- 学习 AI 编程 Agent 的最小实现
- 理解工具调用和 Agent loop
- 二次开发自己的命令行 AI 编程助手
- 实验不同模型的代码能力
- 在受控环境中快速搭建一个轻量 coding agent

---

## 不适合做什么

mycoder 不是完整商业级 IDE Agent。它默认不包含：

- MCP 插件系统
- 复杂权限系统
- 多工作区管理
- 图形界面
- 大规模团队协作功能
- 完整插件市场

这些功能可以在当前核心架构上继续扩展。

---

## License

MIT。
