---
title: Mac의 기본 쉘(Shell) 변경하기 [bash → zsh]
author: abruption
date: 2021-02-09 16:33:00 +0900
categories: [Programming, Mac]
tags: [Mac, Shell, bash, Zsh]
---

macOS 10.15 Catalina부터 기본 쉘(Shell)이 `bash`에서 `zsh`로 변경되어 Terminal 앱을 실행하면 아래와 같은 메세지가 뜨게된다.

~~~bash
Last login: Thu Feb 9 16:55:55 on ttys000

The default interactive shell is now zsh.
To update your acount to use zsh, please run 'chsh -s /bin/zsh'.
For more details, please visit https://support.apple.com/kb/HT208050.
~~~

별도의 설정을 하지 않으면 기본 쉘이 `bash`로 계속 유지되는데, 차후에 기술할 iTerm2이나 oh-my-zsh를 설치하기 위해서는 기본 쉘을 `zsh`로 변경해주는 것이 좋다.

## 기본 쉘 zsh로 변경하기

기본 쉘을 변경하기 위해서는 위 안내 메세지에 나온 것처럼 터미널에 아래와 같은 명령을 입력하면 된다.
~~~bash
# Option 1
$ chsh -s /bin/zsh

# Option 2
$ chsh -s $(which zsh)
~~~
> chsh는 "Change Shell"의 약어로 사용자가 사용하는 쉘을 변경하기 위한 명령어다.

<br />

**참고로 bash 쉘로 다시 변경을 하고 싶다면 `chsh -s /bin/bash`를 입력하면 된다.**