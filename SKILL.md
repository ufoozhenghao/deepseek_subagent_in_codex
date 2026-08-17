---
name: codex-deepseek-subagent
description: 仅在用户要求安装、配置、选择或切换模型、检查、测试、修复、停用或卸载 Codex 的 DeepSeek 原生子 Agent 时使用；首次安装必须向用户确认 Responses-compatible HTTPS 端点、API Key、DeepSeek V4 Flash/V4 Pro 模型以及父/兜底模型（默认 Codex 官方 gpt-5.6-luna、max 思考）。普通 DeepSeek API 问题和已配置后的日常编码任务不要触发。
---

# Codex DeepSeek 子 Agent

本 Skill 只维护配置，不承接日常用户任务。确定性的文件、模型目录和凭据操作交给 `scripts/codex_deepseek.py`；不要手动改 TOML、JSON、Agent 文件或系统凭据库。

## 首次安装输入契约

首次安装或检测到没有本 Skill 管理记录时，必须在执行 `setup` 前向用户明确索取并确认以下四项输入；不得静默采用默认值：

1. **端点地址**：让用户选择 DeepSeek 官方端点 `https://api.deepseek.com/`，或提供一个自定义 Responses-compatible HTTPS base URL。自定义地址必须是 HTTPS，不能包含用户名、密码、查询参数或片段。
2. **API Key**：要求用户提供 DeepSeek API Key。不得回显、打印、写入命令参数、临时文件、配置文件、manifest 或 URL；只能通过 `--api-key-stdin` 的标准输入交给管理脚本，由系统凭据库保存。
3. **模型**：让用户明确选择 `deepseek-v4-flash`（更快、更省）或 `deepseek-v4-pro`（能力更强，适合复杂 Agent 任务）。不得静默替用户选择。
4. **父/兜底模型**：询问用户是否指定用于隔离验收和父任务兜底的 Codex 官方非 DeepSeek 模型；用户未指定时使用官方 `gpt-5.6-luna`，思考程度设为 `max`。用户明确指定其他官方 Codex 模型时，传入该模型，不得把它当作 DeepSeek Provider，也不得改写用户当前主模型。

如果用户没有同时确认这四项，先补齐缺失输入再执行配置。官方端点也必须由用户明确确认；不能因为脚本支持官方默认值就跳过端点询问。父/兜底模型可以接受默认值，但必须向用户展示默认值并允许首次配置时改选。

## 关键契约

- 只使用桌面应用内置的 Codex 运行时；版本仅用于诊断，兼容性以真实派发验收为准。
- 父/兜底模型默认使用 Codex 官方 `gpt-5.6-luna`，思考程度为 `max`；首次配置时用户可以指定其他官方非 DeepSeek 模型。已存在受管配置或用户明确选择时优先使用实际值，并由管理程序应用 v1 明文派发设置。技术原因见 [references/compatibility.md](references/compatibility.md)。
- 如果用户已在当前任务明确选择非 DeepSeek 父模型，但桌面的每任务选择未持久化到 `config.toml`，允许向管理程序传 `--parent-model`；不得自行猜测，首次配置未指定时才使用默认 Luna。
- 父模型变化后必须运行 `repair`，再重新验收。
- 支持 `deepseek-v4-flash` 与 `deepseek-v4-pro`。Flash 更快、更省；Pro 能力更强，适合复杂 Agent 任务。两者默认思考程度均为 `high`。
- 默认使用 DeepSeek 官方 API。只有用户明确提供自定义端点时才传 `--base-url`；必须是无凭据、无查询参数的 HTTPS Responses-compatible base URL。
- DeepSeek 是纯文本 Agent，不处理图片、视频、截图或其他视觉输入。父 Agent 先识别视觉内容，再传入文字事实。
- 日常任务只能直接调用：

  ```text
  spawn_agent(agent_type="DeepSeek", fork_turns="none", ...)
  ```

  不要为日常任务运行本 Skill、管理脚本或 `codex exec`。
- 当前工具若不认识 `DeepSeek` 角色，只提示用户打开新任务或重启 Codex；不得改用默认角色、脚本或 `codex exec` 代做当前任务。

## 触发后的流程

1. 运行 `status --json`，根据结构化状态继续，不靠文件名猜测。
2. 首次配置或 `manifest_exists` 为 `false` 时，先完成上面的端点、API Key、DeepSeek 模型和父/兜底模型四项输入确认。
3. 首次配置必须显式运行 `setup --model <DeepSeek模型> --base-url <用户确认的 HTTPS URL> --api-key-stdin --parent-model <Codex官方父模型> --json`；未指定父模型时传 `--parent-model gpt-5.6-luna`，并以 `max` 思考程度运行隔离验收。API Key 从标准输入传递；不要为父模型设置 DeepSeek Provider。
4. 已配置环境的切换模型或修复请求运行 `repair --model <模型> --json`；用户要求更换端点时追加 `--base-url <HTTPS URL>`。不传 `--model`/`--base-url`/`--parent-model` 的 `repair` 会保留当前值。
5. `setup`、`repair` 或 `test` 使用桌面内置运行时创建隔离验收会话。若返回 `new_task_required` 或 `restart_required`，提示用户重启桌面应用并打开新任务。
6. 验收必须检查子线程数据库 `threads` 表的实际元数据，并确认子 Agent 返回口令 `NATIVE_DEEPSEEK_OK`。实际 `model` 必须等于用户选择的模型，两者缺一不可。
7. 最终只汇报状态、实际 Provider、模型、思考程度、角色和备份位置；不要输出密钥或原始事件日志。

## 管理命令

入口。macOS 使用 `python3`，Windows 使用 `py -3`：

```text
python3 <skill-dir>/scripts/codex_deepseek.py <command> --json
```

- `status`：只读检查桌面内置运行时、配置、模型目录、凭据和客户端能力。
- `setup`：使用 `--model deepseek-v4-flash` 或 `--model deepseek-v4-pro` 写入配置并验收；可选 `--base-url`指定 Responses-compatible HTTPS 端点；未选择时返回 `model_selection_required`。
- `test`：通过桌面内置运行时执行一次直连测试和一次原生 `spawn_agent(agent_type="DeepSeek")` 验收。
- `repair`：按当前父模型重新应用配置并验收；传 `--model` 可切换模型，不传则保留当前模型。
- `disable`：停用本 Skill 创建的角色，保留 Provider、模型目录和凭据。
- `uninstall`：移除本 Skill 管理的配置；只有用户明确要求删除凭据时才传 `--remove-credential`。

默认使用当前 `CODEX_HOME`；仅在用户明确指定其他 Codex Home 时传 `--codex-home`。

## 状态处理

- `ready`：直连、原生路由、数据库元数据和返回口令均通过。
- `configured`：静态配置完整，但尚未完成实时验收。
- `credential_missing`：索要 API Key 后继续原流程。
- `model_selection_required`：向用户展示 Flash/Pro 选项，得到选择后继续原流程。
- `operation_in_progress`：已有配置操作正在运行，稍后重试，不并发修改。
- `conflict`：报告冲突文件和字段，等待用户决定是否替换。
- `unsupported`：报告缺少的系统能力，不按固定版本号猜测兼容性，也不手工绕过。
- `failed`：读取结构化 `errors`；若程序已回滚，明确说明，不再手改配置。

更详细的路径、版本和安全边界见 [references/compatibility.md](references/compatibility.md)。
