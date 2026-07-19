# openclaude (Gitlawb/openclaude) 내장 도구 분석

Claude Code 계보. **도구 인터페이스가 조사 대상 중 가장 풍부**하며(20개 이상의 메타 속성), 도구 수도 가장 많다(50+).

## 0. 소스 맵

| 역할 | 위치 |
|---|---|
| Tool 인터페이스 | `src/Tool.ts` — 속성 정의(400~510), 기본값(793) |
| 도구 목록 | `src/tools.ts` — `getAllBaseTools`(183), `getTools`(262), `filterToolsByDenyRules`(253) |
| 도구 구현 | `src/tools/<ToolName>/` (도구별 디렉토리) |
| 도구 실행 | `src/services/tools/toolExecution.ts` `runToolUse`(407) |

## 1. Tool 인터페이스 — 메타 속성이 곧 정책

각 도구는 단순 실행 함수가 아니라 **동작 정책을 선언**한다(`Tool.ts:400~510`):

| 속성 | 의미 |
|---|---|
| `isReadOnly(input)` | 읽기 전용 여부. 기본 `false`(쓰기로 가정) |
| `isConcurrencySafe(input)` | 병렬 실행 안전. 기본 `false` |
| `isDestructive?(input)` | 비가역 작업(삭제·덮어쓰기·전송) 표시 |
| `isEnabled()` | 런타임 활성 여부 |
| `interruptBehavior?()` | 실행 중 새 메시지 도착 시 `'cancel'`(중단·결과 폐기) vs `'block'`(계속, 메시지 대기). 기본 block |
| `maxResultSizeChars` | 초과 시 결과를 **디스크에 저장하고 미리보기+경로만** 모델에 전달. `Infinity`면 저장 안 함(Read는 Read→file→Read 순환 방지) |
| `isSearchOrReadCommand?(input)` | UI에서 접어 표시할 검색/읽기/목록 작업 구분 |
| `isOpenWorld?(input)` | 외부 세계 접근 여부 |
| `requiresUserInteraction?()` | 사용자 상호작용 필요 |
| `shouldDefer` / `alwaysLoad` | **ToolSearch 지연 로딩** 제어. defer면 스키마를 초기 프롬프트에서 빼고 검색 후 로드, alwaysLoad면 항상 노출 |
| `searchHint` | ToolSearch 키워드 매칭용 3~10단어 문구(도구명에 없는 용어 권장, 예: NotebookEdit엔 'jupyter') |
| `aliases` | 이름 변경 시 하위 호환(과거 트랜스크립트의 옛 이름 호출 처리) |
| `inputSchema` / `inputJSONSchema` | Zod 스키마 또는 MCP용 raw JSON Schema |
| `outputSchema` | 출력 스키마 |
| `isMcp` / `isLsp` / `mcpInfo` | 출처 표시 |
| `strict` | API의 스키마 준수 강제 모드 |

## 2. 내장 도구 목록 (`getAllBaseTools`, tools.ts:183)

### 파일
| 도구 | 설명 |
|---|---|
| `Read` | 파일 읽기. `maxResultSizeChars: Infinity`(순환 방지) |
| `Edit` | 문자열 치환 편집 |
| `Write` | 파일 쓰기 |
| `NotebookEdit` | Jupyter 노트북 셀 편집 |
| `Glob` / `Grep` | 파일 패턴 매칭 / 내용 검색. **ant 네이티브 빌드에선 제외**(bfs/ugrep가 셸에 임베드돼 있어 불필요, 191) |
| `RepoMap` | 저장소 구조 맵 |

### 실행
| 도구 | 설명 |
|---|---|
| `Bash` | 셸 실행 |
| `PowerShell` | Windows 셸(플랫폼별 게터) |
| `REPL` | VM 내부 REPL. 활성 시 Bash/Read/Edit 등을 감싸므로 원시 도구를 숨김(`REPL_ONLY_TOOLS`), `USER_TYPE==='ant'` 한정 |

### 에이전트·태스크
| 도구 | 설명 |
|---|---|
| `Agent` (별칭 `Task`) | 서브에이전트 생성. 자식이 같은 query 루프를 돎 |
| `TaskOutput` / `TaskStop` | 백그라운드 태스크 출력 조회 / 중단 |
| `TaskCreate` / `TaskGet` / `TaskUpdate` / `TaskList` | 태스크 관리(todo v2 활성 시) |
| `TodoWrite` | 레거시 할일 목록 |

### 팀(스웜)
| 도구 | 설명 |
|---|---|
| `SendMessage` | **동료 에이전트에 메시지 전송** — 파일 메일박스에 기록(턴루프 문서 §10) |
| `TeamCreate` / `TeamDelete` | 팀 생성·삭제(`isAgentSwarmsEnabled()` 게이트) |
| `ListPeers` | 동료 목록 |

### 플랜·검증
| 도구 | 설명 |
|---|---|
| `EnterPlanMode` / `ExitPlanMode`(V2) | 플랜 모드 진입·이탈 |
| `VerifyPlanExecution` | 플랜 실행 검증 |
| `ReviewArtifact` | 리뷰 산출물 |

### 웹
| 도구 | 설명 |
|---|---|
| `WebFetch` / `WebSearch` | 페이지 가져오기 / 검색 |
| `WebBrowser` | 브라우저 제어(옵션) |

### 컨텍스트·메타
| 도구 | 설명 |
|---|---|
| `ToolSearch` | **지연 도구 검색** — defer된 도구의 스키마를 키워드로 찾아 로드 |
| `Skill` / `DiscoverSkills` | 스킬 호출 / 탐색 |
| `snip` | 히스토리 구간 절제 |
| `CtxInspect` | 컨텍스트 점검(디버그) |
| `StructuredOutput` | 구조화 출력 강제(`SyntheticOutputTool`) |

### 사용자 상호작용·알림
| 도구 | 설명 |
|---|---|
| `AskUserQuestion` | 선택지 질의 |
| `Brief` / `SendUserMessage` | 사용자 브리핑 |
| `SendUserFile` | 파일 전달 |
| `PushNotification` | 푸시 알림 |

### 스케줄·자동화
| 도구 | 설명 |
|---|---|
| `CronCreate` / `CronDelete` / `CronList` | 크론 스케줄 관리 |
| `RemoteTrigger` | 원격 트리거 |
| `Monitor` | 조건 감시 |
| `Sleep` | 대기 |
| `WorkflowTool` | 워크플로 실행 |

### 개발 환경
| 도구 | 설명 |
|---|---|
| `LSP` | 언어 서버 질의(`isLsp`) |
| `EnterWorktree` / `ExitWorktree` | git worktree 격리(`isWorktreeModeEnabled()`) |
| `TerminalCapture` | 터미널 캡처 |
| `SuggestBackgroundPR` / `SubscribePR` | PR 제안·구독 |

### MCP
`ListMcpResourcesTool`, `ReadMcpResourceTool`, `mcp`(MCPTool), `McpAuthTool`. MCP 서버 툴은 `mcp__{server}__{tool}` 이름으로 편입되며 `mcpInfo`를 갖는다.

### 테스트 전용
`OverflowTest`, `TestingPermissionTool`(NODE_ENV==='test'), `Tungsten`, `firecrawl`.

## 3. 노출 제어 — 4단계

1. **빌드/환경 게이트**: `hasEmbeddedSearchTools()`, `USER_TYPE==='ant'`, `NODE_ENV`, 플랫폼별 게터(PowerShell).
2. **feature 게이트**: `isTodoV2Enabled()`, `isWorktreeModeEnabled()`, `isAgentSwarmsEnabled()`, `isToolSearchEnabledOptimistic()`.
3. **권한 기반 사전 필터**: `filterToolsByDenyRules`(253) — deny 규칙에 걸린 도구는 **모델이 보기도 전에** 제거. `mcp__server` 같은 서버 접두사 규칙이면 해당 서버 도구 전체가 사라진다.
4. **모드 게이트**: `CLAUDE_CODE_SIMPLE`이면 Bash/Read/Edit만, REPL 모드면 원시 도구를 REPL로 대체.

여기에 **ToolSearch 지연 로딩**(`shouldDefer`/`alwaysLoad`/`searchHint`)이 5번째 층으로 작동해, 50+ 도구의 스키마가 프롬프트를 채우지 않게 한다.

## 4. 권한 연동

`runToolUse`(toolExecution.ts:407) 흐름: 도구 조회(별칭 폴백) → abort 검사 → **`canUseTool` 콜백** → PreToolUse 훅 → 실행 → PostToolUse 훅. 거부 시 `executePermissionDeniedHooks`. 스웜에서는 워커의 권한 요청이 메일박스 `permission_request` 프로토콜 메시지로 리더에게 전달돼 리더 UI에서 결정된다.

## 5. 특징 요약

- **도구 = 실행 로직 + 정책 선언**. `isReadOnly`/`isConcurrencySafe`/`isDestructive`/`interruptBehavior`/`maxResultSizeChars`가 스케줄링·UI·컨텍스트 관리를 자동화한다.
- `maxResultSizeChars` 초과 시 **결과를 디스크로 내리고 경로만 전달**하는 자동 오프로딩이 독특하다.
- 도구 수가 가장 많고(50+), 그 대가로 `ToolSearch` 지연 로딩이 필수 인프라가 됐다.
- deny 규칙이 **모델 노출 단계에서 선제 적용**되는 점(다른 구현은 호출 시점 검사)이 안전 설계상 차별점.
