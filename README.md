# 코딩 에이전트 아키텍처 분석

주요 오픈소스 코딩 에이전트 6종의 **턴루프(turn loop)** 와 **내장 도구(built-in tools)** 를 소스 레벨에서 추적해 정리한 문서 모음.

분석 대상: [codex](https://github.com/openai/codex) · [openclaude](https://github.com/Gitlawb/openclaude) · [qwen-code](https://github.com/QwenLM/qwen-code) · [opencode](https://github.com/sst/opencode) · [mistral-vibe](https://github.com/mistralai/mistral-vibe) · [zai-glm-cli](https://github.com/guizmo-ai/zai-glm-cli)

## 구성

| 폴더 | 내용 |
|---|---|
| **[trunloop/](trunloop/README.md)** | 턴루프 분석. 사용자 입력 1건이 들어와 최종 응답으로 끝날 때까지의 제어 흐름 — 게이트·훅·스티어링·자가 턴·에이전트 간 대기를 12개 축으로 정리 |
| **[tools/](tools/README.md)** | 내장 도구 분석. 각 에이전트가 모델에 어떤 도구를 어떻게 노출하는가 |
| **[tools/categories.md](tools/categories.md)** | 도구를 12개 카테고리로 묶고 **각 카테고리가 턴루프의 어느 지점에서 어떻게 작동하는지** 교차 분석 |
| **[reference/](reference/README.md)** | 분석 대상 레포 재구성 가이드 (클론 자체는 git에 미포함) |

## 읽는 순서

1. [trunloop/README.md](trunloop/README.md) — 6개 구현의 턴루프 비교표와 공통 골격 다이어그램
2. 관심 있는 에이전트의 개별 문서 (예: [trunloop/codex.md](trunloop/codex.md))
3. [tools/categories.md](tools/categories.md) — 도구와 루프의 관계를 5유형으로 분류

## 핵심 관점

- **루프의 "재료"가 무엇인가** — 메모리 내 상태(codex·openclaude·qwen-code)인가, 매 반복 다시 읽는 영속 메시지 로그(opencode)인가.
- **자가 턴 발생 계층** — 단순 구현은 도구 호출 유무로만 반복한다. 현대적 구현은 그 위에 Stop 훅·goal·next-speaker·nudge·토큰 예산 continuation이 겹겹이 쌓이고, **각 계층마다 폭주 방지 cap이 반드시 함께 붙는다**.
- **에이전트 간 "대기"의 두 철학** — 블로킹형(codex `wait_agent`: status watch 채널을 deadline까지 정지)과 비블로킹형(openclaude 스웜: 파일 메일박스를 폴링해 attachment로 흡수하고 idle 훅으로 워커를 살려둠).
- **도구 카테고리 확장은 턴루프 복잡도의 함수** — 루프에 개입 계층(훅·큐·압축·자가턴)이 있어야 그것을 조작하는 도구가 존재할 수 있다.

## 주의

- 문서의 `file:line` 참조는 [reference/README.md](reference/README.md)에 기록된 **특정 커밋 기준**이다. 대상 레포는 활발히 개발 중이라 최신 클론에서는 줄 번호가 어긋날 수 있다.
- 분석 문서만 포함하며 대상 레포의 코드를 재배포하지 않는다. 각 레포는 원저작자의 라이선스를 따른다.
