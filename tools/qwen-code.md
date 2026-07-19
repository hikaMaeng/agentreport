# qwen-code (QwenLM/qwen-code) 내장 도구 분석

gemini-cli 계보. 도구가 **`Kind` 열거형으로 분류**되고 그 분류가 병렬성·부작용 판정을 자동으로 결정하는 것이 특징. 도구명이 단일 상수 파일에 집중돼 있어 가장 파악하기 쉽다.

## 0. 소스 맵

| 역할 | 위치 |
|---|---|
| 도구명 상수 | `packages/core/src/tools/tool-names.ts` — `ToolNames`(20), `ToolDisplayNames`(73), 마이그레이션 맵(119) |
| 기반 클래스 | `packages/core/src/tools/tools.ts` — `DeclarativeTool`(206), `BaseToolInvocation`(90), `Kind`(936) |
| 분류 상수 | `tools.ts` — `MUTATOR_KINDS`(950), `CONCURRENCY_SAFE_KINDS`(962) |
| 레지스트리 | `packages/core/src/tools/tool-registry.ts` |
| 스케줄러/승인 | `packages/core/src/core/coreToolScheduler.ts` |
| 도구 구현 | `packages/core/src/tools/*.ts` |

## 1. Kind 기반 분류 — 정책의 자동 유도

각 도구는 `DeclarativeTool` 생성자에 `kind: Kind`를 선언한다(206~218). 이 하나의 값이 여러 정책을 파생시킨다:

```
Kind = Read | Edit | Delete | Move | Search | Execute | Think | Fetch | Agent | Other
MUTATOR_KINDS          = [Edit, Delete, Move, Execute]        ← 부작용 있음
CONCURRENCY_SAFE_KINDS = {Read, Search, Fetch}                ← 병렬 안전
```

`Kind.Think`가 병렬 안전에서 **의도적으로 제외**된 이유가 주석에 명시돼 있다 — `save_memory`, `todo_write` 같은 Think 도구가 디스크에 쓰기 때문(959~961). openclaude가 도구마다 `isConcurrencySafe()`를 개별 구현하는 것과 대조적으로, qwen-code는 **분류에서 정책을 유도**한다.

기타 선언 속성: `isOutputMarkdown`(기본 true), `canUpdateOutput`(스트리밍 출력 여부, 기본 false).

## 2. 내장 도구 목록 (`ToolNames`, tool-names.ts:20)

이름은 `snake_case`(모델 노출용)이고 `ToolDisplayNames`가 UI용 PascalCase를 따로 둔다.

### 파일
| 도구명 | 표시명 | 설명 |
|---|---|---|
| `read_file` | ReadFile | 파일 읽기 |
| `write_file` | WriteFile | 파일 쓰기 |
| `edit` | Edit | 편집 (레거시명 `replace`) |
| `notebook_edit` | NotebookEdit | Jupyter 셀 편집 |
| `list_directory` | ListFiles | 디렉토리 목록 (레거시 표시명 ReadFolder) |
| `glob` | Glob | 파일 패턴 (레거시 표시명 FindFiles) |
| `grep_search` | Grep | 내용 검색 (레거시명 `search_file_content`) |

`ripGrep.ts`가 grep 백엔드를 제공하며, `grepReadTracking.ts`·`priorReadEnforcement.ts`가 **"편집 전 읽기 강제"** 정책을 구현한다.

### 실행
| 도구명 | 설명 |
|---|---|
| `run_shell_command` | 셸 실행 (`shell.ts`, `pid-descendants.ts`로 자손 프로세스 추적) |

### 에이전트·태스크
| 도구명 | 설명 |
|---|---|
| `agent` | 서브에이전트 (레거시명 `task`) |
| `create_sub_session` | 하위 세션 생성 |
| `task_create` / `task_update` / `task_list` / `task_stop` | 태스크 관리 |
| `todo_write` | 할일 목록 (표시명 TodoList) |

### 팀
| 도구명 | 설명 |
|---|---|
| `team_create` / `team_delete` | 팀 생성·삭제 |
| `team_plan_approval` | **리더의 플랜 승인** — 워커가 PLAN 모드에서 대기하는 승인 계층(턴루프 문서 §10) |
| `send_message` | 동료 메시지 전송 |

### 플랜
`enter_plan_mode` / `exit_plan_mode`.

### 웹·외부
| 도구명 | 설명 |
|---|---|
| `web_fetch` | 페이지 가져오기 |
| `lsp` | 언어 서버 |
| `read_mcp_resource` | MCP 리소스 읽기 |

### 스케줄·자동화
| 도구명 | 설명 |
|---|---|
| `cron_create` / `cron_list` / `cron_delete` | 크론 관리 → 기계 발생 `Cron` 턴을 만듦 |
| `loop_wakeup` | **자기 재기동 예약** — 턴루프의 자가 턴 발생과 직결 |
| `monitor` | 조건 감시 |
| `workflow` | 워크플로 |

### 메타·컨텍스트
| 도구명 | 설명 |
|---|---|
| `tool_search` | 지연 도구 검색 |
| `skill` | 스킬 호출 (`skill-utils.ts`) |
| `save_memory` | 메모리 저장 (표시명 SaveMemory) |
| `structured_output` | 구조화 출력 |
| `artifact` / `record_artifact` | 산출물 기록 |
| `ask_user_question` | 사용자 질의 |

### 개발 환경
`enter_worktree` / `exit_worktree` — git worktree 격리.

### Computer Use
`computer_use__*` 접두사의 **35개 도구**가 `computer-use/schemas.ts`에 자동 생성되어 `computer-use/index.ts`로 등록된다. 주석에 따르면 cua-driver 버전마다 바뀌므로 `ToolNames`에 **의도적으로 열거하지 않는다**(57~62).

### MCP
`mcp-client.ts`, `mcp-tool.ts`, `mcp-client-manager.ts`, `mcp-transport-pool.ts`(연결 풀), `mcp-workspace-budget.ts`(워크스페이스별 예산), `mcp-retry.ts`, `mcp-discovery-timeout.ts`. MCP 인프라가 조사 대상 중 가장 정교하다(풀링·예산·재시도·타임아웃 분리).

## 3. 하위 호환 — 마이그레이션 맵

`ToolNamesMigration`(119)과 `ToolDisplayNamesMigration`(127)이 옛 이름을 새 이름으로 매핑한다: `search_file_content→grep_search`, `replace→edit`, `task→agent`, 표시명 `SearchFiles→Grep`, `FindFiles→Glob`, `ReadFolder→ListFiles`, `TodoWrite→TodoList`. 사용자 설정 파일의 옛 이름을 계속 지원하기 위함.

## 4. 승인 게이트

`CoreToolScheduler`의 L3→L4→L5 흐름(coreToolScheduler.ts:2312~)이 도구별 승인을 결정한다:
- `evaluatePermissionFlow` → `finalPermission ∈ {allow, deny, ask}`.
- `ask` → 상태 `awaiting_approval`(433) → `handleConfirmationResponse` 대기.
- `ToolConfirmationOutcome`: `ProceedOnce / ProceedAlways / ModifyWithEditor / RestorePrevious / Cancel`(930~933) — **에디터로 수정 후 진행**이라는 선택지가 독특하다.
- `ApprovalMode`: DEFAULT / AUTO / PLAN.

**`FS_PATH_TOOL_NAMES` 등록 주의사항**이 tool-names.ts 상단 주석에 명시(12~18): 파일 경로를 인자로 받는 도구는 `coreToolScheduler.ts`의 해당 목록에도 등록해야 조건부 규칙·경로 기반 스킬 활성화가 동작한다. **컴파일 타임 보호가 없어 누락 시 조용히 스킵**된다는 경고와 함께 TODO(선언부 `pathFields` 어노테이션으로 대체)가 달려 있다.

## 5. 특징 요약

- **`Kind` 분류가 정책을 유도**한다(병렬성·부작용). 도구마다 플래그를 개별 구현하는 openclaude와 대비되는 설계.
- 도구명이 단일 파일에 집중되고 **레거시 이름 마이그레이션 맵**을 명시적으로 유지.
- `loop_wakeup`·`cron_*`가 자가 턴 발생 장치를 **도구로 노출**한다(모델이 스스로 재기동을 예약).
- Computer Use 35개 도구를 스키마 자동 생성으로 편입하고 상수 열거를 일부러 피한 실용적 판단.
- MCP 인프라가 풀링/예산/재시도/타임아웃으로 세분화돼 가장 성숙하다.
