---
title: "OpenAI Decisions API로 서브에이전트 모델을 고르는 Aptly"
author: abruption
date: 2026-11-27 09:00:00 +0900
categories: [Programming, Open Source]
tags: [aptly, subagents, codex, claude-code, mcp]
description: "OpenAI Decisions API에 작업 요약과 후보를 보내 Claude Code와 Codex의 서브에이전트 모델·추론 강도를 추천하는 Aptly를 만들었습니다."
---

OpenAI Decisions API로 Claude Code와 Codex의 서브에이전트 모델과 추론 강도를 골라 주는 Aptly를 만들었습니다. README 번역 검토와 여러 파일에 걸친 변경 검토는 둘 다 ‘리뷰’입니다. 둘에 매번 같은 모델을 쓸 이유는 없어서, 작업 요약을 보고 미리 정해 둔 후보 중 어떤 조합이 적합한지 Decisions API에 묻기로 했습니다.

계기는 ChatGPT 요금제 혜택으로 받을 수 있는 API 크레딧이었습니다. 반복해서 쓰는 작업에 활용하고 싶었고, 서브에이전트를 띄울 때마다 하는 모델 선택이 그 후보였습니다. 대상 요금제에서 활성화한 크레딧은 연결된 API 조직에 들어갑니다. Aptly에서는 그 조직의 API 키로 Decisions 추천을 요청합니다. 이후 서브에이전트의 사용량은 호스트가 연결한 계정과 모델에 따릅니다. [OpenAI API 크레딧 안내](https://help.openai.com/en/articles/20001626-api-credits-benefit-for-eligible-chatgpt-subscriptions)

## Decisions API에 어떤 정보를 보내나요

Aptly의 프로필에는 모델과 추론 강도가 함께 들어 있습니다. 추천을 요청하면 Aptly는 작업 요약과 후보 프로필 정보를 OpenAI Decisions API에 보냅니다. 어떤 프로필을 쓸지 판단하는 데 드는 비용과, 추천받은 모델이 실제 작업을 하는 비용은 따로 봐야 합니다. 서브에이전트를 띄우는 건 Claude Code나 Codex가 담당합니다.

한 번씩 직접 추천받을 수도 있고, 호스트에 Hook을 등록해 서브에이전트를 띄우기 직전에 추천을 받도록 할 수도 있습니다. 추천만 보여 주는 모드와 조건을 충족할 때 호출 인자의 기본값을 채우는 모드를 뒀습니다. 모델과 추론 강도를 둘 다 명시한 Codex 호출은 그대로 둡니다. 부모 대화의 이력을 넘기는 호출도 이력을 보존하며, 해당 호출에 추천값을 적용할 수 있는지 확인되지 않으면 원래 인자를 유지합니다. 추천 결과가 나왔다는 것과 새 모델로 작업이 시작됐다는 것은 따로 확인해야 합니다.

## 작업명만 넘겼더니 세 번 다 멈췄습니다

2026년 11월 5일에는 [session-peer의 영어 README와 한국어 번역](https://github.com/abruption/session-peer/tree/6e9efda7588e1df638748eeb7ad1b2707e0a3bf4)을 비교하는 작업으로 시험했습니다. 작업명만 준 자동 Hook 시험에서는 세 번 모두 `needs_context`가 나왔습니다. 직접 추천 요청에 저장소와 파일 URL을 넣은 경우, 거기에 문서 요약까지 보탠 경우에는 각각 세 번 모두 Terra/high가 추천됐습니다. URL은 요약의 일부로 전달됩니다. Aptly가 링크를 열어 문서를 읽어 주지는 않습니다.

URL과 요약을 보낸 조건에서 추천받은 Terra/high로 호스트의 서브에이전트를 실행한 뒤, 앞서 같은 과제를 맡긴 Sol/high 결과와 비교했습니다. 조건별로 세 번씩 실행한 회당 평균입니다.

| Standard API 단가 환산 | Sol/high | Terra/high + 분류비 |
|---|---:|---:|
| 실제 캐시 사용량 반영 | $0.03229 | $0.01354 |
| 입력을 전부 비캐시로 환산 | $0.11254 | $0.05363 |

이 과제에서는 분류비를 넣고도 약 52~58% 차이가 났습니다. 호스트에 기록된 토큰을 공개 API 단가로 환산한 값이라 구독 실청구액은 알 수 없습니다. 메인 에이전트가 자료를 준비한 비용도 비교에서 빠져 있습니다. 한 과제의 결과로 리뷰 품질이 동등하다거나 다른 작업에서도 같은 폭으로 절감된다고 말하기는 어렵습니다. 작업명만으로는 추천을 보류했고, 문맥을 보탠 뒤에는 추천과 실제 실행을 따로 확인했다는 점이 이 시험에서 더 중요했습니다. [비용 실측 기록](https://github.com/abruption/aptly/blob/v0.1.22/docs/guide/ko/benchmarks.md)

## 자동으로 보낼 내용도 설정에서 고릅니다

Hook을 연결하면 어디까지 외부로 보내는지도 정해야 합니다. 자동 추천을 위한 전송 범위는 `none`, `description`, `excerpt` 중에서 선택합니다. 새 등록은 선택 전까지 자동 전송을 하지 않습니다. `description`은 호출자가 제공한 설명을 사용하고, `excerpt`는 지원하는 호출에서 작업 내용의 일부를 잘라 보낼 수 있습니다. 발췌는 익명화가 아니므로 업무 자료를 다룰 때는 작업명과 설명에도 어떤 내용이 들어 있는지 확인해야 합니다. 자동 전송을 꺼 두더라도 직접 추천을 요청할 때 작성한 요약은 API에 전달됩니다. [전송 범위 문서](https://github.com/abruption/aptly/blob/v0.1.22/docs/guide/ko/transmission.md)

API 크레딧을 쓸 반복 작업을 찾다가, 서브에이전트를 띄우기 직전의 모델 선택을 따로 떼어 보게 됐습니다. 다음 과제에서도 비용 차이를 보려면 추천에 얼마가 들었는지와 추천받은 모델이 실제로 얼마나 사용했는지를 각각 기록해야 합니다.

- [Aptly 저장소](https://github.com/abruption/aptly)
- [Aptly 0.1.22 릴리스](https://github.com/abruption/aptly/releases/tag/v0.1.22)
- [자동 추천과 적용 조건](https://github.com/abruption/aptly/blob/v0.1.22/docs/guide/ko/routing.md)
