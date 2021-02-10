---
title: Oh-My-Zsh과 iTerm2를 이용하여 Mac 터미널 꾸미기
author: LEE YEONGUK
date: 2021-02-10 08:55:00 +0900
categories: [Programming, Mac]
tags: [Mac, iTerm2, Oh-My-Zsh, Zsh]
---
**macOS 10.15 Catalina부터 기본 쉘(Shell)이 `bash`에서 `zsh`로 변경되었으므로,**   
**기본 쉘이 zsh가 아닌 경우 첨부링크를 참고하여 변경 후에 진행하기를 바란다.**   
<https://abruption.github.io/posts/Change-basic-shell-on-mac>

<br />

## iTerm2

`iTerm2`는 Mac의 기본 Terminal을 대체하는 프로그램이며 MacOS 10.14 이상에서 정상작동한다. 테마 변경이나 여러가지 기능들을 탑재하고 있어 나또한 Terminal 대신 사용중에 있다.  

설치는 아래 링크를 눌러 설치하거나, homebrew에서 아래의 명령어를 이용하여 설치하면 된다.

- iTerm2 공식 홈페이지 설치 링크 : <https://iterm2.com/downloads/stable/latest>

- Homebrew를 통한 iTerm2 설치 명령어 : `brew install --cask iterm2`

<br />

## Oh My Zsh
`Oh My Zsh`는 `Zsh` 구성을 관리하기 위한 오픈소스 커뮤니티기반의 프레임워크이고, 수백 개의 강력한 플러그인과 아름다운 테마를 활용할 수 있다고 공식 사이트에 설명되어있기도 하다.

이름에도 알 수 있듯이 Zsh를 기반으로 하기때문에 기본적으로 zsh가 설치되어 있어야 하고, Oh My Zsh를 다운받기 위해서는 `curl`이나 `wget`이 설치되어 있어야 한다.

다만, 맥은 curl이 기본 탑재되어 있으니 우리는 curl를 이용하도록 한다.   
>혹시나 wget을 설치하고 싶다면 homebrew가 설치되어 있는 환경에서 터미널에 `brew install wget` 를 입력하면 된다.


### Oh My Zsh 설치

아래 명령어 중 하나를 iTerm2에 입력하면 설치가 완료된다.

- curl로 설치하기
~~~bash
$ sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
~~~

- wget으로 설치하기
~~~bash
$ sh -c "$(wget https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"
~~~

<br />

### Zsh 테마 설정하기

![powerlevel10k](/assets/img/sample/powerlevel10k.gif)
Oh My Zsh에 적용 가능한 테마는 아래 링크에서 확인할 수 있는데 필자는 많고 많은 테마 중 `powerlevel10k` 테마를 적용 할 예정이다.   
<https://github.com/ohmyzsh/ohmyzsh/wiki/External-themes>   

<br />

일단 powerlevel10k 테마를 설치하기 위해 테마파일을 다운받기 위해 터미널에 아래와 같은 명령어를 입력한다.
~~~bash
$ git clone https://github.com/romkatv/powerlevel10k.git $ZSH/themes/powerlevel10k
~~~

<br />

그 이후, vi 편집기를 이용할 경우 `vi ~/.zshrc`, nano 편집기를 이용할 경우 `nano ~/.zshrc`를 입력하여 .zshrc 파일을 열고 `ZSH_THEME` 부분을 아래와 같이 수정한다.
~~~bash
# Set name of the theme to load --- if set to "random", it will
# load a random theme each time oh-my-zsh is loaded, in which case,
# to know which specific one was loaded, run: echo $RANDOM_THEME
# See https://github.com/ohmyzsh/ohmyzsh/wiki/Themes
ZSH_THEME="powerlevel10k/powerlevel10k"
~~~
<br />

![configure](/assets/img/sample/p10k_configure.png)
편집을 완료한 뒤 테마를 적용하기 위해 터미널에 `source ~/.zshrc`를 입력하면 위와 같이 powerlevel10k의 초기설정 창이 뜨게되는데, 설명에 따라 본인이 선택하면 된다.

<br />

> 설정을 모두 마치면 터미널이 변경되어 있을텐데 iTerm2의 폰트나 테마설정이 되어있지않다면 본인이 설정한 테마가 제대로 적용되어있지 않을테니 아래의 테마와 폰트 파일을 받아놓자.
>
- iTerm2-color-schemes 파일 : <https://github.com/mbadolato/iTerm2-Color-Schemes/zipball/master>
- MesloLGS NF Regular : <https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf>
- MesloLGS NF Bold : <https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf>
- MesloLGS NF Italic : <https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf>
- MesloLGS NF Bold Italic : <https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf>
>

<br />
이때는 iTerm2을 실행한 상태에서 아래의 순서대로 설정을 마치면 정상적으로 테마가 작동 할 것이다.

1. iTerm2 실행 후 `Command + ,`을 눌러 Preferences를 띄운다.
2. Profiles - Text에서 Font를 `MesloLGS NF`로 설정한다.
3. Profiles - Colors에서 Color Presets을 눌러 `Solarized Dark Theme`로 설정한다.

<br />

![apply_theme](/assets/img/sample/apply_theme.png)
모두 완료하면 위와 같이 powerlevel10k 테마가 깨지지 않고 잘 보일 것이다!