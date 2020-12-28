---
title: Ubuntu apt-get 업데이트 중 The following packages have been kept back 오류 해결법
author: LEE YEONGUK
date: 2020-12-28 21:50:00 +0900
categories: [Programming, Ubuntu]
tags: [Ubuntu, apt, package-management]
---

## The following packages have been kept back 오류 해결법

~~~bash
$ apt-get update && apt-get upgrade
...
The following packages have been kept back:
  gimp gimp-data libgegl-0.0-0 libgimp2.0
~~~

간혹 apt-get update && apt-get upgrade를 진행할 경우 위와 같은 오류가 뜨면서 업데이트가 진행되지 않는 경우가 있는데,  
이 문제는 설치 한 패키지 중 하나에서 종속성이 변경되면서 업그레이드를 실행하기 위해 새 패키지를 설치해야 할 경우에 뜬다.

<br/>

**이 경우에는 `sudo apt-get --with-new-pkgs upgrade` 명령어를 입력하여 해결하면 된다.**
<br />

>검색하다보면 `apt-get dist-upgrade`로도 가능하다고 하는데 일반적인 환경에서는 위험할 수 있다고 한다.

<br/><br/>

<https://qastack.kr/ubuntu/601/the-following-packages-have-been-kept-back-why-and-how-do-i-solve-it>