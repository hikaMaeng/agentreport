# codex (openai/codex) 내장 도구 분석

Rust. 도구가 **정적 목록이 아니라 매 샘플링마다 재구성되는 "계획(plan)"**이라는 점이 최대 특징. 모델·feature 플래그·환경 유무에 따라 노출 집합이 달라진다.

## 0. 소스 맵

| 역할 | 위치 |
|---|---|
| 도구 계획 수립 | `codex-rs/core/src/tools/spec_plan.rs` — `add_core_utility_tools`(702), `add_collaboration_tools`(786), `add_mcp_resource_tools`(693) |
| 라우터 | `codex-rs/core/src/tools/router.rs` — `ToolRouter`(35), `from_context`(60), `model_visible_specs`(75), `tool_supports_parallel`(99) |
| 레지스트리 | `codex-rs/core/src/tools/registry.rs` |
| 핸들러 구현 | `codex-rs/core/src/tools/handlers/*.rs` |
| 스펙(스키마) 정의 | `codex-rs/core/src/tools/handlers/*_spec.rs` |
| 병렬 실행 | `codex-rs/core/src/tools/parallel.rs` — `ToolCallRuntime` |

## 1. 구조: 핸들러 + 스펙 분리

각 도구는 **핸들러**(`XxxHandler`, 실행 로직)와 **스펙**(`*_spec.rs`, 모델에 보낼 JSON 스키마)으로 분리된다. 스펙은 `ToolSpec { name, description, parameters }` 형태이며 `name: "exec_command".to_string()`처럼 문자열로 선언된다(`shell_spec.rs:92`).

턴마다 `built_tools`(session/turn.rs:1228)가 `ToolRouter::from_context`를 호출해 `PlannedTools`를 조립한다. 조립 시 각 도구는 `add`(모델 노출+디스패치), `add_dispatch_only`(모델엔 숨기고 호출은 허용 — 레거시 shell), `add_with_exposure(ToolExposure::DirectModelOnly)`(code-mode 중첩 노출 제외) 중 하나로 등록된다.

## 2. 내장 도구 목록

### 실행 (shell 계열) — `shell_spec.rs`
모델·feature에 따라 **셋 중 하나의 모드**가 선택된다(`spec_plan.rs:653`):

| 도구 | 설명 |
|---|---|
| `exec_command` | UnifiedExec 모드의 주 실행 도구. 파라미터: `cmd, workdir, tty, yield_time_ms, max_output_tokens, shell, login, environment_id`. 세션 기반(장기 실행 프로세스 유지) |
| `write_stdin` | 실행 중 세션에 stdin 주입(`session_id, chars, yield_time_ms, max_output_tokens`). UnifiedExec 짝 도구 |
| `shell_command` | 레거시/기본 단발 실행. UnifiedExec 활성 시엔 `add_dispatch_only`로 숨겨진 채 유지 |
| `request_permissions` | 실행에 필요한 권한을 모델이 명시적으로 요청(`Feature::RequestPermissionsTool`) |

### 파일 편집
| 도구 | 설명 |
|---|---|
| `apply_patch` | 유일한 편집 도구. 전용 문법(`apply_patch.lark` 파서)으로 diff 적용. 모델의 `apply_patch_tool_type`이 있을 때만 노출 |

읽기 전용 파일 도구(Read/Glob/Grep)가 **별도로 없다** — 파일 읽기·검색은 shell 도구로 수행한다는 설계.

### 계획·컨텍스트
| 도구 | 설명 |
|---|---|
| `update_plan` | 할일/계획 갱신(`plan_spec.rs:43`) |
| `get_context_remaining` | 남은 컨텍스트 조회(`Feature::TokenBudget`) |
| `new_context` | 새 컨텍스트 윈도우 요청 = 압축 트리거(DirectModelOnly) |
| `view_image` | 이미지 파일을 컨텍스트에 첨부. `can_request_original_image_detail` 옵션 |
| `curr_time` / `sleep` | 현재 시각 조회 / 대기(`Feature::CurrentTimeReminder`, sleep은 별도 설정) |
| `wait_for_environment` | 환경 준비 대기(`Feature::DeferredExecutor`) |

### 사용자 상호작용
| 도구 | 설명 |
|---|---|
| `request_user_input` | 턴 중 사용자에게 질의(`experimental_request_user_input_enabled`, DirectModelOnly) |

### 멀티 에이전트 — `multi_agents_spec.rs` (v1/v2 두 세대 공존)
| 도구 | 설명 |
|---|---|
| `spawn_agent` | 자식 에이전트 생성(모델·reasoning·agent_type 오버라이드 옵션) |
| `wait_agent` | **자식 완료를 블로킹 대기**(status watch 채널 + deadline) — 턴루프 문서 §10 참조 |
| `send_input` / `send_message` | 자식에 입력/메시지 전달 |
| `followup_task` | 후속 작업 지시 |
| `resume_agent` / `interrupt_agent` / `close_agent` | 재개 / 중단 / 종료 |
| `list_agents` | 활성 에이전트 목록 |
| `spawn_agents_on_csv` / `report_agent_job_result` | CSV 기반 대량 잡 실행·결과 보고(`agent_jobs_spec.rs`) |

### MCP — `mcp_resource_spec.rs`, `mcp.rs`
| 도구 | 설명 |
|---|---|
| `list_mcp_resources` / `list_mcp_resource_templates` / `read_mcp_resource` | MCP 리소스 접근. MCP 서버가 하나라도 있을 때만 노출(693) |
| (동적) MCP 툴 | `McpHandler`가 서버 툴을 `mcp__{server}__{tool}` 또는 네임스페이스 형태로 편입 |

### 플러그인·검색
| 도구 | 설명 |
|---|---|
| `tool_search` | **지연 도구(deferred tools) 검색**. 모든 도구를 상시 노출하지 않고, 모델이 검색해 스키마를 가져오게 함(컨텍스트 절약) |
| `list_available_plugins_to_install` / `request_plugin_install` | 플러그인 추천·설치 요청 |

### 기타
`test_sync_tool`(모델 실험용), `extension_echo`(확장 API 데모), code-mode 전용 `CodeModeExecuteHandler`/`CodeModeWaitHandler`.

## 3. 노출 제어 — 3중 게이트

codex 도구 노출의 핵심은 **동적 필터링**이다:

1. **Feature 플래그**: `Feature::TokenBudget`, `RequestPermissionsTool`, `DeferredExecutor`, `CurrentTimeReminder`, `MultiAgentV2` 등이 각 도구 등록을 감싼다.
2. **모델 능력**: `model_info.apply_patch_tool_type`, `supports_search_tool`, `experimental_supported_tools`, `supports_parallel_tool_calls`가 노출을 좌우.
3. **환경 유무**: `tool_environment_mode`가 `has_environment()`가 아니면 shell/apply_patch/view_image 전체가 빠진다(원격/샌드박스 없는 세션).

추가로 `ToolExposure`가 **모델 가시성과 디스패치 가능성을 분리**한다 — 레거시 `shell_command`는 UnifiedExec 활성 시 모델에게 안 보이지만 과거 대화의 호출은 여전히 처리된다.

`tool_search`는 여기서 한 단계 더 나아가, 도구 스키마 자체를 지연 로딩해 컨텍스트 비용을 줄인다.

## 4. 권한/승인 연동

도구 실행 자체엔 승인 로직이 없고, **exec/patch 계열 핸들러가 세션의 승인 게이트를 호출**한다:
- `request_command_approval`(session/mod.rs:2168) — oneshot 채널 등록 → `ExecApprovalRequest` 이벤트 → `rx.await` 블로킹.
- `request_patch_approval`(2247) — `ApplyPatchApprovalRequest`.
- `ReviewDecision` = Approved / ApprovedForSession / Denied / Abort.
- 네트워크 접근은 `NetworkApprovalContext`로 host 단위 allow/deny amendment 제안.
- `PreToolUse` / `PermissionRequest` / `PostToolUse` 훅이 별도로 개입.

## 5. 병렬성

`ToolRouter::tool_supports_parallel`(router.rs:99)이 도구별 병렬 허용을 판정하고, `ToolCallRuntime`(parallel.rs)이 `FuturesOrdered`로 in-flight 실행을 관리한다. `tool_waits_for_runtime_cancellation`(105)은 취소 시 런타임 정리를 기다려야 하는 도구를 구분한다.

## 6. 특징 요약

- **도구 세트가 턴마다 재계산되는 "계획"**이다. 정적 레지스트리를 쓰는 다른 구현들과 근본적으로 다르다.
- **파일 읽기/검색 전용 도구가 없다** — shell로 통합. 대신 실행 도구가 세션 기반(`exec_command`+`write_stdin`)으로 정교하다.
- 멀티 에이전트 도구군이 조사 대상 중 가장 풍부하며(spawn/wait/send/resume/interrupt/close/list + CSV 잡), `wait_agent`가 블로킹 대기를 제공.
- `tool_search`로 도구 스키마를 지연 로딩하는 컨텍스트 최적화가 독보적.
