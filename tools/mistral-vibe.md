# mistral-vibe (mistralai/mistral-vibe) 내장 도구 분석

Python. **권한 모델이 도구 설정에 직접 내장**(allowlist/denylist/sensitive_patterns)되고, 도구가 **스트리밍 제너레이터**라는 점이 특징.

## 0. 소스 맵

| 역할 | 위치 |
|---|---|
| 기반 클래스 | `vibe/core/tools/base.py` — `BaseTool`(141), `BaseToolConfig`(117), `ToolPermission`(102), `resolve_permission`(383) |
| 내장 도구 | `vibe/core/tools/builtins/*.py` |
| 권한 저장소 | `vibe/core/tools/permissions.py` — `PermissionContext`, `PermissionStore` |
| MCP | `vibe/core/tools/mcp/` — `pool.py`, `tools.py` |
| 커넥터 | `vibe/core/tools/connectors/connector_registry.py` |
| 도구 실행(턴루프) | `vibe/core/agent_loop/_loop.py` — `_execute_tool_call`(1717) |

## 1. BaseTool — 제네릭 4종 + 스트리밍

```python
class BaseTool[ToolArgs, ToolResult, ToolConfig, ToolState](ABC):
    description: ClassVar[str] = ""
    prompt_path: ClassVar[Path] | None = None
    selection_priority: ClassVar[int] = 0

    async def run(args, ctx) -> AsyncGenerator[ToolStreamEvent | ToolResult, None]
```

특징:
- **`run`이 async generator**다. 도구가 진행 중 `ToolStreamEvent`를 흘리다 최종 `ToolResult`를 낸다 → 장기 실행 도구의 실시간 출력이 1급 지원.
- **`selection_priority`**(153): 여러 도구 클래스가 같은 이름을 발행하면 우선순위 최댓값을 가진 "사용 가능한" 변형이 활성화된다. 예) `bash.py` vs `experimental_bash.py`가 같은 이름을 두고 경쟁 → 실험 구현을 설정으로 교체 가능.
- **프롬프트 외부화**: `get_tool_prompt()`(175)가 도구 소스 옆 `prompts/<name>.md`를 로드해 설명으로 쓴다(`functools.cache`).
- 제네릭 4종(Args/Result/Config/State)으로 도구별 **영속 상태**(`ToolState`)를 타입 안전하게 보유.

## 2. 권한 모델 — 설정에 내장

`BaseToolConfig`(117)가 도구마다 4개 필드를 갖는다:

| 필드 | 의미 |
|---|---|
| `permission` | `ALWAYS` / `NEVER` / `ASK` (기본 ASK) |
| `allowlist` | 자동 허용 패턴 |
| `denylist` | 자동 거부 패턴 |
| `sensitive_patterns` | **permission이 ALWAYS여도 ASK로 승격**시키는 패턴 |

`sensitive_patterns`가 특히 중요하다 — "bash는 항상 허용하되 `rm -rf`류만 물어봄" 같은 정책을 도구 설정만으로 표현한다.

추가로 `resolve_permission(args)`(383)가 **호출별 권한 오버라이드**를 제공한다: 인자를 보고 `PermissionContext`(세분화된 `required_permissions` + 레벨)를 돌려주거나, `None`이면 설정 수준 권한으로 폴백. 턴루프의 `_should_execute_tool`(_loop.py:1902)이 이를 호출하고, 미커버 권한만 `_ask_approval`로 사용자에게 묻는다(`PermissionStore.covers`로 기승인 범위 확인).

## 3. 내장 도구 목록 (`builtins/`)

**13개**로 최소 수준에 가깝다.

### 파일
| 도구 | 파일 | 설명 |
|---|---|---|
| `read_file` | read_file.py | 파일 읽기 |
| `write_file` | write_file.py | 파일 쓰기 |
| `edit` | edit.py | 편집 |
| `grep` | grep.py | 내용 검색 |

Glob/LS 전용 도구가 없다 — 패턴 탐색은 grep/bash로 수행.

### 실행
| 도구 | 파일 | 설명 |
|---|---|---|
| `bash` | bash.py | 셸 실행 |
| (실험) | experimental_bash.py | `selection_priority`로 교체되는 대체 구현 |

### 에이전트·계획
| 도구 | 파일 | 설명 |
|---|---|---|
| `task` | task.py | **서브에이전트** — 중첩 `AgentLoop` 인스턴스를 만들어 await |
| `todo` | todo.py | 할일 목록 |
| `exit_plan_mode` | exit_plan_mode.py | 플랜 모드 이탈 |
| `skill` | skill.py | 스킬 호출 |

### 웹·사용자
| 도구 | 파일 | 설명 |
|---|---|---|
| `web_fetch` | web_fetch.py | 페이지 가져오기 |
| `web_search` | web_search.py | 웹 검색 |
| `ask_user_question` | ask_user_question.py | 사용자 질의 |

### 확장
- **MCP**: `tools/mcp/pool.py`(연결 풀), `tools/mcp/tools.py`, `mcp_settings.py`. 스킬이 MCP 의존성을 선언하면 턴 시작 시 설치를 제안한다(`maybe_prompt_and_install_mcp_dependencies`).
- **커넥터**: `connectors/connector_registry.py` — 외부 서비스 통합.

팀/스웜 도구, 크론/스케줄 도구, 워크트리, LSP, 노트북 편집이 **없다**. 주기 실행은 도구가 아니라 세션 밖 `/loop` 스케줄러(`core/loop.py`)가 담당한다.

## 4. 도구 실행 파이프라인 (턴루프 연동)

`_execute_tool_call`(_loop.py:1717) 순서:
1. 도구 조회 → 입력 직렬화(실패 시 tool error 이벤트).
2. **`_run_pre_tool_pipeline`**(1744) — pre_tool 훅: deny(에러 반환) 또는 **인자 rewrite**.
3. **`_should_execute_tool`**(1902) — 위 권한 모델 평가.
4. SKIP이면 `_handle_tool_skip`, 아니면 `_invoke_tool`(post_tool 훅 포함).

여러 도구는 `_run_tools_concurrently`(1651)로 **동시 실행**된다(asyncio.Queue + 완료 센티넬). 도구별 병렬 안전 플래그가 없어 **전부 동시 실행**되는 점이 openclaude(`isConcurrencySafe`)·qwen-code(`CONCURRENCY_SAFE_KINDS`)와 다르다.

## 5. 특징 요약

- **권한이 도구 설정의 1급 필드**(permission + allowlist + denylist + sensitive_patterns)로, "기본 허용하되 위험 패턴만 확인" 정책을 선언적으로 표현.
- `resolve_permission(args)`로 **호출별 세분화 권한**을 제공하고, `PermissionStore.covers`로 기승인 범위를 추적해 재질문을 줄임.
- 도구가 **async generator**라 스트리밍 진행 출력이 자연스럽다.
- `selection_priority`로 같은 이름의 도구 구현을 **교체 가능**(실험 구현 A/B).
- 도구 수는 13개로 미니멀하고, 병렬 안전성 구분 없이 전부 동시 실행.
