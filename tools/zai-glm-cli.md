# zai-glm-cli (guizmo-ai/zai-glm-cli) 내장 도구 분석

grok-cli 포크. 도구가 **OpenAI function-calling 스키마 리터럴 배열**로 하드코딩되고, 실행은 거대한 `switch` 문으로 분기하는 **가장 단순한 구조**.

> Z.ai(Zhipu) 공식 조직에는 공개 CLI 에이전트 레포가 없어, 커뮤니티 구현체를 대상으로 한다.

## 0. 소스 맵

| 역할 | 위치 |
|---|---|
| 도구 스키마 정의 | `src/zai/tools.ts` — `BASE_ZAI_TOOLS`(6), `buildZaiTools`(618), `ZAI_TOOLS`(633), `getAllZaiTools`(715) |
| 도구 실행 분기 | `src/agent/zai-agent.ts` — `executeTool`(935) switch |
| 도구 구현 | `src/tools/*.ts` |
| 확인 게이트 | `src/tools/confirmation-tool.ts` |
| 서브에이전트 | `src/agents/task-orchestrator.ts`, `src/tools/task-tool.ts` |
| MCP | `getMCPManager`(638), `initializeMCPServers`(645), `addMCPToolsToZaiTools`(721) |

## 1. 구조 — 스키마 리터럴 + switch 분기

도구 추상화(기반 클래스·인터페이스·레지스트리)가 **없다.** 대신:

1. `BASE_ZAI_TOOLS`(tools.ts:6)에 OpenAI 형식 `{type:"function", function:{name, description, parameters}}` 객체를 **직접 나열**.
2. `executeTool`(zai-agent.ts:935)이 `switch (toolCall.function.name)`로 각 도구 인스턴스에 위임.

즉 **스키마와 실행이 서로 다른 파일에 수동 동기화**된다 — 이름 불일치를 막을 타입/컴파일 보호가 없다.

## 2. 내장 도구 목록

| 도구명 | 구현 | 설명 |
|---|---|---|
| `view_file` | text-editor.ts | 파일 보기 (`path`, 선택적 `start_line`/`end_line` 범위) |
| `create_file` | text-editor.ts | 파일 생성 (`path`, `content`) — **확인 대상** |
| `str_replace_editor` | text-editor.ts | 문자열 치환 (`path`, `old_str`, `new_str`, `replace_all`) — **확인 대상** |
| `edit_file` | morph-editor.ts | **Morph Fast Apply** 기반 편집 (`target_file`, `instructions`, `code_edit`). `MORPH_API_KEY` 있을 때만 추가(622) |
| `bash` | bash.ts | 셸 실행 (`command`) — **확인 대상** |
| `search` | search.ts | 검색. 옵션이 풍부: `search_type`, `include_pattern`, `exclude_pattern`, `case_sensitive`, `whole_word`, `regex`, `max_results`, `file_types`, `include_hidden` |
| `batch_edit` | batch-editor.ts | 다중 파일 일괄 편집 (`type`, `files`, `pattern`, `params` 등) |
| `create_todo_list` / `update_todo_list` | todo-tool.ts | 할일 목록 생성·갱신 |
| (task) | task-tool.ts | 서브에이전트 생성. `TaskTool.getToolDefinition()`으로 항상 추가(627) |

**9~10개**. 웹 접근(fetch/search), 스킬, 플랜 모드, 팀, 크론, LSP, 노트북 편집이 **전부 없다**.

특이점: `search`가 다른 구현의 Glob+Grep을 하나로 합친 형태이며 옵션이 가장 세분화돼 있다. `batch_edit`는 다른 구현에 없는 일괄 편집 전용 도구다.

## 3. 노출 제어

`buildZaiTools()`(618) 단 하나의 함수로 결정된다:
- `MORPH_API_KEY` 환경변수 → `edit_file`(Morph) 삽입.
- `TaskTool` 정의는 무조건 추가.
- `getAllZaiTools()`(715)가 `addMCPToolsToZaiTools(ZAI_TOOLS)`로 MCP 도구를 합쳐 반환.

feature 플래그·권한 기반 필터·모델 능력 검사·지연 로딩이 **없다** — 모든 도구가 매 요청에 전체 스키마로 전송된다.

## 4. 권한 / 확인 게이트

**루프 레벨 권한 게이트가 없다.** `executeTool`의 switch에도 확인 로직이 없고, 대신 **도구 구현부에 `ConfirmationTool`이 내장**돼 있다:
- 대상: `create_file`, `str_replace_editor`, `bash` (파일 조작 + 셸).
- "세션 동안 승인" 옵션 제공.
- 거부 처리는 **시스템 프롬프트로 모델에 위임**한다 — "작업이 확인 거부로 차단되면 이를 인정하고 안내를 요청하거나 대안을 제시하라"(zai-agent.ts:294).

allowlist/denylist, 민감 패턴, 호출별 권한 오버라이드 같은 개념이 전혀 없다.

## 5. 서브에이전트

`TaskOrchestrator`(task-orchestrator.ts:17, EventEmitter)가 코드리뷰·테스트·문서화 등 특화 에이전트 정의(`agent-types.ts`)를 관리한다. `executeParallel`(145)은 `Promise.all`로 병렬 실행하며 `maxConcurrency` 설정 가능(286), `executeSequential`(164)도 제공. 각 서브에이전트는 독립 히스토리로 돌고 결과가 부모 히스토리에 합쳐진다.

## 6. MCP

`MCPManager` 싱글턴이 설정(`loadMCPConfig`)을 읽어 서버에 연결하고, 도구를 `ZAI_TOOLS`에 병합한다. 연결 중 verbose 로그를 감추려고 **`process.stderr.write`를 일시적으로 가로채는** 구현이 눈에 띈다(650~654) — 다른 구현들의 구조적 로깅 제어와 대비되는 임기응변.

## 7. 특징 요약

- **추상화 없음**: 기반 클래스·레지스트리·정책 플래그가 전무하고, 스키마 배열과 switch 문이 수동 동기화된다.
- 도구별 정책(읽기전용·병렬안전·절단·비가역)을 표현할 수단이 없어, 도구는 순차 실행되고 결과 크기 제한도 없다.
- 확인 게이트가 도구 안쪽에 흩어져 있고, 거부 후 처리를 프롬프트로 모델에 떠넘긴다.
- 그럼에도 `search`의 옵션 세분화와 `batch_edit`·Morph Fast Apply 통합은 실용적 강점이다.
- 도구 계층에서도 "개입 계층을 얼마나 체계화했는가"라는 축의 **원점**에 해당한다.
