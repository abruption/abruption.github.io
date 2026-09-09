---
title: "npm provenance publish가 4연패처럼 보였던 이유 — cc-usage-cli 배포 postmortem"
author: abruption
date: 2026-09-09 09:00:00 +0900
categories: [Programming, CLI]
tags: [npm, provenance, github-actions, release-please, ci-cd, open-source]
---

## 최초 배포가 이틀 밀렸다

`cc-usage-cli`(Claude Code 구독의 rate limit을 API 헤더에서 직접 읽어오는 CLI, [npm](https://www.npmjs.com/package/cc-usage-cli))의 첫 배포는 계획보다 이틀 늦었다. 그동안 두 가지 원인을 의심했는데, 둘 다 틀렸다. 실제 원인은 셋 중 어디에도 없었고, CI 화면은 그 진짜 원인을 이틀 동안 한 번도 보여주지 않았다.

## 잘못 짚은 가설 두 개

**첫 번째 가설: 24시간 unpublish 쿨다운.** npm은 같은 버전을 unpublish한 뒤 24시간 안에 재publish하는 걸 막는다. 실제로 초기 실패의 원인 중 하나이긴 했다. 하지만 쿨다운이 풀린 뒤에도 실패는 계속됐다.

**두 번째 가설: release-please 멱등성.** GitHub Actions의 `publish` 스텝을 release-please의 `release_created` 출력에 게이팅해뒀는데, GitHub 릴리스 `v1.1.0`이 이미 존재하는 상태라 release-please는 매번 "커밋 0건"으로 판단하고 `release_created`를 세우지 않았다. 그 결과 **publish 스텝이 매번 조용히 스킵**됐다. 워크플로는 계속 초록이었고, 그 안에 진짜 에러는 표면화될 기회조차 없었다.

이 두 번째 가설은 *메커니즘*은 정확히 맞혔다. 스킵되고 있다는 진단 자체는 옳았다. 문제는 "왜 스킵을 우회해도 여전히 실패하는가"까지 못 갔다는 것이다.

## 진짜 원인 — `package.json`에 한 줄이 없었다

`workflow_dispatch` 트리거를 추가해 release-please 게이트를 강제로 우회하고 나서야 처음으로 진짜 에러가 나왔다.

```
npm error code E422
Error verifying sigstore provenance bundle: Failed to validate repository information:
package.json: "repository.url" is "", expected to match "https://github.com/abruption/cc-usage-cli"
```

`npm publish --provenance`는 sigstore 서명이 증명하는 빌드 출처(어느 GitHub Actions 워크플로에서 왔는지)와 `package.json`의 `repository.url`이 일치하는지 검증한다. 이 필드가 비어 있으면 검증할 대상 자체가 없어 레지스트리가 거부한다.

자매 프로젝트인 `oci-cost-cli`와 `agy-cli-usage`에는 `repository`/`homepage`/`bugs` 세 필드가 전부 있었다. 이 저장소만 가장 나중에 스캐폴딩하면서 빠뜨린 것이었다.

## CI 화면이 "4연속 실패"로 보였던 이유

원인을 찾는 과정에서 같은 커밋으로 dispatch를 여러 번 돌렸는데, 실제 시퀀스는 이랬다.

| 시각(UTC) | 결과 | 의미 |
|---|---|---|
| 02:28:47 | **E422** | 진짜 원인. `repository` 없음 |
| 02:30:45 | **E409** | `Failed to save packument` — 일시적, unpublish tombstone 때문 |
| **02:31:35** | **✅ 성공** | `+ cc-usage-cli@1.1.0` |
| 02:32:36 | **E403** | "1.1.0 위에 덮어쓸 수 없다" |

CI 로그의 표면만 보면 실패·실패·실패·실패로 보인다. 하지만 실제로는 세 번째 시도에서 성공했다. E409는 50초쯤 지나면 저절로 풀리는 종류였고, 마지막 E403은 사실 성공의 증거였다 — "1.1.0 위에 못 올린다"는 말 자체가 1.1.0이 이미 올라가 있다는 뜻이니까.

이걸 CI 화면만 보고는 알 수 없다. `npm view`로 레지스트리를 직접 확인해야만 어느 시점에 실제로 게시됐는지 알 수 있었다. 여러 run이 뒤섞여 헷갈릴 때는 게시된 패키지의 **provenance attestation을 역추적**하면 확정할 수 있다 — attestation에 어느 run ID와 commit SHA가 게시했는지가 박혀 있다.

## 두 개의 교훈이 같은 사건에서 동시에 나왔다

- **"CI가 초록이라고 성공은 아니다"** — publish 스텝 자체가 스킵되고 있었다
- **"CI가 빨갛다고 실패는 아니다"** — 빨간 로그들 사이에 성공이 끼어 있었다

방향은 정반대지만 해법은 같다. **최종 산출물(이 경우 npm 레지스트리)을 직접 확인하는 것만이 판정 근거가 된다.** CI 상태 배지는 그 산출물에 대한 *주장*일 뿐, 산출물 자체가 아니다.

## 부수적으로 걸린 것 — 잘못 걸린 릴리스 PR

`repository` 필드를 추가한 커밋에 `fix:` 프리픽스를 썼더니, release-please가 즉시 `chore(main): release 1.1.1` PR을 자동 생성했다. 하지만 이 변경은 이미 1.1.0에 포함되어 배포된 뒤였으므로 머지할 이유가 없어 그냥 닫았다.

메타데이터만 고칠 때는 `chore:`나 `ci:` 프리픽스를 쓰면 이런 PR이 애초에 안 생긴다. Conventional Commits 프리픽스가 release-please에게는 "사용자에게 보이는 변경"이라는 신호로 읽힌다는 걸 새삼 확인한 셈이다.

## 참고 링크

- [cc-usage-cli (GitHub)](https://github.com/abruption/cc-usage-cli)
- [cc-usage-cli (npm)](https://www.npmjs.com/package/cc-usage-cli)
- [npm provenance 공식 문서](https://docs.npmjs.com/generating-provenance-statements)
