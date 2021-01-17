---
title: Mac에서 기본 Python 기본버전 설정하기
author: LEE YEONGUK
date: 2021-01-17 20:10:00 +0900
categories: [Programming, Python]
tags: [Python, Mac]
---

Mac에서 Python을 사용하려고 `HomeBrew`나 `python -V` 명령어를 이용하게 되면 기본적으로 설치되어 있는 2.7버전을 나타낸다.
보통 Python을 사용할 때는 3버전 이상을 권장하기때문에 `HomeBrew`를 통해 설치하게되는데 일단 HomeBrew 설치방법에 대해서 살펴보자.

## HomeBrew 설치하기

아래와 같이 Terminal을 실행하여 아래의 명령어를 입력하여 HomeBrew를 설치한다.

~~~bash
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
~~~


## HomeBrew로 Python 설치하기

아래의 명령어를 입력하면 Python을 설치할 수 있으며 2020년 1월 17일 기준으로 3.9 버전이 설치된다.

~~~bash
brew install python
~~~


## Mac에서 Python 기본버전 변경하기

터미널에서 `python` 명령어를 입력하면 아래와 같은 결과가 나오고,
![python](/assets/img/sample/python.png)

`python3` 명령어를 입력하면 아래와 같은 결과가 나온다.
![python3](/assets/img/sample/python3.png)

두 개의 버전이 공존하는 상황이지만 `python -V` 명령어를 사용하면 구버전인 2.7 버전이 나오게 된다.   
최신 버전인 Python 3.8.5를 기본버전으로 변경하기 위해서는 다음 명령어 중 하나를 선택하여 적용한다.


### Python3 설치위치 확인하기

~~~bash
>> which python3
/usr/local/bin/python3
~~~


- bash_profile을 수정하는 방법
~~~bash
  >> vi ~/.bash_profile
  alias python='/usr/local/bin/python3'

  >> python -V
  Python 3.8.5
~~~

- bash_profile을 수정하지 않고 추가하는 방법
~~~bash
echo alias python='/usr/local/bin/python3' > ~/.bash_profile
~~~

둘 중 하나의 방법을 사용 한 뒤 `python -V` 명령어를 입력하면 Python 3.x 버전이 나오는 것을 확인할 수 있다.
