---
title: "Windows 전용 Electron 앱을 macOS로 확장한 이야기 — node-pty 제거부터 Apple Silicon 지원까지"
author: abruption
date: 2024-12-15 10:00:00 +0900
categories: [DevTools, Electron]
tags: [electron, macos, cross-platform, nuxt, apple-silicon, typescript]
---

## 배경

사내 CI/CD 데스크톱 도구가 있다. AWS Lambda, ECS Fargate, API Gateway 등 클라우드 리소스의 배포·관리·모니터링을 GUI 환경에서 수행하는 Electron + Nuxt 2 앱이다.

문제는 이 도구가 **Windows 전용**이었다는 점이다. 팀 내 macOS 사용자(특히 Apple Silicon Mac)가 늘어나면서, 동일한 도구를 macOS에서도 사용할 수 있어야 했다. 단순히 "빌드만 돌리면 되지 않을까?" 싶었지만, 실제로는 터미널 처리, 파일 경로, 빌드 설정, SSH 터널링까지 거의 모든 레이어에서 플랫폼 종속적인 코드가 박혀 있었다.

이 글에서는 Windows 전용 Electron 앱을 macOS(Intel + Apple Silicon)로 확장하면서 겪은 주요 변경사항을 정리한다.

| 항목 | 스택 |
|---|---|
| Framework | Electron 28 + Nuxt 2 + Vue.js |
| Language | TypeScript |
| Build | Webpack + Electron Builder |
| Node.js | >= 18 |

---

## 1. Electron 메인 프로세스 — 환경별 부트스트랩 분리

기존에는 단일 `index.js`에서 개발·프로덕션 환경을 분기하고 있었다. macOS 앱 번들(`app://` 프로토콜)을 지원하려면 프로덕션 환경의 프로토콜 등록이 필요했고, 이를 깔끔하게 분리하기로 했다.

### 개발 모드

```typescript
// boot/index.dev.ts
require('@electron/remote/main').initialize()

app.once('browser-window-created', (_, browserWindow) => {
  require('@electron/remote/main').enable(browserWindow.webContents)
  browserWindow.webContents.once('did-frame-finish-load', () => {
    browserWindow.webContents.openDevTools()
  })
})
```

### 프로덕션 모드

```typescript
// boot/index.prod.ts
protocol.registerSchemesAsPrivileged([
  { scheme: 'app', privileges: { secure: true, standard: true } }
])

protocol.registerFileProtocol('app', (request, callback) => {
  const relativePath = request.url.replace('app://./', '')
  const absolutePath = path.join(PRODUCTION_APP_PATH, relativePath)
  callback({ path: absolutePath })
})
```

Webpack 설정에서 환경에 따라 entry point를 분기한다.

```javascript
entry: isDev
  ? path.join(MAIN_PROCESS_DIR, 'boot/index.dev.js')
  : path.join(MAIN_PROCESS_DIR, 'boot/index.prod.js')
```

---

## 2. BrowserWindow — macOS 특유의 윈도우 라이프사이클

macOS의 Electron 앱은 Windows와 윈도우 라이프사이클이 다르다. 대표적으로 **Dock 아이콘을 클릭하면 닫힌 윈도우가 다시 열려야 한다**.

### 기존 (Windows 전용)

```javascript
app.on('ready', () => {
  this._create()
})
```

### 변경 (크로스 플랫폼)

```javascript
// 이미 ready 상태면 즉시 생성 (macOS에서 activate 이벤트 순서 이슈 방지)
if (app.isReady()) this._create()
else app.once('ready', () => { this._create() })

// macOS: Dock 아이콘 클릭 시 윈도우 재생성
app.on('activate', () => this._recreate())
```

페이지 로딩 시에도 프로토콜을 분기한다.

```javascript
async loadPage(pagePath) {
  const serverUrl = isDev ? DEV_SERVER_URL : 'app://./index.html'
  await this.browserWindow.loadURL(serverUrl + '#' + pagePath)
}
```

개발 환경에서는 `localhost` 개발 서버를, 프로덕션에서는 `app://` 커스텀 프로토콜을 사용한다.

---

## 3. Terminal 컴포넌트 — 가장 큰 변경

이 프로젝트에서 가장 많은 시간을 쓴 부분이다. 기존에는 **node-pty + xterm.js**로 앱 내에 완전한 대화형 터미널을 임베딩하고 있었다.

### 기존 구현 (251줄)

```typescript
import * as pty from 'node-pty'
import { Terminal } from 'xterm'
import { FitAddon } from 'xterm-addon-fit'

const term = new Terminal({ fontFamily: 'Fira Code, Iosevka, monospace', fontSize: 12 })
const shell = process.env[os.platform() === 'win32' ? 'COMSPEC' : 'SHELL']

ptyProcess = pty.spawn(shell, [], {
  name: 'xterm-color',
  cols: term.cols, rows: term.rows,
  cwd: process.cwd(), env: process.env
})

ptyProcess.on('data', (data) => {
  // ANSI escape 파싱, 프롬프트 감지, 콜백 분기...
  // 250줄의 상태 머신 로직
})
```

문제점:
- **node-pty**가 macOS에서 네이티브 빌드 실패가 잦음 (Node.js 버전, Electron 버전, Xcode 버전 조합에 민감)
- ANSI escape 파싱 로직이 지나치게 복잡하고 깨지기 쉬움
- xterm.js 관련 의존성이 5개나 됨

### 변경 구현 (95줄)

실제로 이 앱이 터미널에서 하는 일은 **명령어를 실행하고 결과를 기다리는 것**뿐이었다. 대화형 터미널이 필요 없었다.

```typescript
import { execSync } from 'child_process'
import { promisify } from 'util'

// 비동기 명령어 실행 — 플랫폼별 분기
async sendCommand(cmd: string) {
  const platform = process.platform

  if (platform === 'win32') {
    // Windows: cmd.exe로 실행
    require('child_process').exec(`start cmd.exe /k "${cmd}"`)
  } else if (platform === 'darwin') {
    // macOS: Windows 경로 구분자 제거 후 실행
    cmd = cmd.replace(/C:\s*&&\s*/g, '')
    const { exec: execCb } = require('child_process')
    await promisify(execCb)(cmd)
  } else {
    require('child_process').exec(cmd)
  }
}

// 프롬프트 명령어 — macOS에서 AppleScript 활용
sendPromptCommand(cmd: string) {
  const platform = process.platform

  if (platform === 'win32') {
    execSync(`start cmd.exe /k "${cmd}"`)
  } else if (platform === 'darwin') {
    cmd = cmd.replace(/C:\s*&&\s*/g, '')
    // AppleScript로 시스템 Terminal.app에 명령 위임
    execSync(`osascript -e 'tell application "Terminal" to do script "${cmd}"'`)
  } else {
    execSync(cmd)
  }
}
```

**결과**:
- 251줄 → 95줄 (62% 감소)
- 의존성 5개 제거 (`node-pty`, `xterm`, `xterm-addon-fit`, `xterm-addon-ligatures`, `ansi-escapes`)
- 네이티브 빌드 문제 완전 해소
- macOS에서 `osascript`를 활용하여 시스템 Terminal.app에 명령을 위임

> macOS에서 `osascript`는 AppleScript를 커맨드라인에서 실행하는 명령어다. `tell application "Terminal" to do script "..."` 구문으로 Terminal.app에 직접 명령을 전달할 수 있다.
{: .prompt-info }

---

## 4. 파일시스템 유틸리티 — zip 바이너리 처리

배포 과정에서 파일을 압축/해제하는 기능이 있었는데, Windows에서는 번들된 `zip.exe`를 사용하고 있었다. macOS에는 `zip`이 기본 설치되어 있으므로 분기 처리했다.

### 기존

```typescript
export const getZipPath = () => {
  const app = require('electron').remote.app
  const defaultPath = `${app.getAppPath()}/../../template/files/`
  return _path.join(defaultPath, 'zip.exe')
}
```

### 변경

```typescript
const absoluteGitRootDir = _path.resolve(__dirname)

export const getZipPath = () => {
  if (process.platform === 'darwin') return 'zip'  // macOS 네이티브
  const defaultPath = _path.join(absoluteGitRootDir, '../../../template/files/')
  return _path.resolve(defaultPath, 'zip.exe')
}

export const getUnZipPath = () => {
  if (process.platform === 'darwin') return 'zip'
  const defaultPath = `${absoluteGitRootDir}/template/files/`
  return _path.join(defaultPath, 'unzip.exe')
}
```

이 과정에서 `electron.remote.app` 의존도 제거했다. Electron의 보안 정책 변경으로 `remote` 모듈 사용이 권장되지 않기 때문이다. 대신 `__dirname` 기반의 절대경로를 사용한다.

---

## 5. SSH 터널링 — DB 접속 방식 변경

이 도구는 Bastion Host를 경유하여 RDS에 접속한다. 기존에는 node-pty를 통해 SSH 세션을 대화형으로 제어했는데, Terminal 컴포넌트 변경에 따라 이 부분도 수정이 필요했다.

### 기존: node-pty 기반 대화형 SSH

```typescript
this.terminal.sendCommand(
  `ssh -L ${port}:${host}:${port} user@${bastionIP} -i ${keyPath}`,
  [{ message: 'be established.', command: 'yes' }],
  callback
)
```

### 변경: AppleScript + 연결 캐싱

```typescript
this.terminal.sendPromptCommand(
  `chmod 600 ${keyPath} && ssh -L ${port}:${host}:${port} user@${bastionIP} -i ${keyPath}`
)
await sleep(2000)  // 터널 연결 대기
```

연결 상태를 캐싱하여 중복 터널 생성을 방지했다.

```typescript
private tunnelEstablished = false
private currentKeyPath = ''

public async connect(keyPath: string, ...args) {
  if (!this.tunnelEstablished || this.currentKeyPath !== keyPath) {
    this.tunnelEstablished = true
    this.currentKeyPath = keyPath
    // 새 SSH 터널 생성
  } else {
    if (!this.importer) this.initInstance()
  }
}

public async close() {
  await this.connection?.end()
  await this.importer?.disconnect()
  this.connection = undefined
  this.importer = undefined
}
```

---

## 6. Electron Builder — Apple Silicon 지원

macOS 빌드 타겟을 `electron-builder.yaml`에 정의했다. 핵심은 **Intel(x64)과 Apple Silicon(arm64) 모두 지원**하는 것이다.

```yaml
mac:
  icon: build/icons/icon.icns
  category: public.app-category.developer-tools
  target:
    - target: dmg
      arch: [x64, arm64]
    - target: zip
      arch: [x64, arm64]

win:
  icon: build/icons/icon.ico
  target:
    - target: nsis
      arch: [x64]

linux:
  icon: build/icons
  category: Development
  target:
    - target: AppImage
    - target: deb
    - target: rpm
```

`arch: [x64, arm64]`를 지정하면 Electron Builder가 두 아키텍처에 대해 각각 네이티브 바이너리를 생성한다. 유니버설 바이너리가 아닌 아키텍처별 별도 빌드 방식이다.

---

## 플랫폼 호환성 요약

| 기능 | Windows | macOS | Linux |
|---|---|---|---|
| 터미널 실행 | `cmd.exe` | `child_process` / `osascript` | `bash` |
| Zip 유틸 | `zip.exe` (번들) | 네이티브 `zip` | 네이티브 `unzip` |
| 앱 패키징 | NSIS (.exe) | DMG (x64 + arm64) | AppImage / DEB / RPM |
| 윈도우 관리 | 기본 | Dock `activate` 핸들링 | 기본 |
| 페이지 프로토콜 | `file://` | `app://` | `app://` |
| SSH 터널 | cmd 대화형 | `osascript` Terminal | bash |

---

## 제거된 의존성

macOS 대응 과정에서 꽤 많은 의존성을 정리할 수 있었다.

| 패키지 | 사유 |
|---|---|
| `node-pty` | child_process로 대체 |
| `xterm` | PTY 제거에 따라 불필요 |
| `xterm-addon-fit` | 동일 |
| `xterm-addon-ligatures` | 동일 |
| `ansi-escapes` | ANSI 파싱 로직 제거 |
| `jszip-sync` / `file-saver` | 네이티브 zip 사용 |

---

## 마무리

이 작업을 돌아보면, "Windows 전용 앱을 macOS로 포팅한다"는 것은 결국 **플랫폼 종속적인 가정을 하나씩 찾아서 추상화하는 과정**이었다.

가장 큰 교훈은 Terminal 컴포넌트에서 얻었다. 250줄의 복잡한 PTY 코드가 있었지만, 실제로 필요한 것은 "명령어를 실행하고 결과를 받는 것"뿐이었다. 요구사항을 다시 정의하니 코드가 62% 줄었고, 동시에 크로스 플랫폼 호환성도 확보되었다.

Electron 앱의 macOS 포팅을 계획하고 있다면, 빌드 설정(`electron-builder.yaml`)보다 **런타임에서 플랫폼을 가정하는 코드**를 먼저 점검하는 것을 추천한다. `process.platform` 분기가 필요한 지점은 대부분 다음 네 곳이다.

1. 셸 명령어 실행 방식
2. 파일 경로 및 바이너리 위치
3. 윈도우 라이프사이클 이벤트
4. 네이티브 모듈 의존성

---

## 참고 링크

- [Electron - BrowserWindow API](https://www.electronjs.org/docs/latest/api/browser-window){:target="_blank"}
- [Electron - app 이벤트 (activate)](https://www.electronjs.org/docs/latest/api/app#event-activate-macos){:target="_blank"}
- [Electron - protocol.registerFileProtocol](https://www.electronjs.org/docs/latest/api/protocol#protocolregisterfileprotocolscheme-handler){:target="_blank"}
- [Electron Builder - macOS 빌드 설정](https://www.electron.build/configuration/mac){:target="_blank"}
- [Electron Builder - Multi-Platform Build](https://www.electron.build/multi-platform-build){:target="_blank"}
- [Node.js child_process 공식 문서](https://nodejs.org/api/child_process.html){:target="_blank"}
- [AppleScript - Terminal 제어](https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptLangGuide/introduction/ASLR_intro.html){:target="_blank"}
