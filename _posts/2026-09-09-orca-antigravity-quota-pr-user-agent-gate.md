---
title: "오픈소스에 PR 하나 올리며 발견한 것 — User-Agent가 API 자격을 가르는 방식"
author: abruption
date: 2026-09-09 10:00:00 +0900
categories: [Programming, Open Source]
tags: [open-source, github, oauth, api, debugging, orca]
---

## 문제 — "Antigravity 사용량"이 Antigravity를 조회한 적이 없었다

평소 Claude Code와 Antigravity CLI를 둘 다 쓰는데, [Orca](https://github.com/stablyai/orca)(62.9k★, MIT — 여러 AI CLI의 사용량을 한 화면에 모아 보여주는 터미널 도구)에서 유독 Antigravity 사용량만 제대로 안 잡히는 게 불편했다. 마침 Antigravity 쿼터를 직접 조회하는 CLI(`agy-cli-usage`)를 따로 만들어 배포하고 있던 참이라, 그 지식을 그대로 옮겨서 기여해보고 싶었다.

코드를 열어보니 `antigravity-usage-mirror.ts`라는 29줄짜리 파일이, 실은 **Gemini Code Assist 조회 결과의 `provider` 문자열만 `'antigravity'`로 갈아끼우는 순수 변환 함수**였다. fetch도, 파일시스템 접근도, 키링 접근도 0건. Antigravity 항목에 뜨는 숫자는 전부 Gemini CLI 것이었다.

이게 두 가지 문제를 만든다. 첫째, 두 도구의 쿼터 풀이 다르면 표시되는 숫자가 그냥 틀린다. 둘째, Gemini CLI에 로그인하지 않은 사람은 Antigravity 항목이 영구히 `unavailable`로 뜬다.

이슈 [#11409](https://github.com/stablyai/orca/issues/11409)가 정확히 이 증상(SSH로 접속한 원격 dev box에서 사용량이 안 갱신됨)을 신고하고 있었고, 다른 이슈 [#9122](https://github.com/stablyai/orca/issues/9122)가 근본 원인(미러 구조)을 이미 짚어놓은 상태였다. 이미 경쟁 PR이 두 개(#11536, #16710) 있었지만 둘 다 로컬에서 실행 중인 `agy` 프로세스에 RPC로 붙는 방식이라, SSH로 접속한 원격 호스트 시나리오는 커버하지 못했다. 자격증명 파일을 직접 읽는 방식이면 원격 호스트까지 갈 수 있겠다고 판단해 [PR #19209](https://github.com/stablyai/orca/pull/19209)를 올렸다.

## Cloud Code API를 직접 두드려보니 나온 것 — User-Agent 게이트

Antigravity의 쿼터 조회는 결국 Google의 `cloudcode-pa.googleapis.com`(Cloud Code API)을 호출하는 일이다. 같은 OAuth 토큰으로 User-Agent만 바꿔가며 호출해보니 결과가 완전히 갈렸다.

| User-Agent | `cloudaicompanionProject` | tier | 쿼터 응답 |
|---|---|---|---|
| Antigravity 문자열 미포함 | 없음 | — | 도달 불가 |
| Electron 기본 UA | 있음 | `standard-tier` | 403 |
| `antigravity` 포함 | 있음 | `free-tier` | 200 |

같은 계정, 같은 토큰인데 UA에 따라 서버가 선언하는 tier 자격 자체가 달라진다. 요청 본문에 `ideType: "ANTIGRAVITY"`를 넣어도 소용없다 — **판별 축이 요청 본문이 아니라 User-Agent 헤더**라는 뜻이다. 거부 사유 코드도 `UNSUPPORTED_CLIENT`였다. "이 계정은 자격이 없다"가 아니라 "이 클라이언트는 지원 안 한다"는 것이다.

첫 커밋에는 이걸 "UA가 제품 자체를 판별하는 하드 게이트"라고 적었는데, 리뷰 코멘트에서 스스로 정정했다 — 이 관측은 한 계정 기준이고, Antigravity 단독 entitlement 계정이라면 UA와 무관하게 통과할 가능성이 남는다. `UNSUPPORTED_CLIENT`라는 사유 자체는 여전히 클라이언트 쪽을 가리키므로 완전한 무효화는 아니지만, "무조건 게이트"라는 최초 주장보다는 조심스러운 서술로 남겨야 했다.

덧붙이자면, 이 UA 문자열이 실제로 얼마나 예민한지 보여주는 사례가 하나 있었다. 별개로 관리하던 `agy-cli-usage`라는 CLI가 UA에서 `antigravity` 문자열을 실수로 빼는 커밋 이후 **66일간 API 경로가 조용히 죽어 있었다.** 옛 패키지 이름에 우연히 그 문자열이 들어 있었던 덕에 그때까지는 동작했던 것뿐이었다.

## 리뷰 대응 중 발견한 자기상쇄 버그

리뷰 대응으로 두 가지를 고쳤다.

- (a) 원격 로그인이 만료되거나 네트워크가 끊긴 상태(`stale-token`/`network`)를 미러 뒤로 숨기지 않고 표면화
- (b) 로컬에 `agy` 프로세스가 없어도 상태바가 사라지지 않도록 게이팅 조건 완화

문제는 이 둘을 합치면 서로를 죽인다는 것이었다. (a)로 표면화한 `stale-token`/`network` 상태를, 상태바 슬롯은 여전히 `status === 'ok'`일 때만 보여주고 있었다. 그 결과 원격 로그인이 만료된 사용자는 **상태바 자체가 사라져서, 정작 무엇을 해야 하는지 알려주는 메시지까지 함께 묻혔다.** 리뷰 봇(pullfrog)이 이걸 연달아 두 번 지적했고, 커밋 메시지에 그대로 남겼다: "Two fixes from the last round cancelled each other out."

수정 방향은 "건강한 상태인가"가 아니라 "이름 붙은 credential 출처가 있는가"로 판정 기준을 바꾸는 것이었다.

```ts
isStatusBarItemAvailable('antigravity', detectedAgentIds) ||
  Boolean(antigravity?.usageMetadata?.credentialSource)
```

`credentialSource`는 어느 호스트에서든 로그인을 한 번이라도 찾았을 때만 채워지므로, `ok`·`stale-token`·`network`를 전부 포괄한다. 반대로 `missing-credentials`(애초에 로그인 이력이 없는 새 머신)는 이 값을 남기지 않아 계속 숨겨진다 — 노이즈는 그대로 억제하면서, 조치가 필요한 상태만 드러나게 한 것이다.

## 경쟁 PR과 다른 길

#11536과 #16710은 로컬에서 실행 중인 `agy` 프로세스에 loopback RPC로 붙는 방식이다. 이 PR은 그 방식을 건드리지 않고, 자격증명 파일만 있으면 되는 별도 경로를 추가했다 — 프로세스가 로컬에 없어도 되고, SSH로 접속한 원격 호스트까지 갈 수 있다. 두 접근이 겹치지 않게 파일명도 경쟁 PR이 만드는 `antigravity-usage-fetcher.ts`와 의도적으로 다르게 골랐다.

## 지금 상태

PR은 열려 있고 머지 가능한 상태지만, 첫 기여자 PR이라 메인테이너가 워크플로 실행을 승인해야 실제 CI(lint·typecheck·test·build)가 돈다 — 아직 승인 전이다. 사람 리뷰어도 아직 배정되지 않았다. 같은 날 메인테이너들이 다른 PR 20개 이상을 활발히 처리하고 있는 걸 보면 이 PR만 특별히 막힌 건 아니고, 대형 저장소에서 흔한 표준 보호 장치에 걸려 순서를 기다리는 중이다.

같은 게이트 로직을 일반화하려는 별도 PR(#8064)도 진행 중이라, 그게 먼저 머지되면 그 인터페이스에 맞춰 리베이스할 계획을 PR에 미리 남겨뒀다.

## 참고 링크

- [Orca PR #19209](https://github.com/stablyai/orca/pull/19209)
- [Orca 저장소](https://github.com/stablyai/orca)
- [이슈 #11409](https://github.com/stablyai/orca/issues/11409) / [이슈 #9122](https://github.com/stablyai/orca/issues/9122)
