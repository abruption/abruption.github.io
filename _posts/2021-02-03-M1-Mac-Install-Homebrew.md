---
title: Apple M1 Mac에서 Homebrew 설치하기
author: LEE YEONGUK
date: 2021-02-03 15:30:00 +0900
categories: [Programming, Mac]
tags: [M1, Silicon, Apple, Mac]
---

이 글을 읽고 있다면 당신은 M1 Mac을 사용하고 있을 것이다.
macOS 10.15 Catalina부터 기본 쉘(Shell)이 `bash`에서 `zsh`로 변경되었기때문에 zsh 기반으로 설명 할 예정이다.

먼저 Homebrew는 2020년 12월 1일 2.6.0 버전에서 M1/Apple Silicon/ARM을 Native로 지원하기 시작했다
<https://brew.sh/2020/12/01/homebrew-2.6.0/>

또한, M1/Apple Silicon/ARM 지원 기기들의 Homebrew 기본 디렉토리가 기존의 `/usr/local`에서 `/opt/homebrew`로 변경되었으니 설치 시 참고하길.

Homebrew는 Terminal에서 아래의 명령어를 이용하면 자동으로 설치되며, Command Line Tool (CLT) for Xcode 설치가 선행되어야 한다.

~~~bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
~~~

물론 CLT가 설치되지 않아도 Homebrew 설치과정에서 알아서 설치해주며, Homebrew 설치가 다 되었다면 `brew help` 명령어를 입력하여 정상적으로 설치되었는지 확인해본다.

이후, Homebrew의 설치경로가 달라졌기때문에 해당 경로를 잡아줘야하므로 아래의 명령어 중 하나를 터미널에 입력하며 된다.

~~~bash
echo "export PATH=/opt/homebrew/bin:$PATH" >> ~/.zshrc
~~~

~~~bash
>> vi ~/.zshrc
# Example aliases
# alias zshconfig="mate ~/.zshrc"
# alias ohmyzsh="mate ~/.oh-my-zsh"
export PATH=/opt/homebrew/bin:$PATH
~~~

명령어를 입력한 뒤 터미널에 `source ~/.zshrc`를 입력하여 적용시키면 최종 설치는 끝이난다.