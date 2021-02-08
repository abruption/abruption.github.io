---
title: Apple Silicon Mac에 Jekyll 설치하기
author: LEE YEONGUK
date: 2021-02-08 09:30:00 +0900
categories: [Programming, Mac]
tags: [M1, Silicon, Apple, Mac, Jekyll]
---

Intel 맥에서 Apple Silicon M1 맥으로 기변한 이후에 Github Page에 글을 작성하기 위해 여러차례 시도해보았지만 실패했었던 Jekyll 설치방법에 대해 공유하고자 글을 작성해본다.

>현재 Homebrew 설치버전은 3.0으로 Apple Silicon M1을 Native로 지원하는 버전이다.

일단 Jekyll을 사용하기 위해서는 Ruby와 rbenv(또는 rvm)을 설치해야 하는데, 해당 글에는 rbenv를 설치하는 방법을 서술 할 예정이다.

<br />

## Homebrew 설치하기

맥에서는 Homebrew를 이용하여 간단하게 설치할 수 있으므로, Homebrew 설치방법은 기존에 작성했던 게시글을 참고하자.   
<https://blog.abruption.ml/posts/M1-Mac-Install-Homebrew/>

<br />

## rbenv 설치하기

rbenv는 Ruby의 버전을 독립적으로 사용하도록 도와주는 패키지이다.   
기본적으로 Mac에서는 2.6.x 버전의 Ruby가 설치되어 있는데, 우리는 2.7.2 버전을 사용 할 예정이다.

~~~bash
# rbenv와 ruby-build 설치
brew install rbenv

# Shell과 rbenv 통합 설정
rbenv init
~~~

여기까지 마쳤다면 아래 명령줄 중 하나를 .zshrc 경로 또는 터미널에 입력한다.

~~~bash
# Option 1
echo "eval "$(rbenv init -)""

# Option 2 (add to .zshrc)

# Example aliases
# alias zshconfig="mate ~/.zshrc"
# alias ohmyzsh="mate ~/.oh-my-zsh"
eval "$(rbenv init -)"
~~~

그 이후, .zshrc에 입력한 설정을 적용해주기 위해 `source ~/.zshrc`를 터미널에 입력해주고 다음 단계를 진행한다.

~~~bash
rbenv install 2.7.2
rbenv global 2.7.2
ruby -v
~~~

순차적으로 위의 명령어를 입력한 다음 `ruby -v`의 결과가 아래와 같이 나오면 rbenv는 설치가 성공한 것이다.

~~~bash
ruby 2.7.2p137 (2020-10-01 revision 5445e04352) [arm64-darwin20]
~~~

<br />

## Jekyll 설치하기

Github Page에 포함된 디렉토리에 들어가서 아래의 명령을 터미널에 입력해준다. (디렉토리가 없다면 Git clone 명령을 선행한 뒤 진행한다.)

~~~bash
gem install bundler jekyll
bundle update
bundle install
~~~

이후 설치가 완료되었다면 해당 디렉토리 경로에서 `bundle jekyll exec s` 명령어를 입력해 정상적으로 실행되는지 확인해본다.