---
title: "Claude Code Hooks로 플러그인 자동화 파이프라인 만들기"
author: abruption
date: 2026-04-30 11:00:00 +0900
categories: [Programming, Claude Code]
tags: [claude-code, hooks, mcp, plugin, automation, jwt, authentication]
---

## Hooks라는 게 있더라

Claude Code 플러그인을 만들다 보면 "세션이 시작될 때 이것저것 자동으로 해줬으면" 하는 순간이 온다. JWT 토큰 갱신, 로컬 서버 기동, 설정 파일 검증 같은 것들. 매번 사용자한테 "먼저 이거 실행해주세요"라고 안내하는 건 좋은 경험이 아니다.

Claude Code에는 Hooks라는 기능이 있다. 특정 이벤트(세션 시작, 도구 호출 전후 등)에 셸 스크립트나 Node.js 스크립트를 걸 수 있는 구조인데, 이걸 잘 쓰면 플러그인 초기화를 완전 자동화할 수 있다.

내가 MCP 서버 기반 플러그인을 만들면서 SessionStart 훅 3개를 체이닝한 경험을 공유한다. 구체적인 프로젝트 코드는 생략하고 패턴 위주로 정리했다.

## hooks.json 기본 구조

플러그인 루트에 `hooks/hooks.json` 파일을 만들면 Claude Code가 자동으로 인식한다. SessionStart 이벤트에 여러 훅을 등록하면 배열 순서대로 실행된다.

```json
{
  "hooks": [
    {
      "event": "SessionStart",
      "command": "node ${CLAUDE_PLUGIN_ROOT}/hooks/step-1.mjs"
    },
    {
      "event": "SessionStart",
      "command": "node ${CLAUDE_PLUGIN_ROOT}/hooks/step-2.mjs"
    },
    {
      "event": "SessionStart",
      "command": "node ${CLAUDE_PLUGIN_ROOT}/hooks/step-3.mjs"
    }
  ]
}
```

`${CLAUDE_PLUGIN_ROOT}`는 Claude Code가 자동으로 주입하는 환경변수로, 플러그인이 설치된 경로를 가리킨다. 훅 스크립트 안에서 플러그인의 다른 파일을 참조할 때 유용하다.

## 내가 만든 3-Hook 체인

### Hook 1: 설정 파일 동기화

첫 번째 훅은 가볍게 시작한다. 플러그인이 제공하는 규칙 파일(자연어 라우팅 매핑 같은 것)을 프로젝트의 `.claude/rules/` 디렉토리로 복사하는 역할이다.

왜 이게 필요하냐면, Claude Code에서 사용자가 "이거 분석해줘"라고 말했을 때 어떤 스킬을 호출할지 결정하는 라우팅 규칙이 프로젝트 로컬에 있어야 하기 때문이다. 플러그인 자체에 규칙 파일이 있어도 Claude Code는 프로젝트의 `.claude/rules/`를 우선 참조한다.

이 훅은 단순 파일 복사라서 실패할 일이 거의 없고, 실행 시간도 수십 밀리초 수준이다. 체인의 첫 번째에 두는 이유는 나머지 훅이 실패하더라도 최소한 라우팅은 동작하게 하기 위해서다.

### Hook 2: JWT 인증 수명주기 관리

이게 3개 중 가장 복잡한 훅이다. MCP 서버가 인증이 필요한 API를 호출하는 경우, 세션 시작 시점에 토큰이 유효한지 확인하고 갱신하는 작업을 자동화한다.

흐름을 정리하면 이렇다.

```
JWT 파일 존재?
├─ 없음 + 자동 로그인 활성화 → CLI 로그인 명령 실행 (브라우저 열림)
├─ 없음 + 자동 로그인 비활성화 → 경고 메시지만 출력
└─ 있음 → 파싱
    ├─ 만료됨 → plugin.json에 INVALID 마커 기록 (fail-fast)
    ├─ 24시간 내 만료 → 경고 + 유효 토큰으로 진행
    └─ 유효 → plugin.json의 Authorization 헤더 갱신
```

여기서 핵심 설계 결정이 두 가지 있었다.

첫째, **Fail-fast 마커**. 토큰이 만료되었을 때 단순히 에러를 던지는 대신, `plugin.json`의 Authorization 헤더에 `Bearer INVALID_JWT_EXPIRED` 같은 마커를 기록한다. MCP 서버가 이 마커를 보면 즉시 인증 실패 응답을 보내고, Claude Code가 사용자에게 "재로그인이 필요합니다"라는 안내를 할 수 있다. 에러를 삼키지 않으면서도 사용자 경험을 해치지 않는 방법이다.

둘째, **자동 로그인**. JWT가 아예 없는 경우(첫 사용이거나 토큰이 삭제된 경우) CLI 로그인 명령을 자동으로 실행한다. 로컬 루프백 서버를 띄우고 브라우저를 열어서 OAuth 인증을 수행하는 방식인데, 타임아웃을 10분으로 잡아두고 blocking으로 실행한다. 세션 시작이 좀 느려지지만, 한 번 로그인하면 토큰이 7일간 유효하니 대부분의 세션에서는 파일 존재 확인 → 파싱 → 헤더 갱신으로 끝난다.

### Hook 3: HTTP Gateway 자동 기동

MCP 서버를 stdio 모드뿐 아니라 HTTP 모드로도 운영하고 싶을 때 쓴다. 내 경우 외부 서비스와의 연동이나 Tailscale을 통한 원격 접근을 위해 HTTP Gateway를 별도로 띄우고 있다.

이 훅이 해결하는 문제는 포트 충돌이다. 이전 세션에서 Gateway가 이미 떠 있을 수 있고, 아예 다른 프로세스가 같은 포트를 점유하고 있을 수도 있다.

```
설정 파일에 원격 Gateway URL 있음?
├─ 있음 → 로컬 기동 건너뜀 (원격 모드)
└─ 없음 → 로컬 모드
    ├─ lsof로 포트 점유 확인
    │   ├─ 내 Gateway가 이미 떠 있음 → PID 동기화, 기동 생략
    │   ├─ 다른 프로세스 → 경고, 기동 생략
    │   └─ 비어 있음 → detached 프로세스로 기동
    └─ PID 파일에 기록
```

초기에는 PID 파일만으로 판단했는데, 프로세스가 비정상 종료하면 stale PID 파일이 남아서 문제가 됐다. 결국 `lsof`로 실제 포트 점유자를 확인하고, 프로세스 커맨드 라인까지 비교해서 "내 Gateway인지" 판별하는 로직을 추가했다. 이게 의외로 까다로운 부분이었다.

## 훅 체인 설계에서 배운 것들

### 순서가 중요하다

3개의 훅이 순서대로 실행되기 때문에, 의존 관계를 고려해야 한다. 내 경우 Hook 3(Gateway)이 Hook 2(JWT)에 의존한다. Gateway가 HTTP 요청을 받을 때 JWT로 인증하기 때문이다. 그래서 JWT 갱신이 먼저 완료되어야 Gateway가 올바른 토큰으로 기동할 수 있다.

반면 Hook 1(설정 동기화)은 나머지와 독립적이라 순서가 바뀌어도 문제없다. 하지만 가장 가볍고 실패 확률이 낮은 작업을 앞에 두면 "최소한 이건 된다"는 보장이 생긴다.

### plugin.json을 런타임 상태 저장소로 쓸 수 있다

Claude Code 플러그인의 `plugin.json`은 단순 설정 파일이 아니다. 훅에서 이 파일의 특정 필드를 갱신하면, MCP 서버가 다음 요청에서 갱신된 값을 읽을 수 있다. 나는 Authorization 헤더와 MCP 서버 URL을 이 방식으로 관리하고 있다.

별도의 상태 파일을 만들 수도 있지만, `plugin.json`을 쓰면 Claude Code가 이미 파싱하는 파일에 정보가 모이니까 관리 포인트가 줄어든다.

### 워크스페이스 자동 초기화

플러그인을 처음 쓰는 사용자의 프로젝트에 설정 파일이 없을 수 있다. 이걸 매번 "먼저 `init` 명령을 실행하세요"라고 안내하는 건 번거롭다. 대신 훅에서 설정 파일의 존재 여부를 확인하고, 없으면 템플릿을 복사해주는 bootstrap 로직을 넣었다.

다만 `.git` 디렉토리가 있는 경우에는 자동 생성을 건너뛴다. git 레포에 의도치 않게 설정 파일이 커밋되는 걸 방지하기 위해서다. 이런 경우에는 사용자가 직접 `init` 명령을 실행하도록 안내 메시지만 출력한다.

### 환경변수로 동작 제어

자동 로그인이나 자동 초기화가 항상 바람직한 건 아니다. CI 환경이나 특수한 설정에서는 이런 자동화를 끄고 싶을 수 있다. 그래서 각 자동화 기능에 환경변수 토글을 뒀다.

```bash
MY_PLUGIN_AUTO_LOGIN=0    # 자동 로그인 비활성화
MY_PLUGIN_AUTO_INIT=0     # 설정 파일 자동 생성 비활성화
MY_PLUGIN_GATEWAY_DISABLE=1  # Gateway 자동 기동 비활성화
```

기본값은 전부 "켜짐"이지만, 명시적으로 끌 수 있게 하면 예상치 못한 환경에서의 문제를 줄일 수 있다.

## 원격 접근까지 확장한 이야기

Gateway를 로컬에만 띄우면 해당 머신에서만 접근할 수 있다. 그런데 iPad에서 원격으로 작업하거나, 다른 서버에서 MCP 도구를 호출하고 싶은 경우가 생겼다.

이건 설정 파일에 `gateway.url` 필드를 추가하는 것으로 해결했다. 이 필드가 있으면 Hook 3이 로컬 기동을 건너뛰고, `plugin.json`의 MCP 서버 URL을 원격 주소로 갱신한다. Tailscale 같은 VPN을 쓰면 인터넷에 노출하지 않으면서도 안전하게 접근할 수 있다.

```json
{
  "gateway": {
    "url": "http://my-server:3033"
  }
}
```

이 한 줄로 로컬 모드 ↔ 원격 모드 전환이 되니까, 집 맥미니에 Gateway를 띄워두고 외출 시 iPad에서 접근하는 식의 운용이 가능해졌다.

## 마무리

Hooks는 플러그인의 "보이지 않는 인프라"다. 잘 만들어두면 사용자가 의식하지 않아도 인증·초기화·서버 기동이 자동으로 처리된다. 처음 설계할 때 시간이 좀 들지만, 한 번 안정화되면 이후의 유지보수 비용이 확연히 줄어든다.

핵심 교훈 세 가지를 정리하면 이렇다.

1. 훅 체인은 의존 관계 순서로 배치하되, 독립적이고 가벼운 작업을 앞에 둔다
2. Fail-fast 마커로 에러를 전파하되, 사용자 경험은 보존한다
3. 모든 자동화에 환경변수 토글을 두어 선택적으로 비활성화할 수 있게 한다

## 참고

- [Claude Code Hooks Documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) — Hooks 공식 문서
- [Model Context Protocol Specification](https://modelcontextprotocol.io/) — MCP 프로토콜 스펙
- [Building Claude Code Extensions](https://docs.anthropic.com/en/docs/claude-code/extensions) — 플러그인 개발 가이드
