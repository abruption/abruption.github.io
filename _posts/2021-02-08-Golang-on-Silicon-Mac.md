---
title: Apple Silicon Mac에 Golang 설치하기
author: abruption
date: 2021-02-08 13:30:00 +0900
categories: [Programming, Mac]
tags: [M1, Silicon, Apple, Mac, Golang, Google]
---

Apple Silicon M1 맥북을 사용하면서 M1 Native 앱인지, 아닌지에 대해 신경쓰는 일이 많아졌다.   
아래와 같이 Apple Silicon M1에 호환되는 프로그램인지 아닌지 알려주는 홈페이지까지 있으니 나만의 기우는 아닌가보다.
<https://isapplesiliconready.com>

<br />

## Golang 설치하기

현재 Golang의 최신버전은 1.15.8로 ARM Mac을 공식적으로 지원하지 않는다.
그래서 Homebrew나 Golang 공식사이트에서 Go를 다운받으면 아마 로제타를 통해서 돌아가지 않을까 싶다.

다만, 현재 1.16 RC(Release Candidate version)버전으로 ARM Mac을 지원하고 있기때문에 학부수준의 공부목적으로는 문제없이 사용할 수 있을 것으로 생각한다.

- 기존에 Go가 설치되어 있다면 터미널에서 아래의 명령어를 입력하자.

~~~bash
go get golang.org/dl/go1.16rc1
~~~

- 기존에 Go가 설치되어 있지 않다면 아래의 링크를 누르면 바로 PKG 파일을 받을 수 있다.
<https://golang.org/dl/go1.16rc1.darwin-arm64.pkg>

<br />

설치되었는지 확인하려면 터미널에 `go version`를 입력하면 되고, 문제없이 설치가 되었다면 아래와 같은 화면을 볼 수 있을것이다.
![golang](/assets/img/sample/golang.png)