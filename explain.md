# Fay 数字人框架 — 程序结构与作用说明

> 本文按文件夹分类，逐个介绍 Fay 项目中每个程序的作用。
> 基于对源码的完整梳理编写。

---

## 目录

- [数据流全景](#数据流全景)
- [顶层文件](#顶层文件)
- [core/ — 运行核心](#core--运行核心)
- [ai_module/ — 情绪分析](#ai_module--情绪分析)
- [asr/ — 语音识别](#asr--语音识别)
- [tts/ — 语音合成](#tts--语音合成)
- [llm/ — 大脑](#llm--大脑)
- [gui/ — Web 交互界面](#gui--web-交互界面)
- [faymcp/ — MCP 管理层](#faymcp--mcp-管理层)
- [mcp_servers/ — 示例 MCP 工具服务器](#mcp_servers--示例-mcp-工具服务器)
- [genagents/ — 仿生记忆代理](#genagents--仿生记忆代理)
- [simulation_engine/ — 模拟引擎](#simulation_engine--模拟引擎)
- [utils/ — 工具库](#utils--工具库)
- [scheduler/ / skills/ / scripts/](#scheduler--skills--scripts)
- [test/ — 测试与实验](#test--测试与实验)
- [docs/ — 文档](#docs--文档)
- [数据/资源目录](#数据资源目录)

---

## 数据流全景

整个项目是一条「语音输入 → 理解 → 输出」的数字人管道：

```
麦克风(recorder) → ASR(funasr/ali) → 文字 → StreamManager
  → FeiFei.on_interact → 查 QA(qa_service) → 查记忆(memory_service/genagents)
  → 大模型(llm/nlp_cognitive_stream + execution_manager，可调用 MCP 工具)
  → 流式切句 → TTS(tts模块，按情绪选音色) → 音频队列 + 动作(action_signal)
  → 数字人端播放(WebSocket 10002) / Web 端展示(WebSocket 10003 / Flask 5000 / GUI)
```

Fay 同时扮演两类角色：

- **消费方**：通过 `faymcp/` 连上各种外部 MCP 服务器，把它们的工具交给 LLM 调用（MCP 客户端）。
- **提供方**：通过 `faymcp/mcp_server.py` 把自己暴露为 MCP 服务器（SSE 传输），供 Claude Code 等外部 agent 调用回顾记忆、广播消息（MCP 服务端）。

---

## 顶层文件

| 文件 | 作用 |
|---|---|
| `main.py` | **程序总入口**。启动时依次拉起：WebSocket 数字人服务(10002)、Web UI 服务(10003)、阿里 ASR、Flask(5000)、MCP 管理服务(5010)，还有命令行监听线程；支持 `-config_center` 参数从远程配置中心加载配置 |
| `fay_booter.py` | **核心服务装配器**。创建 `FeiFei` 大脑、录音器、socket 桥接、自动播放服务，并启动 Fay 作为 MCP 服务器(SSE, 8765)。`start()`/`stop()` 供 main 和 GUI 调用 |
| `requirements.txt` | Python 依赖清单 |
| `system.conf` / `config.json` | 本地配置文件（人设、模型 API key、模块开关等）；本地与远程配置中心二选一 |
| `qa.csv` | 自定义问答库，命中后直接返回答案、不走大模型 |
| `verifier.json` | 验证相关配置 |
| `LICENSE` / `README.md` / `icon.png` / `favicon.ico` | 说明文档与图标 |

---

## core/ — 运行核心

程序的心脏就在这里，所有实时服务都在这个目录。

| 文件 | 作用 |
|---|---|
| `core/fay_core.py` | `FeiFei` 类，**大脑中枢**。入口 `on_interact()` 接收各种来源的交互；`say()` 走 TTS → 设置情绪/动作/口型 → 播放声音的完整输出链路；`__process_interact()` 先查 Q&A 再走大模型 |
| `core/recorder.py` | **音频采集**。动态 VAD 静音检测、唤醒词识别，采集到语音后分发给 ASR（阿里/FunASR） |
| `core/wsa_server.py` | **WebSocket 服务框架**（端口 10002 数字人、10003 Web UI）。`MyServer` 基类 + `HumanServer`/`WebServer`，支持多客户端、按用户名定向发消息、断线自动清理 |
| `core/interact.py` | `Interact` 数据类（极简）：`interleaver`（来源：mic/text/socket/auto_play/stream）+ `interact_type`（1=对话、2=透传）+ `data` 字典 |
| `core/stream_manager.py` | **文本流管理器**。每用户两个句子缓存流（主输出流 + NLP 流），监听线程持续消费边写边播；支持会话 ID 标记 `__<cid=..>__`、停止生成标志、按用户清理音频队列 |
| `core/socket_bridge_service.py` | **协议桥**（端口 9001）：把 WebSocket 客户端和 TCP 服务(10001)双向桥接，让 Web 数字人端能访问老的 TCP 音频通路 |
| `core/content_db.py` | 对话存储 `memory/fay.db` 的 `T_Msg` 表 + `T_Adopted` 采纳记录表：增删改查消息、采纳/取消采纳 |
| `core/member_db.py` | 用户管理 `memory/user_profiles.db` 的 `T_Member` 表：用户名、画像 `user_portrait`、补充信息 `extra_info` |
| `core/authorize_tb.py` | `T_Authorize` 表：存第三方 API 的 access token 及过期时间（目前用于百度情绪分析） |
| `core/qa_service.py` | **规则级问答**：人设问答（你叫什么/几岁/什么星座…）、指令问答（关/静音/换声…）走关键词相似度匹配；Q&A 记录还可回写 CSV |
| `core/action_signal.py` | 读 `config/action_rules.csv`，按关键词命中动作规则（拍视频时可让数字人执行点头/摇手等动作） |
| `core/memory_service.py` | **记忆统一入口**。把写记忆/检索记忆收拢成薄适配层：规范打 tag（`kind:/source:/domain:...`）、`remember/search/get_recent/get_active_rules/get_reflections/get_user_profile` 等函数，供 Fay 内部和 MCP 外部共用 |

`config/action_rules.csv` 是动作规则表，列名 `code/behavior/affect/intensity/keywords`。

---

## ai_module/ — 情绪分析

| 文件 | 作用 |
|---|---|
| `ai_module/baidu_emotion.py` | 调**百度情绪识别 API**，管理 app token（走 `authorize_tb` 缓存），返回积极/消极/中性分数，用于决定数字人表情 |
| `ai_module/nlp_cemotion.py` | 本地 Cemotion 模型情绪打分（需加载模型实例后调用 `predict`） |

---

## asr/ — 语音识别

| 文件 | 作用 |
|---|---|
| `asr/ali_nls.py` | 对接**阿里云智能语音(NLS)** 实时识别 |
| `asr/funasr.py` | 本地 **FunASR** 识别封装 |
| `asr/funasr/` | 完整的 FunASR 客户端/服务端套件：`ASR_server.py`(流式服务)、`ASR_client.py`、`funasr_client_api.py`(HTTP 封装)、`data/hotword.txt`（热词）、`requirments.txt` |

---

## tts/ — 语音合成

| 文件 | 作用 |
|---|---|
| `tts/ali_tss.py` | 阿里云 TTS |
| `tts/tts_voice.py` | **音色枚举表** `EnumVoice`：定义各音色的 voiceName、按情绪映射风格（angry/calm/cheerful…） |
| `tts/gptsovits.py` / `tts/gptsovits_v3.py` | **GPT-SoVITS** 语音克隆（v3 新版，音色更接近真人） |
| `tts/ms_tts_sdk.py` | 微软 Azure/Edge TTS（SDK 版） |
| `tts/volcano_tts.py` | 字节火山引擎 TTS |

---

## llm/ — 大脑

| 文件 | 作用 |
|---|---|
| `llm/nlp_cognitive_stream.py` | **核心大脑单文件**（3500+ 行）。`question()` 完整对话管道：构建人设 prompt → 检索仿生记忆 → 注入 MCP 资源和预启动工具结果 → 大小模型协作 → 边生成边切句流出；包含 agent 记忆、定时画像/反思任务、`get_mcp_tools()` 拉取启用的 MCP 工具转成 LangGraph tool spec |
| `llm/execution_manager.py` | **大小模型协作执行器**（LangGraph workflow）：`ExecutionManager`（每用户一线程）、`_big_model_execute(state)` 工具调用循环——小模型规划要不要调工具、大模型根据工具结果生成最终回答 |

---

## gui/ — Web 交互界面

### GUI 前端（Flask 端口 5000）

| 文件 | 作用 |
|---|---|
| `gui/flask_server.py` | **对外主 HTTP 服务**。`/v1/chat/completions`（模型 `fay`/`fay-streaming`/`llm`）、`/v1/embeddings`、`/api/send`、`/transparent-pass`（透传，MCP 广播就打到这）、`/to-greet`、`/to-wake`、`/to-stop-talking`、会员/消息管理、执行状态/取消/改写等几十组接口；也提供 Page2 网页聊天界面 |
| `gui/window.py` | PyQt 桌面窗口（`start_mode='common'` 时使用） |
| `gui/static/` | 前端资源：Vue+Element UI 三页界面（`index.html` 聊天、`setting.html` 设置、`script.js`/`index.js`/`setting.js`）以及机器人表情图 |
| `gui/templates/`、`gui/robot/` | 页面模板 + 数字人情绪动图（伤心/愤/Listening/Speaking…） |

---

## faymcp/ — MCP 管理层

### 服务与运行时（Flask 端口 5010）

| 文件 | 作用 |
|---|---|
| `faymcp/mcp_service.py` | **MCP 管理服务**（Flask 5010）：Server 增删改查、连接/断开/重启、工具开关、预启动配置；`connect_to_real_mcp()` 建连并发现工具，`call_mcp_tool()` 执行调用；`autostart:true` 的 Server 开机自动连 |
| `faymcp/mcp_client.py` | `McpClient`：真正的 MCP 客户端，支持 **stdio** 和 **sse** 两种传输，独立 asyncio 事件循环线程，`list_tools/list_resources/call_tool/disconnect`，60s 定期刷新工具缓存 |
| `faymcp/mcp_server.py` | **反向：Fay 自己作为 MCP Server** 暴露给外部（SSE，默认 8765）：工具 `broadcast_message`（把文本/音频透传进 Fay）、`memory_*` 记忆工具（让外部 agent 读写 Fay 的记忆），还能聚合当前已连接的 MCP 工具为 `id:tool` 命名空间对外 |
| `faymcp/tool_registry.py` | 进程内共享**工具注册表**：记录每个工具的启用/可用状态，供 LLM 拉取 |
| `faymcp/prestart_registry.py` | 预启动工具配置持久化（`data/mcp_prestart_tools.json`） |
| `faymcp/resource_registry.py` | MCP 资源的缓存注册表，资源文本可注入到 prompt |
| `faymcp/runtime_bridge.py` | LLM 进程内直接访问上述状态的桥（`mcp_runtime`），避开 HTTP 回环 |
| `faymcp/data/` | `mcp_servers.json`(已注册的 Server)、`mcp_tool_states.json`(工具开关)、`mcp_prestart_tools.json`(预启动配置) |
| `faymcp/templates/Page3.html` | **MCP 管理界面**（Vue）：添加/连接 Server、勾选工具、设预启动、查看资源 |

### MCP 工具接入的两条链路

1. **管理层面（工具从哪来）**
   注册 MCP Server 配置（stdio：`command/args/cwd/env`；sse：`ip + key`）到 `faymcp/data/mcp_servers.json` → `connect_to_real_mcp()` 用 `McpClient` 建连 → `list_tools()` 拉回工具清单 → `tool_registry.set_server_tools()` 存进共享注册表。

2. **消费层面（工具怎么被用）**
   `llm/nlp_cognitive_stream.py` 的 `question()` 里 `get_mcp_tools()` 从注册表取**已启用**的工具 → 转成 `WorkflowToolSpec` 挂进 LangGraph workflow → 规划器决定调用 → 执行器 `mcp_runtime.call_tool(server_id, name, args)` 回调真工具。

3. **预启动工具**：配置了 `prestart:true` 的工具会在 LLM 回答**之前**被自动调用一次，参数支持 `{{question}}` 占位符——典型场景是先查日程/知识库再回答。

---

## mcp_servers/ — 示例 MCP 工具服务器

| 目录 | 作用 |
|---|---|
| `mcp_servers/schedule_manager/` | **日程管理**：sqlite 存日程、定时提醒、自然语言解析（"明天下午提醒我开会"）、到点通过 Fay API 说话提醒；附独立 Web 管理页(5011)。`server.py` 是完整体现 MCP 三件套（list_tools/call_tool/stdio 主函数）的范例 |
| `mcp_servers/fay_player_knowledge/` | **课程知识库问答**：把 `fay_player_knowledge/` 下的课程知识（文本文档）做成 RAG 检索工具（已配 `autostart:true`） |
| `mcp_servers/logseq/` | 对接 **Logseq 笔记**：读写笔记图的工具，含 `sample_graph` 示例 |
| `mcp_servers/yueshen_rag/` | 越深 RAG：另一套私有知识检索工具 |
| `mcp_servers/window_capture/` | **窗口截图**工具，让 LLM 能截取屏幕/窗口内容 |
| 各自 `README.md` / `requirements.txt` | 说明与依赖 |

---

## genagents/ — 仿生记忆代理

| 文件 | 作用 |
|---|---|
| `genagents/genagents.py` | `GenerativeAgent` 类：加载/创建记忆流（`nodes.json`+`embeddings.json`），从配置实时读取数字人人设进入 `scratch`，是记忆系统的核心数据结构 |
| `genagents/modules/memory_stream.py` | `MemoryStream`：记忆节点增删、重要性打分 `generate_importance_score`、按语义+时间检索 |
| `genagents/modules/interaction.py` | 记忆对话交互逻辑 |
| `genagents/genagents_flask.py` | 决策面谈服务（端口 **5001**）：读取 `instruction.json` 指令，以网页访谈形式（`decision_interview.html`）收集信息，写入记忆 |
| `genagents/instruction.json`、`genagents/templates/` | 面谈指令 + 页面模板 |

---

## simulation_engine/ — 模拟引擎

| 文件 | 作用 |
|---|---|
| `simulation_engine/gpt_structure.py` | 封装 OpenAI 调用：`chat_safe_generate` 带失败兜底与超时控制、prompt 模板渲染、embedding |
| `simulation_engine/llm_json_parser.py` | 从模型输出里安全提取 JSON |
| `simulation_engine/global_methods.py` | 通用工具函数 |
| `simulation_engine/settings.py` | 从 `config_util` 取模型/API 配置并注入路径 |
| `simulation_engine/prompt_template/` | 记忆流 prompt 模板：重要性打分、反思生成、交互（ask/应答/陈述） |

`genagents` + `simulation_engine` 合起来就是著名的 **斯坦福生成式代理（Generative Agents）记忆范式**——观察 → 重要性打分 → 反思 → 影响后续回答。

---

## utils/ — 工具库

| 文件 | 作用 |
|---|---|
| `utils/config_util.py` | **配置中枢**：加载本地 `system.conf`+`config.json` 或远程配置中心；暴露 `ASR_mode`、`tts_module`、`gpt_*`、`big_model_*`、`embedding_api_*`、`start_mode`、`fay_url` 等全局配置 |
| `utils/util.py` | 日志、时间戳、音频文件处理等通用工具 |
| `utils/stream_sentence.py` | 句子切分与缓存（StreamManager 的底层依赖） |
| `utils/stream_state_manager.py` | 会话级流状态管理（session/版本对齐） |
| `utils/stream_text_processor.py` | 流式文本处理（think 标签、sentinel 标记等） |
| `utils/stream_util.py` | 流工具函数 |
| `utils/api_embedding_service.py` | 嵌入向量 API 封装（记忆检索用） |
| `utils/openai_api/` | **本地部署 OpenAI 兼容服务**：给自部署的 ChatGLM3-6B 等模型套上 `/v1/chat/completions`、`/v1/embeddings`、`/v1/models` 接口，让 Fay 用 OpenAI SDK 就能接本地模型（还含 langchain / zhipu 等变体） |

---

## scheduler/ / skills/ / scripts/

| 文件 | 作用 |
|---|---|
| `scheduler/thread_manager.py` | `MyThread`：带 `raise_exception()` 的线程类，可强制停止；`stopAll()` 全局停线程。全项目线程管理的基石 |
| `skills/remote_audio_key0.py` | **按键盘"0"键说话**的技能：按住 0 就采集麦克风音频发到 Fay，松开暂停 |
| `scripts/update_logo_version.py` | 用 Pillow 往 Logo 图上绘制/更新版本号（`python scripts/update_logo_version.py 3.12.4`） |

---

## test/ — 测试与实验

| 文件 | 作用 |
|---|---|
| `test/` 下各 `*test*.py` | 各种功能测试：录音、ASR、GPT 流式、WebSocket 客户端、远程音频(10001/9001)、自动播放、表情识别、RPA、langchain/react/rewoo 实验等 |
| `test/mcp_stdio_example.py` | MCP stdio 最小示例（被 `mcp_servers.json` 的 id=1 引用） |
| `test/easegen_auto_play_server.py` | 自动播放服务实验（弹窗显示图片） |
| `test/FunAudioLLM/SenseVoice/` | 阿里 **SenseVoice** 语音识别模型本地推理 |
| `test/ovr_lipsync/` | **OVRLipSync** 口型驱动实验：`ProcessWAV.exe` 从音频算口型参数，`test_olipsync.py` 调用；附带的 ffmpeg 用于音频处理 |

---

## docs/ — 文档

| 文件 | 主题 |
|---|---|
| `docs/MCP外部调用接口.md` | MCP 外部调用方式说明 |
| `docs/Fay数字人MCP知识库配置指南.md` | 用 MCP 接知识库的配置教程 |
| `docs/Prompt设计文档.md` | 大脑 prompt 设计 |
| `docs/memory_module.md` | 记忆模块说明 |
| `docs/Fay侧标准动作改造说明.md`、`docs/Live2D模型制作要求.md`（+简化版） | 数字人动作/模型制作规范 |
| 根目录 `readme/` | 项目宣传图 |

---

## 数据/资源目录

| 目录 | 内容 |
|---|---|
| `fay_player_knowledge/` | **课程知识库原始文本**（多个 zip：Fay 介绍/图文知识库/多用户分发/prestart 标签/大小模型协同等），由"MCP 课程知识库"工具检索 |
| `memory/` | 记忆数据：`fay.db`(结构归档)、`user_profiles.db`、`User/memory_stream/{nodes,embeddings}.json`(记忆流)、`chroma_db/`(向量库缓存) |
| `logs/` | 运行日志（`asr_result.txt`、`answer_result.txt`） |
| `cache_data/` | 临时音频/缓存 |
| `config/` | 动作规则表等配置文件 |
| `faymcp/robot` | MCP 界面用数字人表情图 |

---

## MCP 工具怎么接入（速查）

1. **写一个 MCP Server**：参考 `mcp_servers/schedule_manager/server.py`，实现 `@server.list_tools()` 声明工具、`@server.call_tool()` 实现调用，最后用 stdio 或 SSE 跑起来。
2. **注册**：在 Page3 界面或 `POST /api/mcp/servers` 添加配置（stdio：`command/args/cwd`；sse：`ip/key`），存到 `faymcp/data/mcp_servers.json`。
3. **连接发现**：触发连接后 `connect_to_real_mcp()` → `McpClient.connect()` → `list_tools()` → `tool_registry.set_server_tools()`。
4. **启用**：在工具列表勾选启用（或设为预启动工具）。
5. **LLM 调用**：`get_mcp_tools()` 取启用的工具 → 转 `WorkflowToolSpec` → 规划器决定 → `mcp_runtime.call_tool()` 回调对端。

---

*文档生成时间：2026-08-18*