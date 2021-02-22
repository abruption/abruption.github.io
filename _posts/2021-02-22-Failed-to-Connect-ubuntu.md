---
title: Failed to connect to https://changelogs.ubuntu.com/meta-release-lts 오류 발생 시 해결법
author: abruption
date: 2021-02-22 11:19:00 +0900
categories: [Programming, Ubuntu]
tags: [Linux, Ubuntu, Azure]
---

최근 Azure for Student Plan에 가입해서 우분투 서버를 하나 더 사용중에 있던 중, 18.03 LTS에서 20.04 LTS로 업그레이드 후에 자주 일어나는 일이라서 문제가 또 발생하면 보기 위해서 글을 적어보려고 한다.

<br />

![ERROR](/assets/img/sample/connect_error.png)

현재 Ubuntu 서버에 로그인을 하게 되면 위와 같은 문제가 뜨는데 아래의 명령어를 이용하면 해결된다고 한다.

~~~bash
sudo rm -f /var/lib/ubuntu-release-upgrader/release-upgrade-available 
sudo /etc/update-motd.d/91-release-upgrade
~~~

<br /> <br />

> 참고링크 : <https://askubuntu.com/questions/919441/failed-to-connect-to-http-changelogs-ubuntu-com-meta-release>