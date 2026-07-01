---
title: "실전에서 겪은 불편함으로 CLI 두 개 만들기 — agy-cli-usage와 oci-cost-cli"
author: abruption
date: 2026-07-01 10:00:00 +0900
categories: [Programming, CLI]
tags: [cli, npm, oci, oracle-cloud, node.js, typescript, release-please, keyring, open-source]
---

## 두 CLI 모두 "지금 당장 불편해서" 시작했다

오픈소스 도구를 만들 때 제일 오래가는 동기는 "언젠가 누가 쓸 것 같아서"가 아니라 "지금 내가 매번 이걸 손으로 하고 있어서"다. `agy-cli-usage`와 `oci-cost-cli` 둘 다 그런 식으로 시작했다.

- **agy-cli-usage**: Antigravity CLI(`agy`)의 `/usage` 패널은 인터랙티브 TUI에서만 볼 수 있다. 헤드리스로 띄운 서버 프로세스가 "지금 쿼타 얼마나 남았는지" 알아야 하는데, 물어볼 방법이 없었다.
- **oci-cost-cli**: Oracle Cloud의 Usage API가 반환하는 JSON은 항목 하나당 필드가 20개 가까이 되는데 대부분 null이고, 같은 SKU가 통화별로 중복돼서 나온다. 한국 리전과 미국 리전 두 개의 Always Free 테넌시를 매번 따로 조회하고 손으로 합산하는 게 일이었다. AWS에는 `aws-cost-cli` 같은 커뮤니티 도구가 있는데 OCI에는 공식 SDK 원시 클라이언트와 스타 0개짜리 데모 스크립트 두 개뿐이었다.

둘 다 npm에 공개했고([`agy-cli-usage`](https://www.npmjs.com/package/agy-cli-usage), [`oci-cost-cli`](https://www.npmjs.com/package/oci-cost-cli)), 만들면서 반복해서 부딪힌 설계 문제와 배포 삽질이 겹쳤다. 이 글은 그 교집합을 정리한 것이다.

## agy-cli-usage — TUI 뒤에 숨은 API를 직접 호출하기

`agy -p`(headless 모드)는 슬래시 커맨드를 렌더링하지 않는다. `/usage`를 쳐도 아무 일도 안 일어난다. 그래서 이 도구는 3단계 폴백으로 쿼타 정보를 가져온다.

```
1순위: OS 키링에서 OAuth 토큰 읽기
       → Cloud Code 내부 API(loadCodeAssist → retrieveUserQuotaSummary) 직접 호출
2순위: 키링이 없는 헤드리스 Linux
       → ~/.gemini/antigravity-cli/antigravity-oauth-token (0600) 파일에서 토큰 읽기
3순위: 위가 모두 실패
       → agy를 PTY로 띄워 /usage를 보내고 @xterm/headless로 패널을 파싱
```

1순위와 2순위는 사실상 같은 데이터를 다른 경로로 읽는 것뿐이라 어렵지 않다. 재미있는 건 3순위다. `agy`가 내부적으로 어떤 엔드포인트를 호출하는지 문서화되어 있지 않으니, 최후의 안전망은 "사람이 보는 화면을 그대로 흉내 내서 파싱하는 것"이다. 가상 터미널에 `agy`를 띄우고, `/usage`를 입력하고, 렌더링된 패널을 문자열로 재구성해서 정규식으로 뜯는다. 비공개 API가 스키마를 바꿔도 화면에 나오는 숫자 형식은 잘 안 바뀐다는 가정에 기대는 방식인데, 지금까지는 잘 버티고 있다.

쿼타는 개별 모델이 아니라 **모델 그룹 단위**로 매겨진다는 것도 API를 직접 두드려보고서야 알았다. Gemini Flash/Pro가 한 그룹, Claude Opus/Sonnet과 GPT-OSS가 한 그룹으로 묶여서 주간·5시간 두 종류의 버킷을 공유한다.

```jsonc
{
  "account", "tier", "fetchedAt", "source": "api|pty",
  "groups": [
    { "name": "CLAUDE AND GPT MODELS", "models": [...],
      "buckets": [
        { "kind": "weekly", "remainingFraction": 0.62, "resetAt": "..." },
        { "kind": "5h", "remainingFraction": 0.31, "resetAt": "..." }
      ] }
  ]
}
```

실제로 이 도구를 사내 다른 프로젝트(대시보드형 서비스)의 `/api/quota` 엔드포인트가 그대로 소비하고 있다. 서버가 `agy-cli-usage --json`을 5분 캐시로 실행하고, 그룹별 quotaId를 모델 하나하나에 매핑해서 UI에 미니바로 보여준다. 여기서 한 번 걸린 건 **파일 폴백의 소유권 문제**였다. 토큰 파일이 특정 사용자 소유(0600)라서, 서버 프로세스가 다른 사용자 권한으로 돌면 파일이 있어도 못 읽는다. "왜 됐다 안 됐다 하지" 하고 삽질하다가 `ps aux`로 프로세스 소유자를 확인하고 나서야 원인을 찾았다 — 헤드리스 자동화에서 흔한, 그러나 로그만 봐서는 절대 안 보이는 종류의 버그다.

## oci-cost-cli — SDK 없이 서명부터 직접 구현하기

OCI Usage API를 부르려면 공식 SDK를 쓰는 게 정석이다. 그런데 SDK를 통째로 의존성에 넣기엔 이 CLI가 하는 일이 "숫자 몇 개 조회해서 표로 찍는 것"뿐이라 너무 무거웠다. 그래서 **OCI API Signature Version 1**을 Node 표준 라이브러리(`node:crypto`, `node:https`)만으로 직접 구현했다.

서명 문자열은 `(request-target)`, `date`, `host`, `content-length`, `content-type`, `x-content-sha256` 순서로 조립하고, `~/.oci/config`에 있는 개인키로 RSA-SHA256 서명한다. 이 부분은 `~/.oci/config`를 읽을 수 있는 임의의 REST 호출에 재사용 가능하도록 `signer.ts`라는 별도 서브패스(`oci-cost-cli/signer`)로 분리해뒀다.

```ts
export function signRequest(
  profile: Profile,
  privateKeyPem: string,
  req: { method: string; host: string; path: string; date: string; body?: string },
) {
  const signingString = buildSigningString(req) // (request-target)\ndate: ...\nhost: ...
  const signature = createSign('RSA-SHA256').update(signingString).sign(privateKeyPem, 'base64')
  return `Signature version="1",keyId="${profile.tenancy}/${profile.user}/${profile.fingerprint}",...`
}
```

여기서부터가 진짜 문제였다. Usage API 응답을 실제로 받아보면 이런 식이다.

- 같은 SKU가 **여러 통화로 중복** 등장한다. 특히 $0짜리 Free Tier 항목은 SGD로 한 번, USD로 한 번 찍히는 경우가 있어서 단순히 다 더하면 두 배로 계산된다.
- Cost API는 실패해도 **에러를 던지지 않고 빈 배열을 조용히 반환**한다. Usage API는 실패하면 명확히 throw하는데, Cost API만 이렇게 다르게 동작해서 "비용 0원"과 "조회 실패"를 구분 못 하면 실제로 돈이 나가고 있는데도 0원으로 표시될 수 있다.
- Free Tier 판별을 SKU 이름의 `- Free` 접미사로 하면 안 된다. 아웃바운드 트래픽처럼 Always Free 한도 내 사용량인데도 이 접미사가 안 붙는 SKU가 실제로 존재한다. (이건 유닛테스트를 짜다가 "이상하게 free-tier 프리셋이 정상 항목을 경고로 잡네"를 발견하고서야 알았다 — 실제 API 응답이 문서보다 정확한 스펙이다.)

집계 로직은 그래서 이렇게 정리됐다: **USD 우선, USD가 없을 때만 비USD 합산, 다른 통화끼리는 절대 섞어서 더하지 않는다. Cost API 실패는 `costApiFailed` 플래그로 명시적으로 경고한다. Free Tier 판별은 SKU 이름이 아니라 `cost > 0` 여부로만 한다.**

```bash
npx oci-cost-cli --preset free-tier
```
```text
▸ DEFAULT  ✅ all items within Free Tier
▸ US       ✅ all items within Free Tier
```

### 사람이 읽는 표와 에이전트가 읽는 JSON을 동시에

처음에는 사람이 보는 표만 있었는데, 자동화 스크립트에서 이 CLI를 호출하려니 `--json`이 필요해졌다. 그런데 "가공된 요약"과 "원본 API 응답 그대로"는 쓰임새가 다르다. 요약본은 사람이 신뢰할 수 있는 최종 숫자고, 원본은 "이 집계 로직이 왜 이렇게 판단했는지" 검증하고 싶을 때 필요하다. 둘 다 만들되 기본은 요약본으로 두고, 원본이 필요하면 `--raw`를 얹는 방식으로 타협했다.

```bash
npx oci-cost-cli --output json --raw --profile DEFAULT
```
```json
{
  "lineItems": [{ "service": "Compute", "cost": 0, "currency": "SGD", "isFreeTierSku": false }],
  "raw": {
    "usage": [{ "computedAmount": 0, "currency": null }],
    "cost":  [{ "computedAmount": 0, "currency": "SGD" }]
  }
}
```

`--raw`가 실제로 잡아낸 건, Cost API 응답 중 일부가 `currency`를 **빈 문자열**(null이 아니라)로 돌려주는 경우였다. 집계 레이어는 이걸 "비용 데이터 없음"으로 처리하는데, `raw`가 없으면 "왜 이 항목만 cost가 null이지?"를 사람이든 에이전트든 재현할 수가 없다. 부수효과가 있는 모든 명령(텔레그램 전송, 키링/파일 쓰기, crontab 수정)에는 `--dry-run`도 붙였다 — 자동화 스크립트나 AI 에이전트가 이 CLI를 호출할 때, 실제로 뭔가 바꾸기 전에 미리 결과를 볼 수 있어야 안심하고 붙일 수 있기 때문이다.

## 두 프로젝트에서 그대로 반복된 패턴 세 가지

### 1. 자격증명은 "OS 키링 우선, 파일은 0600 폴백"

`agy-cli-usage`가 OAuth 토큰을 다루는 방식과, `oci-cost-cli`가 텔레그램 봇 토큰을 다루는 방식이 똑같다. `@napi-rs/keyring`으로 OS 키링(macOS Keychain / Linux Secret Service / Windows Credential Manager)을 먼저 시도하고, 실패하면 `~/.config/<tool>/config.json`에 `0600`/`0700` 권한으로 저장한다.

SQLite나 자체 암호화는 일부러 배제했다. SQLite는 그 자체로 암호화가 아니라서 파일 하나 늘어나는 것 외에 이득이 없다. 자체 암호화(AES 등)는 무인 cron이 재부팅 후에도 복호화해야 하니 **복호화 키를 결국 디스크 어딘가에 평문으로 같이 둬야 한다** — "암호화했다"는 착시만 줄 뿐 실질 보안 이득이 없다. 진짜 보안은 OS 키링만 제공하고, 키링이 없는 환경(헤드리스 리눅스 cron이 대부분 이 경우다)에서는 `~/.aws/credentials`나 `~/.npmrc`와 같은 신뢰 모델 — 파일 권한 격리 — 로 충분하다고 판단했다.

### 2. release-please는 커밋 타입에 관대하지 않다

두 프로젝트 모두 [release-please](https://github.com/googleapis/release-please)로 배포를 완전 자동화했다. `feat:`/`fix:` 커밋이 쌓이면 버전 범프 PR을 자동으로 만들고, 그 PR을 머지하면 태그·GitHub Release·`npm publish`까지 알아서 돈다. 편한 만큼 함정도 명확했다.

- **Squash 커밋의 타입만 본다.** `develop` 브랜치를 `master`로 동기화하는 PR을 `chore: sync master with develop` 같은 제목으로 올리면, 그 안에 진짜 `feat:` 작업이 있어도 release-please는 그냥 지나친다. Conventional Commits 파서가 보는 건 squash 커밋 자체의 타입이지, 스쿼시되기 전 개별 커밋들이 아니기 때문이다. 이후로는 develop→master 동기화 PR 제목을 항상 그 안에서 가장 의미 있는 변경 타입(`feat:`/`fix:`)으로 붙이는 규칙을 정했다.
- **NPM_TOKEN은 2FA 우회 토큰이어야 한다.** 일반 granular 토큰으로는 `npm publish`가 `403 ... Two-factor authentication or granular access token with bypass 2fa enabled is required`로 막힌다. Classic "Automation" 토큰이거나, granular 토큰이면 "Bypass two-factor authentication" 옵션을 켜야 한다.
- **한 번 실패한 publish는 재실행해도 다시 안 돈다.** release-please 워크플로를 재실행하면 이미 릴리스/태그가 존재한다는 이유로 `release_created=false`를 반환하고, 그 뒤에 붙어있는 `npm ci`/`test`/`publish` 스텝이 통째로 스킵된다. 겉보기엔 "성공"처럼 초록불이 뜨지만 실제로는 아무것도 배포되지 않은 상태다. 이 문제는 두 가지 방식으로 풀었다. `agy-cli-usage`는 릴리스를 삭제하고(`gh release delete vX.Y.Z --cleanup-tag`) PR 라벨을 `autorelease: pending`으로 되돌린 뒤 워크플로를 재실행하는 복구 절차를 쓴다. `oci-cost-cli`는 아예 임의의 태그를 대상으로 `npm publish`만 독립적으로 수행하는 `workflow_dispatch` 트리거의 `publish.yml`을 별도로 만들어뒀다 — release-please의 생성 감지 로직에 기대지 않는 안전망이다.

### 3. "동작하는 것처럼 보이지만 아무 출력도 없는" 버그는 Node의 디테일에서 나온다

`oci-cost-cli`를 실제로 크론에 태울 때, macOS에서만 조용히 exit 1로 끝나고 아무 메시지도 안 남기는 문제가 있었다. 원인은 두 겹이었다.

첫째, Node의 dual-stack 연결(Happy Eyeballs)이 타임아웃되면 `AggregateError`를 던지는데, 이 에러는 **최상위 `.message`가 비어 있다.** 실제 원인은 `.errors[]` 배열 안에 있다. 에러 메시지를 그냥 `e.message`로만 찍던 코드는 빈 줄만 출력하고 있었다.

```ts
function errMessage(e: unknown): string {
  if (e instanceof AggregateError) {
    const inner = e.errors.map(errMessage).join('; ')
    return e.message || inner || e.constructor.name // .message가 비어있으면 안쪽을 본다
  }
  return e instanceof Error ? e.message || e.constructor.name : String(e)
}
```

둘째, "이 파일이 CLI로 직접 실행됐는지"를 판별하는 흔한 ESM 패턴 — `import.meta.url === pathToFileURL(process.argv[1]).href` — 이 심링크 환경에서 깨졌다. 개발 디렉터리 자체가 심링크였고(iCloud/원격 마운트 환경에서 흔하다), Node의 ESM 로더는 `import.meta.url`을 실제 경로로 풀어서 주는데 `path.resolve()`는 파일시스템을 건드리지 않으니 심링크를 그대로 남겨둔다. 두 값이 절대 같아질 수 없는 상황이었다. `realpathSync()`로 양쪽을 같은 기준으로 맞추고 나서야 상대경로/절대경로/심링크 세 가지 실행 방식 모두에서 정상 동작했다.

```ts
function isMainModule(): boolean {
  if (!process.argv[1]) return false
  try {
    return fileURLToPath(import.meta.url) === realpathSync(process.argv[1])
  } catch {
    return false
  }
}
```

이런 종류의 버그는 로컬에서는 절대 재현이 안 되고, 실제 배포 환경(다른 OS, 심링크 마운트, 크론)에서만 튀어나온다. 그래서 "로컬에서 테스트 통과했다"와 "실제로 동작한다" 사이의 간극을 좁히려면, 최소한 한 번은 진짜 배포 대상 환경에 SSH로 들어가서 `npx`로 직접 돌려보는 과정을 생략하면 안 된다는 걸 다시 확인했다.

## 정리

두 CLI는 다루는 도메인이 완전히 다르지만(하나는 AI 에이전트 쿼타, 하나는 클라우드 비용) 만들면서 겪은 문제는 놀랍도록 겹쳤다. **자격증명은 OS가 제공하는 안전장치를 우선하고 파일 폴백은 정직하게 권한으로만 방어한다. 자동화 배포 파이프라인은 "성공한 것처럼 보이는 실패"를 반드시 한 번은 겪는다. 그리고 정말 골치 아픈 버그는 코드 로직이 아니라 런타임/OS의 디테일(에러 객체 모양, 심링크 해석)에 숨어 있다.**

두 도구 모두 의존성을 최소로 유지했고(`oci-cost-cli`는 자격증명 저장용 라이브러리 하나가 전부다), 결과적으로 "내가 매번 손으로 하던 걸 대신 해주는" 정도의 작고 실용적인 크기로 남았다. 화려한 기능보다 "실행했을 때 예상한 그대로 동작하는가"를 더 중요하게 봤다.

## 참고

- [agy-cli-usage (GitHub)](https://github.com/abruption/agy-cli-usage) / [npm](https://www.npmjs.com/package/agy-cli-usage)
- [oci-cost-cli (GitHub)](https://github.com/abruption/oci-cost-cli) / [npm](https://www.npmjs.com/package/oci-cost-cli)
- [release-please](https://github.com/googleapis/release-please) — Conventional Commits 기반 자동 릴리스
- [OCI REST API Signing](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/signingrequests.htm) — OCI Signature v1 공식 스펙
