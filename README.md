# Local Tool-Calling LLM Agent with Safety Guardrails — Project Showcase

A small, framework-free AI agent that runs entirely on your own machine. It talks to a local LLM through an OpenAI-compatible API (LM Studio today, vLLM on a GPU server tomorrow), uses tools to read and write files and to run code, checks its own work, and keeps a human in control of every action that changes something.

> This repository is a **project showcase**: it describes the design, the safety architecture and the test results of the agent. The source code is kept in a private repository. If you are evaluating me for a role and would like to review the code, contact me through [GitHub](https://github.com/AliBaki-Turkoz) or LinkedIn and I will grant read access.

| | |
|---|---|
| **Author** | Ali Baki Türköz · personal project, August 2026 |
| **Source code** | Private repository · available to recruiters and interviewers on request |
| **Language** | Python (~1,100 lines), one dependency: `openai` |
| **Models tested** | Qwen3 Coder 30B, Qwen3.6 35B A3B (tool-calling models served by LM Studio) |
| **Safety tests** | 13 / 13 passing (runs without a model) |
| **Interface** | Terminal (Turkish prompts and messages) |

## Why this project
Most agent demos give the model direct access to the machine. This one is built the other way round: **the model can only act through a small set of tools, and every tool goes through a safety layer.** No file is ever really deleted, file tools cannot leave a dedicated workspace folder, dangerous shell commands are blocked, and anything with side effects waits for a yes/no from the user.

## Architecture
```
  [ Terminal / phone via VPN ]              access layer
              │
              ▼
  ┌─────────────────────────────┐
  │  Agent  (this repository)   │           orchestration layer
  │  agent.py  → loop, reflection│
  │  tools.py  → tool functions  │
  │  safety.py → guardrails      │
  │  memory.py → saved sessions  │
  └──────────────┬──────────────┘
                 │  OpenAI-compatible Chat Completions API (tools, streaming)
                 ▼
  ┌─────────────────────────────┐
  │  Model server               │           model layer
  │  LM Studio  /  vLLM         │
  └─────────────────────────────┘
```
The agent only knows the OpenAI API standard, so switching from LM Studio on a Mac to vLLM on a Linux GPU server means changing **one line** (`BASE_URL` in `config.py`).

## How the agent works
1. **Agent loop (ReAct style).** The model reads the task, decides on a tool call, the agent runs the tool safely and returns the result, and the model continues until it gives a final answer. A hard limit of `MAX_STEPS = 12` stops runaway loops.
2. **Streaming.** Reasoning tokens (if the model emits them), the answer and tool calls are streamed live to the terminal.
3. **Self-review (reflection).** After any task that used tools, the agent asks the model once to check its own result and fix real errors with further tool calls (at most 4 extra steps). This runs on a **throw-away copy** of the conversation, so repeated "review OK" messages never pollute the saved history.
4. **Execution-based verification.** The system prompt requires the model to run the code it writes (`run_python`) and look at the real output instead of guessing.
5. **Session memory.** Each conversation is saved as JSON with a timestamp and topic. On start-up you can resume any earlier session with its full history.

### Tools
| Tool | What it does | Needs approval |
|---|---|---|
| `list_files` | Lists a folder inside the workspace | no |
| `read_file` | Reads a text file inside the workspace | no |
| `write_file` | Writes a text file; the old version is first backed up to `.trash` | yes |
| `move_to_trash` | "Deletes" by moving to `.trash` with a timestamp (always recoverable) | yes |
| `run_python` | Runs Python in a separate process, in the workspace, with a 30 s timeout | yes |
| `run_shell` | Runs a shell command after the forbidden-pattern check, 30 s timeout | yes |

Every tool call carries a one-sentence `reason` that is shown in the approval prompt, so you always see *why* the model wants to do something.

## Safety design (defense in depth)
| Layer | Mechanism | Where |
|---|---|---|
| 1. Workspace jail | All file paths are resolved with `Path.resolve()` and must stay inside `WORKSPACE_DIR`; `../` and absolute paths outside it are rejected | `safety.resolve_within_workspace` |
| 2. No real deletion | Delete = move to `.trash/<timestamp>__name`; overwrites back up the previous version first | `safety.safe_delete`, `tools.write_file` |
| 3. Forbidden commands | Regex block-list (`rm -rf`, `sudo`, `dd`, `mkfs`, fork bomb, `curl … \| sh`, …) rejected **without** asking | `config.FORBIDDEN_COMMAND_PATTERNS` |
| 4. Human in the loop | Side-effecting actions show the exact code/command and wait for `[e/H]` | `safety.request_approval` |
| 5. Audit log | Every action, its result (OK / REJECTED / FORBIDDEN / TIMEOUT) and a timestamp go to `.audit.log` | `safety.audit` |
| + Limits | `MAX_STEPS = 12` per task, `EXECUTION_TIMEOUT = 30 s` per run | `config.py` |

## Tests
The repository contains two test scripts:

| Script | What it checks | Needs a model? |
|---|---|---|
| Safety test suite | Path-traversal rejection (3 checks), forbidden commands (4), a normal command is allowed (1), delete only moves to trash with content intact (3), overwrite keeps a backup (2) | No; runs in a temporary folder |
| Tool-calling check | The served model calls a test tool with the correct name and arguments | Yes (LM Studio running) |

Last run of the safety suite: **13 passed, 0 failed**.

## Code organisation
| Module | Responsibility |
|---|---|
| `agent.py` | Agent loop, streaming, reflection step, interactive terminal session |
| `tools.py` | Tool functions and the JSON schemas sent to the model |
| `safety.py` | Workspace jail, trash, command block-list, approval prompt, audit log |
| `memory.py` | Session save / list / resume (JSON) |
| `config.py` | All settings (model server, workspace, approval policy, limits) |

## Remote access
To use the agent from a phone, connect both devices with a private VPN such as **Tailscale** and point `BASE_URL` at the server's VPN address. **Do not expose the agent or the model server to the internet with port forwarding.**

## Limitations
- **Code execution is not sandboxed.** The workspace jail applies to the file tools. `run_python` and `run_shell` start in the workspace folder but run with your user's permissions, so the code they run could reach other files. The approval prompt is the real safeguard here; read the code before you approve it. A container or VM sandbox is the natural next step.
- **The command block-list is a second line of defence, not a guarantee.** Regex patterns can be bypassed (for example by a full binary path or an interpreter call).
- **Single user, terminal only,** synchronous, with Turkish prompts and messages.
- Tests are plain scripts, not yet a `pytest` suite with a mocked model.

## Roadmap
pytest suite with a mock model client · container sandbox for code execution · retrieval (RAG) tool with source citations · evaluation set for tool-use accuracy · minimal web UI.

## Development note
Built with AI-assisted development. The architecture, the safety design and the decisions behind each guardrail are mine, and I can explain every part of the loop and each safety layer.

---
© 2026 Ali Baki Türköz. All rights reserved. This page describes original work; no part of the project may be reused without permission.
