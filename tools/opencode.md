# opencode (sst/opencode) 내장 도구 분석

TypeScript + Effect. 도구 세트가 **가장 작고(16개) 미니멀**하며, 정책 플래그 대신 **래퍼가 공통 처리를 자동 적용**하는 설계.

## 0. 소스 맵

| 역할 | 위치 |
|---|---|
| 도구 정의 타입 | `packages/opencode/src/tool/tool.ts` — `Def`(55), `Context`(36), `define`(151), `wrap`(99) |
| 레지스트리 | `packages/opencode/src/tool/registry.ts` — `Tool.init` 목록(205~221), `builtin` 배열(226~244) |
| 도구 구현 | `packages/opencode/src/tool/*.ts` |
| 프롬프트(설명문) | `packages/opencode/src/tool/*.txt` — 도구 설명이 별도 텍스트 파일 |
| 출력 절단 | `packages/opencode/src/tool/truncate.ts` |

## 1. Tool.Def — 극단적 미니멀리즘

```ts
interface Def {
  id: string
  description: string
  parameters: Schema.Decoder      // Effect Schema
  jsonSchema?: JSONSchema7
  execute(args, ctx): Effect<ExecuteResult>
  formatValidationError?(error): string
}
```

`isReadOnly` / `isConcurrencySafe` / `kind` 같은 **정책 플래그가 하나도 없다.** 대신:

- **권한**은 `ctx.ask(...)`(tool.ts:45)로 도구가 필요할 때 직접 요청한다 — 선언이 아니라 호출.
- **출력 절단**은 `wrap`(99)이 **모든 도구에 자동 적용**한다: `execute` 반환 후 `truncate.output(result.output, {}, agent)`를 거쳐, 잘렸으면 `metadata.truncated`와 `outputPath`를 붙인다(131~144). 도구가 이미 `metadata.truncated`를 설정했으면 건너뛴다. → openclaude의 `maxResultSizeChars`에 해당하는 기능이지만 **에이전트별 설정으로 중앙에서 강제**된다.
- **인자 검증**은 `decodeUnknownEffect`로 파싱하고 실패 시 `InvalidArgumentsError`(24)를 던진다. 이 에러의 `message` 게터가 모델이 볼 프로즈를 생성한다: *"...Please rewrite the input so it satisfies the expected schema."* — 즉 **스키마 위반을 모델에게 재작성 지시로 되돌리는 정형화된 경로**.
- 파서 클로저를 도구 init당 1회만 컴파일하는 최적화(110~111).

`description`은 대부분 `*.txt` 파일에서 로드된다(`read.txt`, `edit.txt`, `grep.txt` 등) — 프롬프트를 코드에서 분리.

## 2. 내장 도구 목록 (`registry.ts:226`)

| id | 파일 | 설명 |
|---|---|---|
| `invalid` | invalid.ts | 알 수 없는 도구 호출을 흡수하는 폴백 |
| `shell` | shell.ts | 셸 실행 |
| `read` | read.ts | 파일 읽기 |
| `glob` | glob.ts | 파일 패턴 매칭 (Ripgrep 백엔드) |
| `grep` | grep.ts | 내용 검색 |
| `edit` | edit.ts | 편집 |
| `write` | write.ts | 파일 쓰기 |
| `patch` | apply_patch.ts | 패치 적용 |
| `task` | task.ts | **서브에이전트** — 턴루프의 `handleSubtask`가 이것을 인라인 await |
| `fetch` | webfetch.ts | 웹 페이지 |
| `search` | websearch.ts | 웹 검색 (`mcp-websearch.ts` 경유 옵션) |
| `todo` | todo.ts | 할일 목록 |
| `skill` | skill.ts | 스킬 호출 |
| `question` | question.ts | 사용자 질의 (`questionEnabled` 게이트) |
| `lsp` | lsp.ts | 언어 서버 (`flags.experimentalLspTool`) |
| `plan` | plan.ts | 플랜 모드 이탈 (`experimentalPlanMode` + CLI 클라이언트 한정) |
| `execute` | code-mode.ts | 코드 모드 실행 (조건부) |

**16개**로 조사 대상 중 가장 적다. 팀/스웜 도구, 크론/스케줄 도구, 워크트리 도구, 노트북 편집이 **없다**.

## 3. 노출 제어

세 가지뿐이라 단순하다:
1. **런타임 플래그**: `flags.experimentalLspTool`, `flags.experimentalPlanMode && flags.client === "cli"`, `questionEnabled`.
2. **code-mode 조건부**: `codeModeTool`이 있으면 `execute` 추가.
3. **권한 규칙**: 턴루프의 `SessionTools.resolve`가 에이전트의 `PermissionV1.Rule[]`을 반영해 도구셋을 구성.

`custom` 도구(플러그인 제공)와 `builtin`이 분리 관리되고, MCP 도구는 `MCP`/`McpCatalog` 서비스로 편입된다.

## 4. 권한 게이트

도구가 `ctx.ask({permission, patterns, metadata, always, ruleset})`를 호출하면 Effect가 사용자 응답까지 대기한다. 특수 사례로 턴루프의 **doom-loop 가드**가 있다 — 동일 도구·동일 input이 `DOOM_LOOP_THRESHOLD`회 반복되면 `permission.ask({permission:"doom_loop"})`로 확인을 요구(processor.ts:372).

서브에이전트(`task`)는 `Permission.merge(taskAgent.permission, session.permission)`로 부모 규칙을 상속받는다(prompt.ts:346).

## 5. 특징 요약

- **도구는 순수 실행 단위**, 정책은 전부 바깥(래퍼·권한 서비스·에이전트 설정)에 있다. 도구 인터페이스가 6필드로 가장 얇다.
- **출력 절단이 래퍼에서 전역 강제**되고 잘린 내용은 `outputPath`로 디스크에 남는다.
- 스키마 위반을 `InvalidArgumentsError`의 정형화된 문구로 모델에 되돌려 자가 수정을 유도.
- 도구 설명을 `.txt`로 분리해 프롬프트를 코드에서 떼어냄.
- 도구 수가 최소(16개)이며 팀/크론/워크트리 계열이 통째로 없다 — 확장은 plugin과 MCP로 푼다는 일관된 철학.
