---
title: "오픈소스에 PR 하나 올리며 발견한 것 — User-Agent가 API 자격을 가르는 방식"
author: abruption
date: 2026-09-09 10:00:00 +0900
updated: 2026-09-24
categories: [Programming, Open Source]
tags: [open-source, github, oauth, api, debugging, orca]
---

처음 이 글을 쓴 건 9월 9일이었다. Orca에서 Antigravity 사용량만 제대로 잡히지 않는 게 답답해서, 따로 만들어 쓰던 `agy-cli-usage`의 조회 로직을 Orca에 옮겨보려고 했다. 그때 올린 [PR #19209](https://github.com/stablyai/orca/pull/19209)가 이 글의 출발점이다.

다만 그 뒤에 구현 방향과 PR 스택이 꽤 바뀌었다. 아래에서 당시의 발견과 2026년 9월 24일 현재 상태를 섞지 않으려고 나눠 적었다.

> **2026-09-24 KST 업데이트**
>
> 기능 자체는 구현됐지만 아직 Orca `main`에 들어간 건 아니다. 현재 main 반영을 추적할 PR은 [#22055](https://github.com/stablyai/orca/pull/22055)이고, 여전히 `OPEN`이다. 9월 24일 직접 확인한 GraphQL 응답은 `MERGEABLE` / `UNSTABLE`이었지만, 이걸 병합 승인이나 CI 통과로 읽으면 안 된다. 현재 HEAD에서 확인된 성공 체크는 Pullfrog뿐이고, `PR Checks`와 `PR test LoC`는 메인테이너 승인 대기 중이다.
>
> 그리고 이 작업에서 내가 한 버전 호환성·토큰 소비 검증은 #22055 본문에 공개적으로 출처가 남았다. 반면 기능 커밋에 내가 authored/co-authored로 들어간 건 아니어서 GitHub 코드 Contributor 등재 목표는 여기서 종결했다. 기능이 언제 main에 들어가는지 추적하는 일과 개인 기여 목표는 별개다.

## 문제 — "Antigravity 사용량"이 Antigravity를 조회한 적이 없었다

평소 Claude Code와 Antigravity CLI를 둘 다 쓰는데, [Orca](https://github.com/stablyai/orca)(여러 AI CLI의 사용량을 한 화면에 모아 보여주는 터미널 도구)에서 유독 Antigravity 사용량만 제대로 안 잡히는 게 불편했다. 마침 Antigravity 쿼터를 직접 조회하는 CLI(`agy-cli-usage`)를 따로 만들어 배포하고 있던 참이라, 그 지식을 그대로 옮겨서 기여해보고 싶었다.

코드를 열어보니 `antigravity-usage-mirror.ts`라는 29줄짜리 파일이, 실은 **Gemini Code Assist 조회 결과의 `provider` 문자열만 `'antigravity'`로 갈아끼우는 순수 변환 함수**였다. fetch도, 파일시스템 접근도, 키링 접근도 0건. Antigravity 항목에 뜨는 숫자는 전부 Gemini CLI 것이었다.

이게 두 가지 문제를 만든다. 첫째, 두 도구의 쿼터 풀이 다르면 표시되는 숫자가 그냥 틀린다. 둘째, Gemini CLI에 로그인하지 않은 사람은 Antigravity 항목이 영구히 `unavailable`로 뜬다.

이슈 [#11409](https://github.com/stablyai/orca/issues/11409)가 정확히 이 증상(SSH로 접속한 원격 dev box에서 사용량이 안 갱신됨)을 신고하고 있었고, [#9122](https://github.com/stablyai/orca/issues/9122)가 근본 원인(미러 구조)을 이미 짚어놓은 상태였다. 당시 경쟁 PR도 있었지만 로컬에서 실행 중인 `agy` 프로세스에 RPC로 붙는 방식이라, SSH로 접속한 원격 호스트 시나리오는 커버하지 못했다. 자격증명 파일을 직접 읽는 방식이면 원격 호스트까지 갈 수 있겠다고 판단해 #19209를 올렸다.

## Cloud Code API를 직접 두드려보니 나온 것 — User-Agent 게이트

Antigravity의 쿼터 조회는 결국 Google의 `cloudcode-pa.googleapis.com`(Cloud Code API)을 호출하는 일이다. 같은 토큰으로 User-Agent만 바꿔가며 호출해보니 결과가 완전히 갈렸다.

| User-Agent | `cloudaicompanionProject` | tier | 쿼터 응답 |
|---|---|---|---|
| `antigravity-usage-monitor/0.1 linux/arm64` | 있음 | `free-tier` | 200, groups=2 |
| `agy-cli-usage/0.4.5 linux/arm64` | 없음 | — | `UNSUPPORTED_CLIENT`, 도달 불가 |
| `curl/8.7.1` | 없음 | — | `UNSUPPORTED_CLIENT`, 도달 불가 |

같은 계정·같은 토큰인데 UA에 따라 서버가 선언하는 tier 자격 자체가 달라진다. 요청 본문에 `ideType: "ANTIGRAVITY"`를 넣어도 소용없다. `allowedTiers`와 `ineligibleTiers`를 보면 서버가 계정이 아니라 클라이언트를 보고 `free-tier`를 허용하거나 `UNSUPPORTED_CLIENT`로 거절하는 모양새다.

처음에는 “Antigravity 단독 entitlement 계정이라면 UA와 무관하게 통과할 수도 있다”는 여지를 크게 남겨뒀다. 지금도 표본이 완전한 증명은 아니다. 다만 KR의 별도 머신·네트워크·credential에서 재현했고, 예전에 반례라고 생각했던 KR 측정도 실제로는 v0.4.1 배포본이 `antigravity`가 들어간 UA를 보내고 있었던 것으로 정정됐다. 그래서 현재 데이터로는 **UA가 강한 클라이언트 게이트라는 해석이 더 자연스럽다**고 보는 편이 맞다.

덧붙이자면, 이 UA 문자열이 실제로 얼마나 예민한지 보여주는 사례가 하나 있었다. 별개로 관리하던 `agy-cli-usage`가 UA에서 `antigravity` 문자열을 실수로 빼는 커밋 이후 **66일간 API 경로가 조용히 죽어 있었다.** 옛 패키지 이름에 우연히 그 문자열이 들어 있었던 덕에 그때까지는 동작했던 것뿐이었다.

## 리뷰 대응 중 발견한 자기상쇄 버그, 그리고 아직 남은 문제

당시 리뷰 과정에서는 두 가지를 고쳤다.

- 원격 로그인이 만료되거나 네트워크가 끊긴 상태를 `stale-token`·`network`로 표면화
- 로컬에 `agy` 프로세스가 없어도 credential 출처가 있으면 상태바가 사라지지 않도록 게이팅 조건 완화

문제는 이 둘을 합치면 서로를 죽인다는 것이었다. 표면화한 상태를 상태바 슬롯은 여전히 `status === 'ok'`일 때만 보여주고 있었다. 그 결과 원격 로그인이 만료된 사용자는 상태바 자체가 사라져서, 정작 무엇을 해야 하는지 알려주는 메시지까지 함께 묻혔다. 수정 방향은 “건강한 상태인가”가 아니라 “이름 붙은 credential 출처가 있는가”로 판정 기준을 바꾸는 것이었다.

그렇다고 **현재 main 반영 후보의 인증 오류 분류까지 해결됐다고 보면 안 된다.** CodeRabbit이 지적한 “만료된 Agy 로그인 실패가 사용 불가 원인을 설명하지 않는다”는 문제를 `faa0069`의 fetcher에서 다시 확인해보면, `/usage`의 모든 non-zero 종료가 일반 오류 문구와 `failureKind: unknown`으로 매핑된다. 만료·인증 오류를 따로 분류하는 로직은 확인되지 않았다. 따라서 이 문제는 아직 미해결이거나 메인테이너가 수용 여부를 판단한 상태로만 봐야 한다. 다른 finding이 닫혔거나 Pullfrog이 “No new issues found”를 냈다는 이유로 고쳐졌다고 추론하면 안 된다.

## 경쟁 PR과 후속 구현은 어떻게 이어졌나

#11536과 #16710은 로컬에서 실행 중인 `agy` 프로세스에 loopback RPC로 붙는 방식이다. 이후 이어진 native CLI 방식은 이 경로를 건드리지 않고, 자격증명 파일과 `agy /usage`를 직접 사용하는 별도 경로를 추가했다. 실행 중인 프로세스가 없어도 되고, SSH로 접속한 원격 호스트까지 다룰 수 있다는 점이 이 접근의 장점이다.

Agy 버전 차이도 중요했다. 1.1.10의 `agy -p /usage --output-format json`은 read-only 조회처럼 보이지만 실제로는 일반 agent turn을 실행했고(`num_turns=1`, `total_tokens=3181`), 1.1.11 이상은 read-only slash command로 `num_turns=0`, 토큰 0과 native quota groups를 반환했다. 그래서 안전한 구현은 호출한 뒤 결과를 보고 실패시키는 게 아니라, 호출 전에 지원 버전을 확인하고 버전 미확인·구버전·판별 불가면 fail-closed하는 쪽이다.

## 2026-09-24 현재 — 구현 완료와 main landing은 별개다

후속 스택의 `merged` 표시는 모두 어느 브랜치에 병합됐는지를 같이 봐야 한다.

| PR | 2026-09-24 기준 상태 |
|---|---|
| [#21654](https://github.com/stablyai/orca/pull/21654) | `nwparker/agy-manual-command` feature base에 병합. `main` 병합 아님 |
| [#21678](https://github.com/stablyai/orca/pull/21678) | `nwparker/agy-worker-models` feature base에 병합. `main` 병합 아님 |
| [#21653](https://github.com/stablyai/orca/pull/21653) | `OPEN` Draft. 상위 스택도 아직 `main`에 반영되지 않음 |
| [#20797](https://github.com/stablyai/orca/pull/20797) | `OPEN`. 새 native CLI 구현으로 대체됐지만 아직 종결된 것으로 쓰면 안 됨 |
| [#16710](https://github.com/stablyai/orca/pull/16710), [#20933](https://github.com/stablyai/orca/pull/20933) | 둘 다 `OPEN`. 현재 native CLI 접근으로 대체된 이전 구현 PR |
| [#22055](https://github.com/stablyai/orca/pull/22055) | `OPEN`, base `main`, non-draft, HEAD `faa0069d4aafea5155f3b120197c6f62a1a55b08` |

#22055는 실제 upstream landing PR이지만 아직 main에 들어가지 않았다. 9월 24일 KST 직접 조회에서 GraphQL은 `MERGEABLE` / `UNSTABLE`을 반환했다. `PR Checks`와 `PR test LoC`는 메인테이너 workflow 승인 전이라 `action_required` 상태이고, 사람 승인도 아직 없다. 보류된 run `35640662382`는 예전 HEAD `e5023282cb66f955939e661e9d881dd10ecf5661`에 고정돼 있다. 따라서 그 run은 현재 `faa0069`에 대한 CI 증거가 아니다. 현재 HEAD에서 성공한 체크로 확인되는 것은 Pullfrog뿐이다. 이를 두고 PR이 병합됐거나 CI가 통과했다고 쓰면 안 된다.

기능 상태와 개인 기여 성과도 분리해서 봐야 한다. Agy 1.1.10의 버전 호환성·토큰 소비 문제를 확인한 내용과 리뷰 기여는 #22055 본문에서 `@abruption`의 finding으로 공개적으로 출처가 표시됐다. 이 부분은 검증·리뷰 기여로 인정된 셈이다.

반면 #22055의 작성자는 `werlang`이고, quota 구현 커밋에 `@abruption` authored/co-authored 메타데이터는 없다. 따라서 GitHub 코드 Contributor로 등재되는 개인 목표는 달성되지 않았고 여기서 종결했다. 앞으로 #22055는 기능이 `main`에 landing되는지만 추적한다. 그 PR의 병합 여부를 개인 Contributor 목표와 다시 연결하지 않는다.

## 별도로 진행한 Orca 기여의 9월 24일 스냅샷

아래 두 PR은 quota 구현과 같은 작업은 아니지만, 같은 시기에 진행한 Orca 기여라 현재 상태를 함께 남겨둔다.

| PR | 현재 상태 |
|---|---|
| [#22112](https://github.com/stablyai/orca/pull/22112) | 내가 작성한 PR. `OPEN`, non-draft/ready for review, HEAD `84472764dd5a5d90ae7a2bdcad5eb010588f84e7`. `track-community-pr`와 Pullfrog 두 자동 체크는 성공했지만 사람 메인테이너 리뷰는 없음 |
| [#22115](https://github.com/stablyai/orca/pull/22115) | 내가 작성한 PR. `OPEN` Draft, HEAD `da498b65a840bc202e0e6d9df512d12ef1654e86`. 체크·리뷰·코멘트 없음 |

두 PR 모두 2026년 9월 24일 KST 직접 조회에서 GraphQL `MERGEABLE` / `UNSTABLE`로 관찰됐다. 이전에는 `UNKNOWN`으로 관찰된 적도 있어서, 이 값은 조회 시각이 붙은 스냅샷일 뿐 병합 승인으로 해석하지 않는다. 둘 다 아직 병합되지 않았다.

연결 이슈 [#22110](https://github.com/stablyai/orca/issues/22110)은 `OPEN`이고 `AmethystLiang`에게 할당돼 있다. 9월 22일에 확인한 마지막 메인테이너 코멘트는 “thanks for reporting. looking into it”이었다.

## 정리

이 글에서 지금 확실하게 말할 수 있는 건 네 가지다. 직접 quota 조회 구현은 기술적으로 끝났고, main 반영은 아직 기다리는 중이다. #22055의 현재 상태는 `OPEN`·`MERGEABLE`/`UNSTABLE`이지 merged나 CI-passing이 아니다. 내 버전 호환성 검증·리뷰 기여는 공개적으로 인정됐지만 code Contributor 등재 목표는 끝났다. 그리고 만료 로그인 오류를 auth-specific하게 분류하는 문제는 아직 해결됐다고 단정할 수 없다.

## 참고 링크

- [Orca PR #19209](https://github.com/stablyai/orca/pull/19209) — 이 글의 역사적 출발점
- [Orca PR #22055](https://github.com/stablyai/orca/pull/22055) — native quota 기능의 main landing 추적
- [Orca PR #21654](https://github.com/stablyai/orca/pull/21654) / [#21678](https://github.com/stablyai/orca/pull/21678) — feature-stack 후속 구현
- [Orca PR #22112](https://github.com/stablyai/orca/pull/22112) / [#22115](https://github.com/stablyai/orca/pull/22115) — 별도 Orca 기여
- [Orca 이슈 #11409](https://github.com/stablyai/orca/issues/11409) / [#9122](https://github.com/stablyai/orca/issues/9122)
- [Orca 저장소](https://github.com/stablyai/orca)
