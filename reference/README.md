# reference/ — 분석 대상 레포지토리 재구성 가이드

이 폴더의 클론들은 **git에 포함되지 않는다**(`.gitignore`). 분석 문서([`../trunloop/`](../trunloop/README.md), [`../tools/`](../tools/README.md))의 `file:line` 참조를 검증하거나 재현하려면 아래 절차로 직접 클론한다.

## 빠른 재구성

프로젝트 루트에서 실행:

```bash
mkdir -p reference && cd reference

git clone --depth 1 https://github.com/openai/codex.git            codex
git clone --depth 1 https://github.com/Gitlawb/openclaude.git      openclaude
git clone --depth 1 https://github.com/QwenLM/qwen-code.git        qwen-code
git clone --depth 1 https://github.com/mistralai/mistral-vibe.git  mistral-vibe
git clone --depth 1 https://github.com/sst/opencode.git            opencode
git clone --depth 1 https://github.com/guizmo-ai/zai-glm-cli.git   zai-glm-cli
```

PowerShell도 동일한 명령을 그대로 쓸 수 있다.

## 분석 기준 커밋 (재현용)

문서의 줄 번호는 **아래 커밋 기준**이다. `--depth 1`은 최신 커밋만 가져오므로, 시간이 지나면 줄 번호가 어긋난다. 문서를 정확히 재현하려면 해당 커밋을 체크아웃한다.

| 디렉토리 | 레포 | 커밋 | 커밋 일시 |
|---|---|---|---|
| `codex` | [openai/codex](https://github.com/openai/codex) | `6bd3f5e3db8275c10c7e4bbcc1342c32a89b7eee` | 2026-07-18 |
| `openclaude` | [Gitlawb/openclaude](https://github.com/Gitlawb/openclaude) | `0effa0f42b6dbc6a4800e19f4b2d8d588269906b` | 2026-07-17 |
| `qwen-code` | [QwenLM/qwen-code](https://github.com/QwenLM/qwen-code) | `b4559dc8ba92984396a447fb25a2826804d27d74` | 2026-07-18 |
| `mistral-vibe` | [mistralai/mistral-vibe](https://github.com/mistralai/mistral-vibe) | `0685654a40a4035966891289065379a751a7e617` | 2026-07-17 |
| `opencode` | [sst/opencode](https://github.com/sst/opencode) | `45cd8d76920839e4a7b6b931c4e26b52e1495636` | 2026-07-17 |
| `zai-glm-cli` | [guizmo-ai/zai-glm-cli](https://github.com/guizmo-ai/zai-glm-cli) | `faa1f3ba09fdcf77cd57389ea2e1884d594bd9b4` | 2025-10-28 |

특정 커밋으로 맞추려면 얕은 클론에 해당 커밋만 추가로 받는다:

```bash
cd reference/codex
git fetch --depth 1 origin 6bd3f5e3db8275c10c7e4bbcc1342c32a89b7eee
git checkout 6bd3f5e3db8275c10c7e4bbcc1342c32a89b7eee
```

전체 히스토리가 필요하면 `--depth 1` 없이 클론한 뒤 체크아웃한다.

## 레포 선정 근거

요청받은 이름과 실제 레포가 1:1로 대응하지 않는 경우가 있어 아래와 같이 판단했다.

| 요청 이름 | 선정 레포 | 근거 |
|---|---|---|
| codex | `openai/codex` | 공식 |
| openclaude | `Gitlawb/openclaude` | Claude Code 계보의 오픈소스 포크 |
| qwencode | `QwenLM/qwen-code` | 공식 (gemini-cli 계보) |
| mistral code | `mistralai/mistral-vibe` | **"Mistral Code" 제품 자체는 Continue 기반 비공개 엔터프라이즈 IDE 어시스턴트**다. Mistral의 공식 공개 CLI 에이전트는 mistral-vibe이므로 이것을 대상으로 함 |
| glm code | `guizmo-ai/zai-glm-cli` | **Z.ai(Zhipu) 공식 조직(zai-org)에 공개 CLI 에이전트 레포가 없음**. GLM Coding Plan은 Claude Code/Cline 등 타사 도구에 GLM 엔드포인트를 연결하는 구독 상품. grok-cli를 포크한 커뮤니티 구현체가 가장 근접 |
| opencode | `sst/opencode` | 공식 |
| claim | — | **해당 이름의 코딩 에이전트를 GitHub에서 찾지 못함.** 유사 명칭 레포(claimcheck, cngx, musts)는 모두 "에이전트가 주장(claim)한 작업을 검증하는 도구"였다. Cline 등의 오기일 가능성 |

## 주의

- 각 레포는 원저작자의 라이선스를 따른다. 이 프로젝트는 **분석 문서만** 포함하며 대상 코드를 재배포하지 않는다.
- 클론 총 용량은 약 2GB 내외(파일 23,000+개)다.
