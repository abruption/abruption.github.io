---
title: Mac 터미널 접근 시 Last login ~ ttys000 없애기
author: abruption
date: 2021-01-22 16:19:00 +0900
categories: [Programming, Mac]
tags: [Unix, Mac, Ubuntu]
---

최근 Terminal 대신 iTerm2를 설치하면서 이리저리 꾸미고 바꿔보던 중 해당 문구가 거슬려서 없애는 방법을 찾기로 하면서 안 사실인데, `Last login: Fri Jan 22 16:15:41 on ttys000` 같은 로그인 시 뜨는 문구를 MOTD(Message Of The Day)라고 하는 모양이다.

<br />

여튼 해당 문구를 삭제하는 방법은 홈디렉토리에 `.hushlogin` 파일을 만들면 문구가 뜨지 않는다고 한다.
빈 내용이어도 상관이 없으니 nano 에디터나 vi 에디터를 사용해도 되고, 간단하게 `touch .hushlogin` 명령어를 입력하면 된다.

<br />

차후에 해당 MOTD를 다시 살리고 싶다면 `rm .hushlogin`을 하면 돌아온다.