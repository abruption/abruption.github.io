---
title: "iPad에서 Mac Mini의 Claude Code를 원격 실행하기 — Tailscale + tmux 가이드"
author: abruption
date: 2026-03-04 10:00:00 +0900
categories: [DevOps, Remote Access]
tags: [tailscale, ssh, tmux, claude-code, ipad, mac-mini]
---

Claude Code를 쓰다 보면 자연스럽게 드는 생각이 있다. "이걸 iPad에서도 쓸 수 있으면 좋겠는데." 터미널 기반이라 iOS에서 직접 돌리는 건 안 되지만, 집에 Mac Mini를 하나 켜놓고 SSH로 접속하면 이야기가 달라진다.

지금 내 환경은 이렇다. 자택 Mac Mini에 Tailscale과 tmux를 깔아놓고, iPad의 Termius로 접속해서 Claude Code를 돌린다. 카페든 지하철이든 iPad만 있으면 된다. 전부 무료 도구다.

```
iPad (어디서든)
  ├─ Tailscale (메시 VPN)
  └─ Termius (SSH 클라이언트)
       │
       ▼  SSH 접속
       │
Mac Mini - 자택, 항상 켜져 있음
  ├─ Tailscale
  ├─ SSH
  └─ tmux → Claude Code
```

---

## 왜 Tailscale인가

원격 접속 방법을 고민하면서 포트 포워딩, DDNS, Cloudflare Tunnel을 전부 검토했다. 결론부터 말하면 이 용도에는 Tailscale이 압도적으로 간단하다.

Cloudflare Tunnel도 써봤는데, SSH 접속하려면 `cloudflared access ssh` 프록시를 거쳐야 하고, 도메인도 필요하고, iOS에서 설정이 복잡하다. 웹 서비스를 외부에 공개할 때는 Cloudflare가 맞지만, 내 기기끼리 사설 네트워크를 만드는 건 Tailscale이 훨씬 적합하다. 앱 설치하고 로그인하면 끝이다. 진짜로.

한 가지 아쉬운 점이 있다면 mosh 지원인데, Tailscale 자체는 UDP를 지원해서 mosh가 동작하지만 Termius 무료 버전에서 mosh를 못 쓴다. mosh를 쓰고 싶으면 Blink Shell(유료)이 필요하다. 나는 tmux로 세션을 유지하는 걸로 대체했고, 실사용에서 큰 불편은 없었다.

---

## Mac Mini 설정

### SSH 켜기

시스템 설정 > 일반 > 공유에서 "원격 로그인"을 켜면 된다. 확인은:

```bash
sudo systemsetup -getremotelogin
# → Remote Login: On
```

### Tailscale 설치

```bash
brew install tailscale
```

앱 실행 후 로그인하면 Mac Mini에 Tailscale IP가 할당된다.

```bash
tailscale ip -4
# → 100.x.y.z
```

이 IP를 나중에 iPad에서 SSH 접속할 때 쓴다.

### tmux 설치

```bash
brew install tmux
```

### 절전 모드 — 반드시 끄기

이걸 빼먹으면 외출 후 접속이 안 된다. Mac Mini가 잠들면 SSH가 안 되기 때문이다. 처음 세팅했을 때 카페에서 "왜 안 되지?" 하고 한참 삽질했는데, 원인이 절전 모드였다.

```bash
sudo pmset -c sleep 0 displaysleep 10
```

디스플레이만 끄고 시스템은 깨어 있게 설정하는 명령어다. 또는 시스템 설정 > 에너지 절약에서 "잠자기"를 "안 함"으로 바꿔도 된다.

---

## iPad 설정

App Store에서 Tailscale과 Termius를 설치한다. Tailscale은 Mac Mini와 같은 계정으로 로그인해야 서로 보인다.

Termius에서 호스트를 하나 등록한다. Hostname에 아까 확인한 Tailscale IP(`100.x.y.z`)를 넣고, Port 22, macOS 사용자명과 비밀번호를 입력하면 된다. 저장하고 탭하면 바로 SSH 접속.

비밀번호 인증이 불편하면 SSH 키로 바꾸는 걸 추천한다. Termius의 Keychain에서 Ed25519 키를 생성하고, 공개키를 Mac Mini의 `~/.ssh/authorized_keys`에 추가하면 된다.

```bash
mkdir -p ~/.ssh
echo "공개키_내용" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
```

---

## tmux가 핵심이다

솔직히 이 구성에서 가장 중요한 건 Tailscale이 아니라 tmux다. SSH만 쓰면 iPad를 닫거나 네트워크가 잠깐이라도 끊기면 Claude Code 세션이 날아간다. 긴 대화를 하고 있었다면 꽤 고통스럽다.

tmux를 쓰면 SSH 연결과 터미널 세션이 분리된다. 연결이 끊겨도 세션은 Mac Mini에서 계속 살아 있고, 다시 접속하면 그대로 이어갈 수 있다.

```bash
# 새 세션
tmux new -s claude
claude

# 재접속 시
tmux attach -s claude
```

작업을 중단할 때는 `Ctrl+B` → `D`로 세션을 분리하면 된다. 그냥 Termius를 닫아도 세션은 유지된다. 다음 날 접속하면 어제 대화하던 Claude Code가 그대로 떠 있다. 이게 진짜 편하다.

tmux 조작이 처음이면 이것만 기억하면 된다:
- `Ctrl+B` → `D` : 세션 분리 (나가기, 세션은 유지)
- `Ctrl+B` → `[` : 스크롤 모드 (`q`로 복귀)

---

## 트러블슈팅

두 달 정도 쓰면서 겪은 문제들이다.

**Mac Mini가 Tailscale에 안 보인다** — 십중팔구 절전 모드다. 위에 쓴 `pmset` 설정을 확인하자.

**SSH가 자꾸 끊긴다** — SSH KeepAlive를 설정하면 개선된다. Mac Mini의 `/etc/ssh/sshd_config`에 아래 두 줄을 추가한다.

```
ClientAliveInterval 60
ClientAliveCountMax 3
```

60초마다 패킷을 보내서 연결을 유지하고, 3번 연속 응답이 없으면 끊는다.

**Tailscale이 처음에 느리다** — DERP 릴레이를 거치고 있어서 그렇다. 잠시 후 P2P로 전환되면 빨라진다. 체감상 수초 내에 해결된다.

---

## 두 달 사용 후기

결론부터 말하면, iPad가 사실상 개발 터미널이 됐다. 카페에서 코드 리뷰하다가 이동 중에 이어서 하고, 집에 돌아와서 MacBook으로 넘어간다. tmux 세션이 며칠이고 살아 있어서 컨텍스트가 끊기지 않는 게 가장 큰 장점이다.

Wi-Fi에서 셀룰러로 전환될 때 Tailscale이 자동으로 재연결하는 것도 좋다. SSH는 끊기지만 tmux가 살아 있으니 `tmux attach`만 하면 복귀한다.

추가 비용 0원인 것도 마음에 든다. Tailscale 무료, Termius 무료, tmux 무료. Mac Mini 전기세 빼면.

Claude Code 말고도 git이나 Docker 같은 CLI 도구를 iPad에서 쓸 수 있게 되는 건 보너스다. iPad의 물리적 한계를 Mac Mini가 대신 처리해주는 구조라서, 활용 범위가 넓다.

---

## 참고 링크

- [Tailscale 공식 문서](https://tailscale.com/kb/)
- [tmux Cheat Sheet](https://tmuxcheatsheet.com/)
- [Termius - App Store](https://apps.apple.com/app/termius-terminal-ssh-client/id549039908)
- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
