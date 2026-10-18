<div align="center">

# rlm.pi

**面向 Pi 的递归语言模型（RLM）。**

用低成本模型高精度地读取超大上下文。

<p>
<a href="https://www.npmjs.com/package/@hicaru/pi-rlm"><img src="https://img.shields.io/npm/v/@hicaru/pi-rlm?color=cb3837&label=npm" alt="npm" /></a>
<a href="https://arxiv.org/abs/2512.24601"><img src="https://img.shields.io/badge/arXiv-2512.24601-b31b1b" alt="RLM paper" /></a>
<img src="https://img.shields.io/badge/license-MIT-blue" alt="MIT" />
</p>

<sub><a href="README.md">English</a> · <b>中文</b> · <a href="README.ru.md">Русский</a></sub>

</div>

## 安装

| | 安装 | 升级 | 卸载 |
|---|---|---|---|
| **Pi** | `pi install npm:@hicaru/pi-rlm` | 加 `--force` | `pi uninstall npm:@hicaru/pi-rlm` |
| **oh-my-pi** | `omp plugin install @hicaru/pi-rlm` | 加 `--force` | `omp plugin uninstall @hicaru/pi-rlm` |

1. 安装插件。
2. 输入 `/reload`，或重启 Pi。
3. 确认 **[Extensions]** 中有 `/rlm`、`/rlm-config` 和 `/rlm-stop`。
4. 按 `Ctrl+Shift+R` 或输入 `/rlm`。之后你的提示会交给 RLM 引擎处理。

## 工作原理

```
 你 ──► 提示 ──► RLM 引擎 ──► 主模型（在 repl() 中写 Python，每次一轮）
                    │            ├─► search / grep / outline   （不耗 token）
                    │            ├─► llm_query ──► 低成本 worker
                    │            └─► rlm_query ──► 子 RLM（深度上限）
                    └──► 状态 Σ + 校验 + 限额（token、时间、错误）
```

- 上下文保存在 Python 沙箱中。模型只看到索引。
- 主模型编写 Python 代码。代码找到模型需要的数据。
- 低成本 worker 模型负责阅读和总结。主模型不读取全文。
- 子 RLM 解决困难的子任务。它免费获得整个沙箱，不耗 token。
- 引擎保存运行状态 `Σ`，并校验每一次修改。
- `/compact` 生成会话摘要。它不调用 LLM。

## 基准测试

基准为 **OOLONG**（oolong-synth）：24 个任务，上下文最多 65,536 token，温度 0。

| 模型 | OOLONG | token/任务 | 成本/任务 |
|---|---|---|---|
| `zai/glm-5.3` | 待测 | — | — |
| `b-ai/qwen3.8-27b` | 待测 | — | — |
| `openrouter/openai/gpt-oss-120b` | 待测 | — | — |

模型列表见 [`bench/panel.json`](bench/panel.json)。每个模型有自己的结果目录：
`bench/runs/<model>/`。运行全部模型：

```bash
export ZAI_API_KEY=... BAI_API_KEY=... OPENROUTER_API_KEY=...
bun run bench --oolong-max-cl 65536 --oolong-limit 24
```

更多信息见 [`bench/README.md`](bench/README.md)。

## 命令

| 命令 | 快捷键 | 功能 |
|---|---|---|
| `/rlm` | `Ctrl+Shift+R` | 开启或关闭 RLM 模式 |
| `/rlm-stop` | | 停止当前运行 |
| `/rlm-config` | | 设置采样、预算和限额 |

## 沙箱 API

模型只能使用以下检索和委派函数：

```python
search("needle", k=10)        # 上下文中的 BM25 命中
grep_context(pattern, k=50)   # 带行号的正则命中
outline(path)                 # 文件中的定义
add_context("report.pdf")     # 添加 PDF、DOCX、XLSX、EPUB、CSV 或 HTML 文件
llm_query(prompt)             # 把文本发给低成本 worker
map_files(paths, prompt)      # 对多个文件提同一个问题
rlm_query(task, paths=...)    # 启动子 RLM
answer["content"] = "..."     # 写入最终答案
answer["ready"] = True        # 结束运行
```

## 安全

- 引擎不写磁盘。答案是一次运行的唯一输出。
- API 密钥不会进入沙箱。所有 sub-LLM 调用由宿主发出。
- Python 在受保护的子进程中运行，并带有保留名称列表。

## 许可证

[MIT](pi-plugin/rlm/LICENSE) © hicaru contributors
