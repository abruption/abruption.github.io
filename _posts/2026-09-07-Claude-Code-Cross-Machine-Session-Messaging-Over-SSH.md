---
title: "다른 서버의 Claude Code 세션에 메시지 보내기 — Anthropic 서버를 거치지 않고"
author: abruption
date: 2026-09-07 18:00:00 +0900
categories: [Programming, Claude Code]
tags: [claude-code, unix-socket, ssh, tmux, tailscale, automation, open-source]
---

## 세션은 여러 대에 흩어져 있는데, 서로 말을 못 건다

Claude Code에는 다른 세션에 메시지를 보내는 기능이 있다. `ListAgents`로 찾고 `SendMessage`로 보낸다. 같은 머신에서는 잘 된다.

문제는 내 세션들이 한 머신에 있지 않다는 것이다. 노트북에 몇 개, 서울 리전 서버에 워커 세 개, 미국 리전에 하나. 전부 tmux 안에서 돌고 있고 tailnet으로 묶여 있다. 노트북에서 `/list-agents`를 치면 노트북 것만 나온다.

공식 문서를 보면 이렇게 갈린다.

| 대상 | 전달 경로 |
|---|---|
| 같은 머신 | 세션별 유닉스 소켓. 서버를 거치지 않음 |
| 다른 머신 | **Anthropic 서버 경유** (Remote Control) |
| 웹 세션 | Anthropic 서버 경유 |

Remote Control을 켜면 크로스 머신도 공식 지원된다. 다만 조건이 붙는다. 양쪽 세션 모두 연결돼 있어야 하고, claude.ai 로그인이 필요하고, Bedrock/Vertex/Foundry에서는 기능 자체가 비활성이다. 그리고 **메시지가 Anthropic 서버를 경유한다.**

tailnet 안에서만 돌리고 싶은데 굳이 밖으로 나갔다 와야 하나. 소켓이 로컬에 있는데.

## `ssh -L`은 안 된다

유닉스 소켓이니 OpenSSH의 소켓 포워딩으로 뚫으면 되지 않을까 싶었다. 안 된다.

바이너리를 뜯어보면 송신 측이 연결 직후 이런 검사를 한다.

```js
// 연결된 엔드포인트의 PID를 읽어서 기대값과 대조
if (peerPid !== expectedPid) {
  return reject("Refusing to send: connected endpoint is not the expected process");
}
// uid도 대조
if (myUid !== peerUid) {
  return reject("Refusing to send: connected endpoint is not owned by this user");
}
// PID 재활용까지 고려해 프로세스 시작 시각도 대조
```

포워딩을 걸면 로컬 소켓의 상대편은 `ssh` 프로세스다. 기대하던 Claude Code 프로세스가 아니니 즉시 거부된다. 공식 문서에도 *"an endpoint that isn't the expected process"* 라는 안전 점검으로 명시돼 있다.

즉 **소켓을 옮기려 하면 안 된다.** 대신 소켓이 있는 곳에서 쓰면 된다.

## 되는 방법: 원격 셸 안에서 쓴다

문서의 [The session's inbox socket](https://code.claude.com/docs/en/cross-session-messaging#the-sessions-inbox-socket) 절에 프로토콜이 그대로 적혀 있다. *"스크립트나 훅이 세션에 메시지를 post하고 싶을 때 읽으라"* 고까지 쓰여 있다. 리버스 엔지니어링할 것도 없었다.

JSON 한 줄이면 된다.

```json
{"type":"user","message":{"role":"user","content":"메시지"}}
```

이걸 SSH로 원격 셸에 들어가서 쓰면, 연결의 양 끝이 다시 같은 머신이 된다. PID 검사도 uid 검사도 정상 통과한다.

```bash
ssh build-server "python3 -" <<'PY'
import socket, json
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect("/tmp/cc-socks/4011.sock")
s.sendall((json.dumps({"type":"user","message":{"role":"user","content":"안녕"}})+"\n").encode())
s.close()
PY
```

실제로 원격 세션의 트랜스크립트에 `origin.kind: "peer"` 로 기록되고, 세션이 깨어난다. auth 라인은 macOS/Linux에서는 optional이라 토큰도 필요 없다.

여기까지가 30줄짜리 이야기다. 그런데 실제로 쓰려고 하니 함정이 두 개 있었다.

## 함정 1 — 소켓 경로를 추측하면 안 된다

맥미니와 서울 서버는 `/tmp/cc-socks/<pid>.sock` 이었다. 그래서 그 경로를 상수로 박아뒀는데, 미국 서버에서 세션이 안 잡혔다.

```console
$ ls -d /tmp/cc-socks*
(없음)

$ jq -r .messagingSocketPath ~/.claude/sessions/1092268.json
/run/user/1001/cc-socks/1092268.sock
```

같은 Claude Code v2.1.263인데 한쪽은 `/tmp`, 다른 쪽은 `$XDG_RUNTIME_DIR` 아래에 바인딩했다. 문서를 보면 Claude Code가 쓰려던 디렉토리를 거부할 경우 per-user 디렉토리(`/tmp/cc-socks-<uid>`)로 폴백하는 경로도 따로 있다. 즉 경우의 수가 최소 셋이다.

**답은 추측하지 말고 레지스트리를 읽는 것이다.** `~/.claude/sessions/<pid>.json` 에 `messagingSocketPath` 가 들어 있다.

```json
{
  "pid": 1092268,
  "name": "ontology",
  "status": "idle",
  "messagingSocketPath": "/run/user/1001/cc-socks/1092268.sock",
  "pidDomain": "linux:5ce79a...:pid:[4026531836]",
  "tmux": "Ontology:@0.%0"
}
```

`pidDomain` 필드도 눈에 띈다. macOS는 그냥 `darwin`, 리눅스는 `linux:<machine-id>:pid:[네임스페이스]` 형태다. PID로 생존 확인을 할 때 그 PID가 같은 도메인의 것인지 구분하려는 용도로 보인다. 컨테이너나 다른 머신의 PID를 자기 PID로 착각하지 않으려는 것이다.

살아있는 PID인데 소켓이 없는 세션도 있다. 그건 인박스가 아예 없는 것이라 메시지를 받을 수 없다. 죽은 세션의 레코드도 청소되기 전까지 남아 있다. 둘 다 걸러내야 한다.

## 함정 2 — Posted는 Delivered가 아니다

소켓에 쓰는 데 성공했다고 상대 Claude가 그걸 읽는 건 아니다.

서울 서버 워커에 테스트 메시지를 보내고 화면을 열어보니 이랬다.

```
● Released 1 held cross-session message to Claude's queue (you approved it).
❯ cc-peer 도구 검증 메시지입니다. ...
```

**held** 됐다가 승인되고 나서야 들어갔다. 문서의 인바운드 기본 규칙을 보면 이렇다.

- 받는 세션이 권한 프롬프트를 쓰면 → 바로 전달
- 받는 세션이 `--dangerously-skip-permissions` 로 돌면 → **보낸 쪽도 bypass라고 밝히지 않는 한 승인 대기**

스크립트는 자기가 bypass라고 주장할 방법이 없다. 그러니 무인 워커가 bypass 모드로 돌고 있으면, 보낸 메시지는 승인 다이얼로그에 걸려 있다가 기본 5분 뒤에 그냥 사라진다. 아무도 그 화면을 안 보고 있으니까.

무인으로 받게 하려면 받는 쪽에 이걸 켜야 한다.

```json
{ "crossSessionInbound": "accept" }
```

당연히 그만한 의미가 있다. 그 소켓에 쓸 수 있는 무엇이든 그 머신에서 턴을 시작시킬 수 있다는 뜻이다. 알고 켜야 한다.

## 그냥 `tmux send-keys` 쓰면 안 되나

세션들이 어차피 tmux 안에서 도니까 이렇게 해도 된다.

```bash
ssh build-server 'tmux send-keys -t worker "메시지" Enter'
```

가볍게 툭 던지는 용도로는 이게 낫다. 스크립트도 필요 없다. 다만 몇 가지가 걸린다.

| | `tmux send-keys` | 인박스 소켓 |
|---|---|---|
| 세션이 도구 실행 중일 때 | 키가 포커스 있는 곳으로 감 — 실행 중인 서브프로세스 stdin일 수도 | 큐에 들어갔다가 턴 경계에서 읽힘 |
| 따옴표·백틱·`$`·개행·이모지 | 셸과 터미널이 각각 한 번씩 건드림 | JSON으로 그대로 |
| 연속 전송 | 레이스 | 큐잉·버스트 제한·중복 드롭 내장 |
| 출처 | 사용자가 친 것과 구분 불가 | 다른 세션이 보낸 것으로 표시됨 |

특수문자는 실제로 테스트해봤다. ``따옴표 " 백틱 ` 달러 $HOME 이모지 🚀`` 를 그대로 보냈고 그대로 도착했다.

마지막 줄이 제일 중요하다고 본다. 소켓으로 들어온 메시지는 **권한 프롬프트를 승인할 수 없고 설정을 바꿀 수 없다.** Claude Code가 받는 쪽에 그렇게 지시한다. 반면 `send-keys` 로 밀어넣은 텍스트는 사용자가 직접 타이핑한 것과 구별되지 않는다. 권한 프롬프트가 떠 있을 때 `y` 를 밀어넣는 것도 가능하다는 뜻이다.

## 도구로 묶었다

매번 PID 찾고 경로 확인하는 게 번거로워서 한 파일짜리 스크립트로 만들었다. [abruption/cc-peer](https://github.com/abruption/cc-peer).

```console
$ cc-peer list --host build-server
Sessions on build-server:
NAME               PID   STATUS  CWD
ontology           2448  idle    /srv/notes
ontology-worker    4011  idle    /srv/notes
ontology-worker-2  4614  idle    /srv/notes

$ cc-peer send --host build-server --to ontology-worker "스키마 마이그레이션 끝났습니다."
Posted to ontology-worker's inbox on build-server (16 chars).
```

`--host` 를 주면 스크립트가 자기 소스를 SSH로 넘겨 원격에서 실행한다. 원격에는 `python3` 만 있으면 되고 설치할 게 없다. 표준 라이브러리만 쓴다.

Claude Code Skill도 같이 넣어뒀다. 스크립트가 전송이라는 결정론적인 부분을 맡고, "어느 세션에 무엇을 보낼지" 라는 판단은 스킬이 맡는 구조다. 세션 디스커버리는 레지스트리 스키마에 의존하는데 이건 문서화된 인터페이스가 아니라서 언젠가 바뀔 수 있다. 코드에 하드코딩하는 것보다 절차를 자연어로 적어두고 에이전트가 적응하게 하는 편이 오래 간다고 봤다.

## 언제 쓰지 말아야 하나

**Remote Control이 되면 그걸 쓰는 게 맞다.** 설정할 게 없고, 받는 쪽 Claude에게 회신 주소까지 준다. 이 방식은 회신 주소가 없어서 단방향이다. 답을 받으려면 반대 방향으로 한 번 더 쏘는 수밖에 없다.

이게 필요한 경우는 좁다.

- Bedrock / Vertex / Foundry — Remote Control 자체가 비활성
- API 키 인증 — claude.ai 로그인이 없음
- 망분리·컴플라이언스 — 제3자 경유가 문제인 환경
- 무인 워커 — 연결해줄 사람이 없음

넷 중 하나도 해당 안 되면 공식 기능으로 충분하다.

그리고 이건 권한 상승 수단이 아니다. 소켓은 세션 소유자에게만 열려 있고, 애초에 그 계정의 셸이 있어야 쓸 수 있다. 셸이 있으면 어차피 뭐든 되는 상태다. 다만 그 반대도 성립한다 — 그 머신의 셸을 가진 사람은 누구든 거기 도는 에이전트에게 말을 걸 수 있다.

---

Claude Code v2.1.263, macOS 15 / Ubuntu 24.04 (arm64)에서 확인했다. 소켓 프로토콜은 문서화돼 있지만 세션 레지스트리 스키마는 아니다. 버전이 올라가면 깨질 수 있는 쪽은 후자다.
