---
title: "claude -p가 과금 채널이 될 뻔한 이야기 — 데스크톱 앱 채널 재설계와 PTY+JSONL Watch 패턴"
author: abruption
date: 2026-06-17 14:00:00 +0900
categories: [Programming, Claude Code]
tags: [claude-code, agent-sdk, electron, node-pty, jsonl-watch, mcp, architecture, headless]
---

> **2026-06-15 업데이트**: Anthropic은 본문에서 다루는 Agent SDK monthly credit 정책 변경을 **일시 보류**한다고 공식 발표했다. 현재 `claude -p`와 Agent SDK 호출은 종전대로 구독의 usage limit를 사용한다. 본 글은 정책이 시행될 것이라 가정해 5월에 내렸던 채널 결정이 어떻게 흘러왔는지에 대한 회고다. 정책은 다시 시행될 가능성이 남아 있고, 더 중요한 건 그와 무관하게 채택한 아키텍처가 옳았는지의 평가다.

## 데스크톱 앱이 Claude Code를 자동화해야 한다

도메인 지식을 4NS 구조로 체계화하는 온톨로지 엔진을 만들고 있다. 엔진 자체는 MCP 서버이고, 그 위에 사용자 워크플로를 얹는 데스크톱 앱이 한 층 더 있다. 사용자 모드(4단계)와 개발자 모드(13페이지)를 듀얼로 제공하고, 두 모드 모두 같은 Claude Code 세션을 공유한다.

처음 그린 그림은 단순했다. 데스크톱 앱이 사용자의 자연어 입력을 받으면 Claude Code를 호출하고, 응답을 받아 UI에 뿌린다. CLI를 호출하는 가장 쉬운 방법은 `claude -p`(headless 모드)다. stdout으로 응답이 깔끔하게 떨어지고, `--output-format json --json-schema`로 구조화 출력까지 강제할 수 있다. 시연용 데모도 두 줄짜리 명령으로 끝난다.

그런데 5월 중순에 Anthropic 정책 페이지에서 한 줄을 읽고 그림을 다시 그려야 했다.

## 5월의 정책 — `claude -p`가 monthly credit를 차감한다

[Claude 공식 support 문서](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)에 따르면, 6월 15일부터 시행 예정이라고 안내된 정책은 다음과 같았다.

**Agent SDK monthly credit 대상 — 구독에서 별도 차감**

| 플랜 | 월 크레딧 |
|---|---|
| Pro | $20 |
| Max 5x | $100 |
| Max 20x | $200 |
| Team (Standard) | $20 |
| Team (Premium) | $100 |
| Enterprise (usage-based) | $20 |
| Enterprise (Premium seat) | $200 |

**크레딧을 소모하는 호출**

- Claude Agent SDK를 사용하는 사용자 프로젝트 (Python·TypeScript)
- Claude Code의 **`claude -p`** 명령 (non-interactive)
- Claude Code GitHub Actions 통합
- 구독으로 인증하는 서드파티 앱

**대상이 아닌 호출**

- 터미널·IDE의 **인터랙티브 Claude Code** — 종전대로 구독의 usage limit 사용

읽으면서 곤란해졌다. 우리가 만드는 데스크톱 앱은 정확히 "구독으로 인증하는 서드파티 앱"에 해당한다. 첫 사용자가 앱을 띄우고 자연어 한 줄을 입력하면 그 즉시 monthly credit가 줄어든다. 시연 자리에서 두세 번 시연하면 Pro 사용자의 $20는 금방 바닥난다.

Anthropic의 가이드도 명료했다. "Teams running shared production automation should use Claude Platform with an API key for predictable pay-as-you-go billing." API 키를 BYO로 받는 옵션이 자연스러운 답이지만, 우리는 그러고 싶지 않았다. 데스크톱 앱의 미덕은 사용자가 자기 구독으로 그냥 쓰는 것이고, API 키 입력을 강요하는 순간 첫인상이 완전히 달라진다.

남은 길은 하나뿐이다. **인터랙티브 모드**로 가야 한다.

## 인터랙티브 모드의 제약을 받아들이기

`claude -p`를 폐기한다는 결정은 단순해 보이지만 부수 효과가 크다. headless 모드의 가장 큰 장점이었던 두 가지가 통째로 사라진다.

- `--output-format json --json-schema` 강제 구조화 출력 — 인터랙티브 stdout은 사용자용 포맷이라 파싱 불안정
- stream-json 5-handler 라우팅 — non-interactive 전용

대신 인터랙티브 모드는 사용자가 직접 키보드를 두드리는 형태를 가정한다. 우리 앱은 사용자 대신 키를 두드려야 하고, 응답도 사용자가 화면을 보고 읽는 게 아니라 코드가 파싱해야 한다. 이 두 가지를 어떻게 해결할 것인가가 새 아키텍처의 전부였다.

## 대체 채널 — PTY + JSONL Session Watch

답은 두 조각으로 나뉜다.

**조각 1 — node-pty로 인터랙티브 Claude Code spawn**

Electron 메인 프로세스가 `node-pty`로 가상 터미널을 띄우고, 그 안에 `claude` CLI를 인터랙티브로 실행한다. 사용자 입력은 메인이 PTY에 명령으로 주입하고, 화면 출력은 xterm.js로 렌더링해 사용자도 직접 볼 수 있게 한다(투명성이 곧 신뢰다).

```typescript
import { spawn as ptySpawn } from 'node-pty'
import { randomUUID } from 'node:crypto'

const sessionId = randomUUID()

const term = ptySpawn('claude', [
  '--session-id', sessionId,            // 새 세션을 이 UUID로 강제
  '--mcp-config', '.mcp.json',          // 프로젝트 scope MCP만
  '--strict-mcp-config',                // 사용자 전역 MCP 누수 차단
  '--append-system-prompt', deferSystemPrompt,
  '--permission-mode', 'dontAsk',       // 화이트리스트 밖은 조용히 거부
  '--allowedTools', 'mcp__plugin_cozy_cozy__*,Read,Edit,Write,Glob,Grep,Bash,Skill,Agent,TodoWrite',
  '--disallowedTools', 'AskUserQuestion',
], {
  name: 'xterm-256color',
  cols: 120, rows: 40,
  cwd: workspacePath,
  env: { ...process.env },
})
```

이때 `--session-id`로 미리 채번한 UUID를 강제하는 게 핵심이다. CLI가 알아서 발급하는 ID를 기다리지 않고 우리가 정한 ID로 세션을 시작해야, 다음 조각이 가능해진다.

**조각 2 — `~/.claude/projects/<derived>/<UUID>.jsonl` 파일 watch**

Claude Code 인터랙티브는 모든 turn(사용자 입력·assistant 응답·tool_use·tool_result)을 JSONL 파일에 실시간으로 기록한다. 경로 규칙은 다음과 같다.

- `<derived>`: 현재 작업 디렉토리(cwd)의 `/`를 `-`로 치환한 문자열
- 예: cwd가 `/Users/abruptly/QuinTet/cozy-devtool`이면 `<derived>`는 `-Users-abruptly-QuinTet-cozy-devtool`
- 파일: `~/.claude/projects/<derived>/<UUID>.jsonl`

이 파일을 `fs.watch`로 추적하면 인터랙티브 세션의 모든 이벤트를 stream-json 핸들러 없이도 받을 수 있다.

```typescript
import { watch } from 'node:fs'
import { createReadStream } from 'node:fs'
import readline from 'node:readline'

const jsonlPath = `${homedir()}/.claude/projects/${derived}/${sessionId}.jsonl`

let lastSize = 0
watch(jsonlPath, async () => {
  const { size } = await stat(jsonlPath)
  if (size <= lastSize) return

  const stream = createReadStream(jsonlPath, { start: lastSize, end: size })
  const rl = readline.createInterface({ input: stream })
  for await (const line of rl) {
    if (!line.trim()) continue
    const event = JSON.parse(line)
    // event.type: 'user' | 'assistant' | 'tool_use' | 'tool_result' | ...
    dispatch(event)
  }
  lastSize = size
})
```

각 줄이 하나의 turn이고, `tool_use` 이벤트에는 MCP 도구 호출 인자가, `tool_result` 이벤트에는 응답이 그대로 들어있다. JSON Schema 강제 출력이 없어도 도구 응답의 구조는 MCP 서버가 보장하므로 파싱이 깨질 일이 없다.

## 부수 통제 — 인터랙티브를 무인처럼 다루기

인터랙티브 모드의 또 다른 함정은 사용자에게 묻는 것이다. `AskUserQuestion` 도구가 호출되면 TUI가 응답을 기다리며 멈춘다. 데스크톱 앱에서 사용자는 xterm.js 화면을 보지만 거기서 키보드 인풋을 받아 다시 PTY에 주입하는 건 UX가 거칠다.

`--disallowedTools AskUserQuestion`로 도구 자체를 차단하고, 차단 시 fallback 평문 질문(스킬 SKILL.md가 의도하는)도 막기 위해 `--append-system-prompt`에 정확한 지시를 주입한다.

```typescript
const deferSystemPrompt =
  'AskUserQuestion은 사용 불가. 진행 도중 사용자에게 묻지 말 것 — ' +
  '도구 호출이든 평문 turn 질문이든 모두 금지. ' +
  '도메인 지식·메타모델·일반 결정형 선택은 도메인 baseline + 제공된 컨텍스트로 ' +
  'best-effort 진행한다. 사용자 고유 정보(로컬 경로·URL·자격증명·내부 식별자)가 ' +
  '꼭 필요한 스텝만 그 스텝을 skip하고, 필요한 입력은 마지막 턴에서 평문으로 일괄 ' +
  '보고하라.'
```

권한 모드는 `dontAsk`로 두어 화이트리스트 밖 도구가 호출되면 사용자에게 묻지 않고 조용히 거부한다. `--allowedTools`에는 우리 MCP 서버 도구(`mcp__plugin_cozy_cozy__*`)와 표준 도구 일부만 명시한다. 인터랙티브 모드인데도 무인 모드처럼 굴러간다.

`--strict-mcp-config`는 보안 측면에서 중요하다. 명시한 `.mcp.json`만 사용하고 사용자 전역 MCP 설정은 무시한다. 사용자의 다른 프로젝트에 등록된 MCP 서버가 우리 세션에 끼어드는 사고를 막는다.

## 6월 15일의 반전

새 채널을 만들고 시연도 몇 번 돌리고, 마일스톤 두 개를 더 진행한 뒤에 정책 페이지를 다시 확인했다. 6월 15일자로 한 문장이 추가돼 있었다.

> We're pausing the changes to Claude Agent SDK usage described below. For now, nothing has changed: Claude Agent SDK, `claude -p`, and third-party app usage still draw from your subscription's usage limits.

정책이 보류됐다. 한 달 가까이 들인 채널 재설계가 무의미해졌나? 잠시 그렇게 생각했지만 결론은 정반대였다.

**첫째**, 정책은 다시 시행될 수 있다. 보류이지 철회가 아니다. 시행 시점이 정해지면 사용자에게 "이번 버전부터 채널이 바뀌었습니다"라고 안내하며 며칠 안에 갈아엎는 것보다, 처음부터 영향받지 않는 구조로 가 있는 게 안전하다.

**둘째**, 인터랙티브 채널은 사용자 경험 측면에서 더 낫다. xterm.js 화면에 Claude가 무슨 도구를 호출했고 어떤 응답을 받았는지 그대로 보인다. headless였다면 사용자는 결과만 보고, 중간 과정은 블랙박스다. 데스크톱 앱의 신뢰를 만드는 건 이 가시성이다.

**셋째**, JSONL Watch 패턴 자체가 강력한 도구다. 세션 영속·재개·재방문이 자연스럽다. 사용자가 앱을 끄고 나갔다가 돌아와도 `~/.claude/projects/.../<UUID>.jsonl`을 읽으면 모든 turn이 그대로 있다. `--resume` 플래그로 같은 세션을 이어 갈 수도 있다. headless였다면 stdout을 따로 영속해야 했을 것이다.

## 회고 — 외부 정책에 의존하지 않는 결정

소프트웨어 아키텍처는 보통 "지금 가장 쉬운 방법"으로 흘러간다. `claude -p`는 정말 쉬웠다. 정책이 그대로였다면 우리는 시연 자리에서 사용자의 monthly credit가 바닥나는 걸 발견하고서야 부랴부랴 갈아엎었을 것이다.

이 사례에서 배운 한 가지는 "외부 정책 페이지에 한 줄이 추가되면 한 줄짜리 답이 통째로 바뀌는 의사결정"을 피하라는 것이다. 정책 변화 자체는 통제 불가능하지만, 그 변화의 폭에 노출되지 않는 채널을 고를 수는 있다. 우리는 운 좋게 5월에 그 한 줄을 보고 결정을 바꿨고, 6월에 정책이 보류됐어도 결정은 옳았다.

기술적으로 한 가지 더 부수효과가 있었다. PTY + JSONL Watch 패턴은 Claude Code 자동화에 일반적으로 쓸 수 있는 패턴이다. 같은 패턴으로 CI에서 인터랙티브 Claude 세션을 모니터링하거나, IDE 확장에서 백그라운드 세션의 도구 호출을 가시화하거나, 팀 모드 게이트웨이에서 여러 세션의 turn을 한 곳에 모을 수도 있다. headless 한 줄로 끝났다면 보지 못했을 패턴이다.

## 정리

- `claude -p` headless는 (정책이 다시 시행되면) 구독의 별도 monthly credit를 소모한다 — 데스크톱 앱·자동화 시연·시연용 데모에 부적합한 채널이 될 수 있다.
- 인터랙티브 Claude Code는 종전대로 구독 usage limit 내 사용 — 채널로 안전하다.
- 인터랙티브를 무인처럼 운영하려면 두 조각 — `node-pty` spawn + `~/.claude/projects/<derived>/<UUID>.jsonl` 파일 watch.
- `AskUserQuestion` 차단 + defer system prompt + `--permission-mode dontAsk` + `--strict-mcp-config`로 통제권을 회수한다.
- 정책이 보류돼도 이 채널 선택은 옳았다. 정책 재시행에 노출되지 않고, 사용자 경험은 더 낫고, JSONL Watch는 그 자체로 재사용 가능한 패턴이다.

## 참고 자료

- [Use the Claude Agent SDK with your Claude plan](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan) — 2026-06-15 일시 보류 공지 포함
- [Claude Code CLI Reference](https://docs.claude.com/en/docs/claude-code) — `--session-id`·`--strict-mcp-config`·`--permission-mode` 등 플래그
- [node-pty](https://github.com/microsoft/node-pty) — Electron 35+ 환경에서 `electron-rebuild` 필수
