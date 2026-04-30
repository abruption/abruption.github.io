---
title: "Secret Key 없이 CLI 인증 만들기 — Clerk + Loopback 서버 패턴"
author: abruption
date: 2026-04-30 11:00:00 +0900
categories: [Programming, Authentication]
tags: [clerk, jwt, authentication, cli, oauth, fastify, jose, mcp]
---

## CLI 도구에 인증이 필요해졌다

MCP 서버를 만들면서 사용자별 데이터를 분리해야 하는 시점이 왔다. stdio 모드에서는 로컬 파일만 다루니까 인증이 필요 없었는데, HTTP Gateway를 띄우고 원격에서도 접근 가능하게 만들면서 "이 요청이 누구 건지" 알아야 했다.

전통적으로 CLI 도구의 인증은 두 가지 중 하나다. API Key를 환경변수에 넣거나, OAuth Device Flow를 쓰거나. 그런데 이번에는 조건이 하나 더 있었다. **사용자 머신에 Secret Key를 두지 않는다.** 플러그인 형태로 배포되는 도구라서, Secret Key가 코드에 박히거나 설정 파일에 노출되면 안 됐다.

Clerk를 쓰면서 이 조건을 만족하는 패턴을 찾았다. Publishable Key만으로 브라우저 기반 로그인을 수행하고, JWT를 로컬에 저장하고, 세션 시작 시 Hook이 자동 갱신하는 구조다.

## 설계 원칙 — Publishable Key만 배포

Clerk는 두 종류의 키를 제공한다. Backend API를 호출하는 `sk_live_*`(Secret Key)와, 프론트엔드에서 Clerk.js를 초기화하는 `pk_live_*`(Publishable Key). Secret Key가 있으면 사용자 정보 조회, 세션 관리 등 서버 사이드 작업이 가능하고, Publishable Key는 로그인 UI를 렌더링하는 것까지만 할 수 있다.

CLI 도구는 사용자 머신에 설치된다. npm으로 배포하든, 플러그인 마켓플레이스로 배포하든, 코드가 사용자에게 보인다. 여기에 Secret Key를 넣으면 추출당할 수 있다.

그래서 빌드 시점에 Publishable Key만 bake하고, 인증은 전부 브라우저 기반으로 처리하기로 했다. 사용자가 브라우저에서 Clerk 로그인을 완료하면, Clerk.js가 클라이언트 사이드에서 JWT를 발급한다. 이 JWT를 CLI가 받아서 로컬에 저장하는 구조다. 서버 사이드 Secret Key가 필요한 작업은 하나도 없다.

## Loopback 로그인 플로우

### 전체 흐름

```
CLI (cozy login)
  │
  ├─ 1. Loopback HTTP 서버 기동 (127.0.0.1:54545)
  ├─ 2. 브라우저 자동 열기 → http://127.0.0.1:54545/callback
  │
  │   [브라우저]
  │   ├─ 3. Callback HTML 수신 (Clerk.js CDN 로드)
  │   ├─ 4. Clerk.openSignIn() 모달 표시
  │   ├─ 5. 사용자 로그인 완료
  │   ├─ 6. session.getToken({template}) → JWT 발급
  │   └─ 7. POST /token {jwt, state} → CLI로 전달
  │
  ├─ 8. nonce(state) 검증
  ├─ 9. JWT payload 파싱 (issuer, exp 확인)
  └─ 10. ~/.my-tool/current-jwt에 저장 (0600 퍼미션)
```

이 패턴은 GitHub CLI(`gh auth login`)나 Google Cloud CLI(`gcloud auth login`)에서도 쓰는 방식이다. 로컬 루프백 서버를 띄워서 브라우저 인증 결과를 받아오는 것.

### Loopback 서버 구현

핵심은 3개의 엔드포인트다. `GET /callback`은 Clerk.js를 포함한 HTML을 반환하고, `POST /token`은 브라우저에서 발급된 JWT를 수신하고, `POST /error`는 브라우저 측 오류를 CLI에 전파한다.

```ts
const server = createServer(async (req, res) => {
  const url = new URL(req.url ?? '/', `http://${req.headers.host}`);

  // 브라우저에 로그인 UI 제공
  if (req.method === 'GET' && url.pathname === '/callback') {
    res.setHeader('Content-Type', 'text/html; charset=utf-8');
    res.end(renderLoginHtml(publishableKey, nonce));
    return;
  }

  // JWT 수신
  if (req.method === 'POST' && url.pathname === '/token') {
    const body = JSON.parse(await readBody(req));

    // CSRF 방지: nonce 검증
    if (body.state !== nonce) {
      res.statusCode = 400;
      res.end('STATE_MISMATCH');
      return;
    }

    const payload = decodeJwtPayload(body.jwt);

    // issuer 검증
    if (payload.iss !== expectedIssuer) {
      res.statusCode = 400;
      res.end('ISSUER_MISMATCH');
      return;
    }

    res.statusCode = 204;
    res.end();
    resolve({ jwt: body.jwt, payload });
    return;
  }
});

server.listen(54545, '127.0.0.1');
```

`nonce`는 `randomUUID()`로 생성한다. HTML에 심어서 브라우저가 `POST /token` 시 함께 보내도록 하고, 서버에서 일치 여부를 확인한다. 다른 탭이나 프로세스가 무작위로 JWT를 보내는 걸 방지하는 최소한의 CSRF 보호다.

### 브라우저 측 HTML

Callback HTML은 Clerk.js CDN을 로드하고, 세션이 없으면 로그인 모달을 띄우고, 세션이 생기면 JWT를 발급받아 CLI에 전달한다.

```html
<script
  async crossorigin="anonymous"
  data-clerk-publishable-key="pk_live_..."
  src="https://your-instance.clerk.accounts.dev/npm/@clerk/clerk-js@latest/dist/clerk.browser.js"
></script>

<script>
  window.addEventListener('load', async () => {
    // Clerk.js 로드 대기 (최대 10초)
    while (!window.Clerk && Date.now() - start < 10000) {
      await new Promise(r => setTimeout(r, 100));
    }
    await window.Clerk.load();

    // 이미 세션이 있으면 바로 JWT 발급
    if (window.Clerk.session) {
      const jwt = await window.Clerk.session.getToken({ template: 'my-template' });
      await fetch('/token', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ jwt, state: NONCE }),
      });
      return;
    }

    // 세션 없으면 로그인 모달
    window.Clerk.openSignIn({ afterSignInUrl: window.location.href });

    // 로그인 완료 폴링
    const poll = setInterval(async () => {
      if (window.Clerk.session) {
        clearInterval(poll);
        const jwt = await window.Clerk.session.getToken({ template: 'my-template' });
        await fetch('/token', { ... });
      }
    }, 500);
  });
</script>
```

`getToken({ template })` 에서 `template`은 Clerk Dashboard에서 정의한 JWT 템플릿이다. 기본 토큰에는 `sub`(user_id)만 들어가지만, 템플릿을 설정하면 `email`, `org_id`, `org_role` 같은 커스텀 클레임을 추가할 수 있다. 서버 측에서 어떤 정보가 필요한지에 따라 템플릿을 조정하면 된다.

### 타임아웃과 에러 처리

Loopback 서버는 영원히 대기하면 안 된다. 10분 타임아웃을 걸어두고, 다음 상황별로 exit code를 분리했다.

| Exit Code | 상황 | 대응 |
|:---------:|------|------|
| 0 | 로그인 성공 | JWT 저장 완료 |
| 2 | 포트 점유 (EADDRINUSE) | 기존 프로세스 종료 안내 |
| 3 | 10분 타임아웃 | 재시도 안내 |
| 4 | 브라우저 측 오류 | Clerk.js 로드 실패 등 |
| 5 | nonce/issuer 불일치 | 보안 경고 |
| 6 | 파일 쓰기 실패 | 권한 확인 안내 |

## 자격증명 저장 구조

로그인이 완료되면 JWT와 사용자 정보를 홈 디렉토리에 저장한다.

```
~/.my-tool/
├── current-jwt        (Clerk JWT, 7일 유효, 0600 퍼미션)
├── user-config.json   (email, user_id, last_login_at)
├── jwt-refresh.log    (Hook 실행 이력)
├── gateway.pid        (HTTP Gateway PID)
└── gateway.log        (Gateway 로그)
```

`current-jwt`는 JWT 원문 하나만 들어있는 파일이다. 파일 퍼미션을 `0600`으로 설정해서 소유자만 읽을 수 있게 한다. JSON으로 감싸지 않은 이유는, SessionStart Hook에서 `readFileSync`로 읽어서 바로 `Authorization: Bearer` 헤더에 넣을 수 있게 하기 위해서다.

`user-config.json`은 참고용이다. CLI가 "현재 누구로 로그인되어 있는지" 표시할 때 쓰고, JWT 검증 로직에서는 참조하지 않는다.

## SessionStart Hook — JWT 자동 갱신

매 세션 시작 시 Hook이 JWT 상태를 확인하고, 플러그인 설정 파일의 Authorization 헤더를 갱신한다. 이전에 쓴 [Hooks 포스트](https://abruption.github.io/posts/Claude-Code-Hooks-Practical-Guide/)의 Hook 2에 해당하는 부분인데, 구체적인 Clerk JWT 처리 로직을 설명한다.

### 5가지 상태 분기

```
JWT 파일 확인
├─ 파일 없음 + 자동 로그인 ON  → cozy login 자동 실행 (10분 blocking)
├─ 파일 없음 + 자동 로그인 OFF → Bearer INVALID_JWT_MISSING 마커
├─ 파싱 실패                    → Bearer INVALID_JWT_PARSE_FAILED 마커
├─ issuer 불일치                → Bearer INVALID_JWT_ISSUER_MISMATCH 마커
├─ 만료됨 (exp ≤ now)          → Bearer INVALID_JWT_EXPIRED 마커
├─ 24시간 내 만료              → stderr 경고 + 기존 Bearer 유지
└─ 정상                         → Bearer <jwt> 갱신
```

### Fail-fast 마커의 설계 의도

토큰이 만료되었을 때 Hook이 에러를 던지면 세션 시작 자체가 실패한다. 그러면 사용자가 Claude Code를 아예 쓸 수 없게 된다. 인증이 필요한 건 MCP 도구를 호출할 때뿐인데, 세션 전체를 막아버리는 건 과한 대응이다.

대신 `plugin.json`의 Authorization 헤더에 `Bearer INVALID_JWT_EXPIRED` 같은 마커를 기록한다. MCP 서버의 HTTP Gateway가 이 마커를 보면 즉시 401을 반환하고, Claude Code가 사용자에게 "재로그인이 필요합니다"라고 안내한다. 인증이 필요 없는 다른 기능은 정상적으로 사용할 수 있다.

```ts
// Hook에서 마커를 기록
function writeBearerMarker(reason: string): void {
  const marker = `Bearer INVALID_${reason}`;
  updatePluginJson({ authorization: marker });
}

// 만료 감지 시
if (exp <= now) {
  writeBearerMarker('JWT_EXPIRED');
  process.exit(0);  // 에러가 아닌 정상 종료
}
```

`process.exit(0)`으로 정상 종료하는 게 핵심이다. Hook이 비정상 종료하면 Claude Code가 세션 시작을 중단할 수 있기 때문에, 마커만 남기고 깔끔하게 빠진다.

## HTTP Gateway — per-request JWT 검증

Loopback 서버는 로그인할 때만 쓰고, 실제 API 요청은 Fastify 기반 HTTP Gateway에서 처리한다. Gateway의 preHandler hook에서 매 요청마다 JWT를 검증한다.

### jose + Remote JWKS

Clerk JWT는 RS256으로 서명된다. 검증에는 `jose` 라이브러리의 `createRemoteJWKSet`을 쓴다. JWKS(JSON Web Key Set) 엔드포인트에서 공개 키를 가져와 서명을 검증하는 방식인데, `jose`가 내부적으로 10분 캐시를 관리해서 매 요청마다 네트워크 호출이 발생하지 않는다.

```ts
import { createRemoteJWKSet, jwtVerify } from 'jose';

const jwks = createRemoteJWKSet(
  new URL('https://your-instance.clerk.accounts.dev/.well-known/jwks.json')
);

async function verifyToken(token: string): Promise<AuthContext> {
  const { payload } = await jwtVerify(token, jwks, {
    issuer: 'https://your-instance.clerk.accounts.dev',
  });

  return {
    userId: payload.sub ?? payload.user_id,
    orgId: payload.org_id ?? null,
    email: payload.email ?? null,
    orgRole: payload.org_role ?? null,
    issuer: payload.iss,
  };
}
```

Secret Key가 없어도 JWT 검증이 가능한 이유가 여기에 있다. RS256은 비대칭 키 알고리즘이라서, Clerk가 Private Key로 서명하고 JWKS 엔드포인트에 공개한 Public Key로 검증한다. 우리는 Public Key만 있으면 된다.

### Fastify preHandler로 인증 부착

검증된 AuthContext를 Fastify의 request 객체에 부착하면, 이후 라우트 핸들러에서 `request.authContext.userId`로 바로 접근할 수 있다.

```ts
function createAuthHook(opts: { getClerk: () => ClerkConfig | undefined }) {
  const publicPaths = new Set(['/healthz']);

  return async function authHook(request: FastifyRequest, reply: FastifyReply) {
    // 헬스체크는 인증 없이 통과
    if (publicPaths.has(request.url)) return;

    // 매 요청 시 config 재검증 — clerk 블록이 삭제되면 즉시 503
    const clerk = opts.getClerk();
    if (!clerk) {
      await reply.code(503).send({ error: 'clerk config missing' });
      return;
    }

    const header = request.headers['authorization'];
    if (!header?.startsWith('Bearer ')) {
      await reply.code(401).send({ error: 'missing Bearer token' });
      return;
    }

    const token = header.slice(7).trim();
    request.authContext = await verifyToken(token);
  };
}
```

`getClerk()`이 매 요청마다 설정 파일을 다시 읽는 이유는, Gateway 재시작 없이 설정을 변경할 수 있게 하기 위해서다. 설정 파일의 mtime을 체크해서 변경된 경우에만 실제로 읽기 때문에 성능 부담은 거의 없다.

## 전체 구조 요약

```
최초 로그인 (1회)
  CLI → Loopback :54545 → 브라우저 Clerk.js → JWT 발급
      → ~/.my-tool/current-jwt 저장

세션 시작 (매회)
  SessionStart Hook
    → current-jwt 읽기 → 만료 확인
    → plugin.json Authorization 헤더 갱신

API 요청 (매회)
  클라이언트 → HTTP Gateway :3033
    → Fastify preHandler: Bearer 추출 → jose JWKS 검증
    → request.authContext 부착 → 도구 핸들러 실행
```

사용자 입장에서는 처음 한 번 브라우저 로그인만 하면 된다. 이후 7일간은 Hook이 자동으로 JWT를 확인하고 헤더를 갱신한다. 만료되면 자동 로그인이 다시 브라우저를 열거나, 마커를 남겨서 사용 시점에 재로그인을 안내한다.

## 이 패턴이 맞지 않는 경우

이 구조가 모든 CLI 도구에 적합한 건 아니다.

**CI/CD 환경**에서는 브라우저가 없다. 이 경우 Service Account용 API Key를 환경변수로 주입하는 기존 방식이 맞다. Hook에 `AUTO_LOGIN=0`을 설정하고 JWT를 직접 주입하는 우회 경로를 열어두는 게 좋다.

**오프라인 환경**에서는 JWKS 엔드포인트에 접근할 수 없다. `jose`의 캐시가 있어서 한 번 가져온 키는 10분간 유효하지만, 장시간 오프라인이면 키 로테이션을 놓칠 수 있다. 이런 경우 Public Key를 로컬에 번들링하는 방식을 고려해야 한다.

**Clerk가 아닌 IdP**를 쓴다면, Loopback 서버의 HTML 부분만 바꾸면 된다. Auth0든 Firebase Auth든, 브라우저에서 JWT를 발급받아 `POST /token`으로 보내는 구조는 동일하다. 핵심은 "CLI가 Secret Key 없이, 브라우저를 통해 토큰을 받는다"는 패턴 자체다.

## 참고

- [Clerk JWT Templates](https://clerk.com/docs/backend-requests/making/jwt-templates) — 커스텀 클레임 설정
- [jose - JavaScript JOSE](https://github.com/panva/jose) — JWT/JWS/JWE 라이브러리
- [Clerk.js Frontend API](https://clerk.com/docs/references/javascript/overview) — Publishable Key 기반 프론트엔드 SDK
- [RFC 8252 - OAuth 2.0 for Native Apps](https://datatracker.ietf.org/doc/html/rfc8252) — Loopback redirect 패턴의 표준 근거
