---
title: Mac에서 xcrun :error :invalid active developer path 문제 발생 시 해결 방법
author: abruption
date: 2021-02-03 15:40:00 +0900
categories: [Programming, Mac]
tags: [xcrun, xcode, Mac]
---

![xcrun](/assets/img/sample/xcrun.png)

어제까지 잘 되던 git이 macOS 11.2로 판올림 이후에 오류를 내뿜으며 정상적으로 작동하지 않는다.

해당 문제는 xcode CLI 설치문제라고는 하는데 왜 정상적으로 되던게 오류를 뿜는건지 원..

아무튼 해당 문제는 아래의 명령어를 이용하여 설치하면 정상작동 한다.

~~~bash
$ xcode-select --install
~~~