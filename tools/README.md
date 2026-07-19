# 코딩 에이전트 내장 도구(Built-in Tools) 분석

`reference/`에 클론한 각 오픈소스 코딩 에이전트가 **모델에게 어떤 도구를 어떻게 노출하는가**를 소스 레벨로 정리한 문서 모음. 턴루프 분석은 [`../trunloop/`](../trunloop/README.md) 참조.

| 에이전트 | 언어 | 도구 수 | 문서 |
|---|---|---|---|
| codex | Rust | ~30 (동적) | [codex.md](codex.md) |
| openclaude | TypeScript | **50+** | [openclaude.md](openclaude.md) |
| qwen-code | TypeScript | ~37 + CU 35 | [qwen-code.md](qwen-code.md) |
| opencode | TypeScript (Effect) | **16** | [opencode.md](opencode.md) |
| mistral-vibe | Python | 13 | [mistral-vibe.md](mistral-vibe.md) |
| zai-glm-cli | TypeScript | 9~10 | [zai-glm-cli.md](zai-glm-cli.md) |

> **[categories.md](categories.md) — 카테고리 × 턴루프 구조 분석**: 위 문서들이 *에이전트별* 나열이라면, 이 문서는 도구를 **12개 카테고리로 묶고 각 카테고리가 턴루프의 어느 지점에서 어떻게 작동하는지**를 정리한다. 도구가 루프와 맺는 관계를 5유형(순수 실행 / 게이트 통과 / 루프 제어 / 루프 정지 / 턴 생성)+메타(도구 세트 변경)로 분류.

---

## 1. 도구 추상화 비교

| 항목 | codex | openclaude | qwen-code | opencode | mistral-vibe | zai-glm-cli |
|---|---|---|---|---|---|---|
| 정의 형식 | 핸들러 + `*_spec.rs` 분리 | `Tool` 객체(20+ 속성) | `DeclarativeTool` 클래스 + `Kind` | `Tool.Def`(6필드) | `BaseTool` 제네릭 4종 | 스키마 리터럴 배열 |
| 정책 표현 | 등록 시 `ToolExposure` | **속성별 개별 선언** | **`Kind`에서 유도** | 없음(래퍼가 전역 처리) | 설정 필드(권한 중심) | **없음** |
| 병렬 안전성 | `tool_supports_parallel` | `isConcurrencySafe(input)` | `CONCURRENCY_SAFE_KINDS` | 명시 없음 | 없음(전부 동시) | 없음(전부 순차) |
| 결과 크기 제어 | — | `maxResultSizeChars`→디스크 | — | **래퍼가 전역 truncate**→`outputPath` | — | 없음 |
| 설명문 위치 | spec 코드 | 도구별 `prompt.ts` | 클래스 내 | **`.txt` 파일 분리** | **`prompts/*.md` 분리** | 스키마 리터럴 |
| 스키마 검증 실패 처리 | — | Zod | — | **`InvalidArgumentsError`→재작성 지시** | Pydantic | 없음 |

## 2. 권한/승인 모델

| 에이전트 | 모델 |
|---|---|
| **codex** | 도구가 세션의 `request_command_approval`/`request_patch_approval` 호출 → oneshot 블로킹. `ReviewDecision` 4값. 네트워크는 host 단위 amendment |
| **openclaude** | `canUseTool` 콜백 + PreToolUse 훅. **deny 규칙이 모델 노출 전 선제 필터**(`filterToolsByDenyRules`) |
| **qwen-code** | 스케줄러 L3→L4→L5 → `awaiting_approval`. `ToolConfirmationOutcome`에 **ModifyWithEditor**(수정 후 진행) 포함 |
| **opencode** | 도구가 `ctx.ask()` 직접 호출. doom-loop 반복 시 별도 확인. 서브에이전트는 부모 권한 병합 상속 |
| **mistral-vibe** | **도구 설정에 내장**: `permission`(ALWAYS/NEVER/ASK) + `allowlist` + `denylist` + **`sensitive_patterns`**(ALWAYS여도 ASK로 승격). `resolve_permission(args)`로 호출별 오버라이드 |
| **zai-glm-cli** | **루프 레벨 없음**. 파일/bash 도구 구현부에 `ConfirmationTool` 내장, 거부 처리는 프롬프트로 모델에 위임 |

가장 세분화된 것은 mistral-vibe(패턴 기반 승격)와 codex(host 단위 네트워크 amendment). 가장 선제적인 것은 openclaude(노출 단계 필터).

## 3. 도구 세트 교차 매핑

| 기능 | codex | openclaude | qwen-code | opencode | mistral-vibe | zai-glm-cli |
|---|---|---|---|---|---|---|
| 파일 읽기 | (shell로) | `Read` | `read_file` | `read` | `read_file` | `view_file` |
| 파일 쓰기 | (apply_patch) | `Write` | `write_file` | `write` | `write_file` | `create_file` |
| 편집 | `apply_patch` | `Edit` | `edit` | `edit`/`patch` | `edit` | `str_replace_editor`, `edit_file`(Morph), `batch_edit` |
| 패턴 탐색 | (shell로) | `Glob` | `glob` | `glob` | — | `search`(통합) |
| 내용 검색 | (shell로) | `Grep` | `grep_search` | `grep` | `grep` | `search`(통합) |
| 셸 | `exec_command`+`write_stdin` / `shell_command` | `Bash`, `PowerShell`, `REPL` | `run_shell_command` | `shell` | `bash` | `bash` |
| 서브에이전트 | `spawn_agent`+`wait_agent` 외 8종 | `Agent`(별칭 Task) | `agent`, `create_sub_session` | `task` | `task` | (TaskTool) |
| 할일 | `update_plan` | `TodoWrite`, `Task*` 4종 | `todo_write`, `task_*` 4종 | `todo` | `todo` | `create/update_todo_list` |
| 웹 | (없음, 호스티드 search) | `WebFetch`, `WebSearch`, `WebBrowser` | `web_fetch` | `fetch`, `search` | `web_fetch`, `web_search` | **없음** |
| 사용자 질의 | `request_user_input` | `AskUserQuestion` | `ask_user_question` | `question` | `ask_user_question` | 없음 |
| 스킬 | (주입 방식) | `Skill`, `DiscoverSkills` | `skill` | `skill` | `skill` | 없음 |
| 플랜 모드 | (mode) | `EnterPlanMode`/`ExitPlanMode` | `enter/exit_plan_mode` | `plan` | `exit_plan_mode` | 없음 |
| LSP | 없음 | `LSP` | `lsp` | `lsp`(실험) | 없음 | 없음 |
| 팀/스웜 | `spawn_agents_on_csv` 등 | `SendMessage`, `TeamCreate/Delete` | `team_*`, `send_message` | 없음 | 없음 | 없음 |
| 크론/스케줄 | 없음 | `Cron*` 3종, `Monitor`, `RemoteTrigger` | `cron_*` 3종, `loop_wakeup`, `monitor` | 없음 | 없음(세션 밖 `/loop`) | 없음 |
| 워크트리 | 없음 | `EnterWorktree`/`ExitWorktree` | `enter/exit_worktree` | 없음 | 없음 | 없음 |
| 노트북 | 없음 | `NotebookEdit` | `notebook_edit` | 없음 | 없음 | 없음 |
| 지연 도구 검색 | `tool_search` | `ToolSearch` | `tool_search` | 없음 | 없음 | 없음 |
| MCP | 리소스 3종 + 동적 | 리소스 2종 + 동적 | **풀/예산/재시도 분리** | MCP 서비스 | MCP 풀 | MCPManager |
| Computer Use | 없음 | 없음 | **`computer_use__*` 35종** | 없음 | 없음 | 없음 |

## 4. 특징적 설계 5가지

1. **codex — 도구가 "계획"이다.** 정적 레지스트리가 아니라 매 샘플링마다 feature 플래그·모델 능력·환경 유무로 `PlannedTools`를 재조립한다. `ToolExposure`가 모델 가시성과 디스패치 가능성을 분리해, 레거시 `shell_command`는 안 보이지만 과거 호출은 계속 처리된다. **파일 읽기/검색 전용 도구가 아예 없고 shell로 통합**한 것도 독특하다.

2. **openclaude — 메타 속성이 곧 정책.** `isReadOnly`/`isConcurrencySafe`/`isDestructive`/`interruptBehavior`/`maxResultSizeChars`/`shouldDefer`/`searchHint`가 스케줄링·UI·컨텍스트를 자동화한다. 결과가 크면 **디스크로 내리고 경로만 모델에 준다**. 도구 50+개의 대가로 `ToolSearch` 지연 로딩이 필수가 됐다.

3. **qwen-code — 분류에서 정책을 유도.** `Kind` 하나로 `MUTATOR_KINDS`·`CONCURRENCY_SAFE_KINDS`가 파생된다. `Kind.Think`를 병렬 안전에서 뺀 이유(save_memory·todo_write가 디스크에 씀)까지 주석에 남아 있다. `loop_wakeup`·`cron_*`로 **자가 턴 발생을 도구로 노출**한 유일한 구현.

4. **opencode — 정책을 도구 밖으로.** `Tool.Def`는 6필드뿐이고, 출력 절단·인자 검증·트레이싱을 `wrap()`이 전역 적용한다. 스키마 위반은 `InvalidArgumentsError`의 정형 문구로 모델에 되돌려 자가 수정을 유도한다. 도구 16개로 최소이며 팀/크론/워크트리가 통째로 없다.

5. **mistral-vibe — 권한이 설정의 1급 필드.** `sensitive_patterns`가 "bash는 허용하되 위험 패턴만 확인" 같은 정책을 선언적으로 표현한다. 도구가 **async generator**라 스트리밍 진행 출력이 자연스럽고, `selection_priority`로 같은 이름의 구현을 교체할 수 있다.

## 5. 관전 포인트

- **파일 도구를 둘 것인가, shell로 통합할 것인가** — codex는 통합(모델이 shell로 읽고 찾음), 나머지는 전용 Read/Glob/Grep을 둔다. 통합은 도구 수를 줄이지만 권한·병렬성·결과 크기 제어를 셸 안쪽으로 밀어 넣는다.
- **정책을 어디에 둘 것인가** — 도구 속성(openclaude), 분류(qwen-code), 설정(mistral-vibe), 래퍼(opencode), 등록 시점(codex), 없음(zai). 도구 수가 늘수록 선언적 정책의 가치가 커진다.
- **도구 수와 컨텍스트 비용의 트레이드오프** — 50+개를 가진 openclaude와 ~30개 codex, 37+35 qwen-code는 모두 **지연 로딩(ToolSearch)**에 도달했다. 16개 이하 구현들은 필요 없다.
- **자가 턴을 도구로 노출할 것인가** — qwen-code만 `loop_wakeup`/`cron_*`로 모델이 스스로 재기동을 예약하게 한다. 다른 구현은 훅/스케줄러 등 루프 바깥 계층에 둔다.
