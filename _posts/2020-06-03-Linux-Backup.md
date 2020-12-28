---
title: Linux에서 Crontab을 통한 자동백업
author: LEE YEONGUK
date: 2020-06-03 18:15:00 +0900
categories: [Leature, Enterprise Server]
tags: [Linux, WAS]
---

## 디렉토리 생성 및 쉘스크립트 작성을 위한 ROOT 계정 로그인
- `su-root` 명령어를 통해 비밀번호 입력 후 `root` 계정으로 로그인 

![linux1](/assets/img/sample/linux1.jpg)

<br/>

## 쉘스크립트 위치와 백업 저장위치 생성 (mkdir 명령어 이용)
- cd 명령어를 이용하여 `/home` 디렉토리에서 `mkdir backup` 실행
- cd 명령어를 이용하여 `/home/inhatc` 디렉토리에서 `mkdir scripts` 실행

![linux2](/assets/img/sample/linux2.jpg)

<br/>

## backup 디렉토리의 권한 설정 (chmod 명령어 이용)
![linux3](/assets/img/sample/linux3.jpg)

<br/>

## 백업을 위한 쉘스크립트 작성 후 권한설정
~~~bash
#!/bin/bash

today=$(date +%Y%m%d)
tar zcfp /home/backup/backup-$today.tar.gz /home/inhatc 
find /home/backup/ -type f -mtime +5 | sort | xargs rm -f
~~~

![linux4](/assets/img/sample/linux4.jpg)

- `today=$(date +%Y%m%d)` : 백업파일명의 생성날짜를 추가하기 위해 `today`라는 변수를 생성하고, 그 안에 `date` 명령어를 활용하여 날짜값을 넣어준다. 이후에 생성되는 백업파일은 `backup-YYYYMMDD.tar.gz` 의 이름으로 명명된다.
- `tar zcfp /home/backup/backup-$today.tar.gz /home/inhatc` : `/home/inhatc`의 모든 파일을 `/backup`에 `backup-DATE.tar.gz`라는 파일로 백업(저장) 시킨다.
- `find /home/backup/ -type f -mtime +5 | sort | xargs rm -f` : `/home/backup`에서 5일이 지난 파일을 삭제하도록 한다.
- `chmod 700 backup.sh` : 쉘스크립트 실행을 위해 `chmod` 명령어를 이용하여 사용자에게만 읽기, 쓰기, 실 행 모든 권한을 부여한다.
- `./backup.sh` : 쉘스크립트가 동작하도록 실행시킨다.


<br/>

### tar 명령어
- 여러개의 파일을 하나로 묶는 명령어(=압축)
- `# tar [OPTION] [ADD tar FILE] [FILE LIST]`
- `# tar [OPTION] [tar FILE] -C [UNZIP LOCATION]`


<br/>

### find 명령어
- 파일 및 디렉토리를 검색할 때 사용하는 명령어
- `# find [OPTION...] [PATH] [EXRPESSTION..]`
- `type`은 지정된 파일 타입에 해당되는 파일을 검색하는 표현식이다.


<br/>

## Crontab을 이용하여 스케줄러를 생성
- 모든 엔트리 필드는 공백으로 구분된다.
- 한 줄당 하나의 명령 (두줄로 나눠서 표시할 수 없음)
- #으로 시작하는 줄을 실행하지 않는다. (=주석)
- `# crontab [OPTION] FILE`
- `# crontab [OPTION]`


<br/>

### Crontab 파일 형식
- 첫 번째 필드는 분을 의미하며 범위는 0-59이다.
- 두 번째 필드는 시를 의미하며 범위는 0-23이다.
- 세 번째 필드는 일을 의미하며 범위는 0-31이다.
- 네 번째 필드는 월을 의미하며 범위는 1-12이다.
- 다섯 번째 필드는 요일을 의미하며 범위는 0-7이다. (0 또는 7 = 일요일, 1=월요일, 2=화요일, ...)
- 여섯 번째 필드는 명령어를 의미하며 실행할 명령을 한 줄로 작성한다.

![linux5](/assets/img/sample/linux5.jpg)

- `crontab -e` : `-e`는 Edit user's crontab으로 Crontab을 생성하는 역할을 한다. (대개 vi 편집기 이용)
- 일일 백업(매일 17시 백업)을 위해 crontab 파일에 `0 17 * * * /home/inhatc/scripts/backup.sh`라고 작성하고 저장한다.


<br/>

## 작성한 쉘 스크립트가 정상적으로 실행되어 백업 파일이 생성되었는지 확인
- 정상적으로 백업파일이 생성되었으며, `today` 변수를 이용하여 파일명에 생성날짜를 추가한 것도 정상적으로 동작하는 것을 확인할 수 있다.

![linux6](/assets/img/sample/linux6.jpg)

- 생성된 백업파일을 내용을 확인하기 위해서는 아래 `tar` 명령어를 사용하면 된다.
- `tar tvf FILENAME.tar` 또는 `tar tvfz FILENAME.tar.gz`

<br/>

**[실행결과]**

~~~bash
[root@ip-172-31-35-139 backup]# tar tvf backup-20200603.tar.gz
drwx------ inhatc/inhatc     0 2020-06-03 16:05 home/inhatc/
-rw-r--r-- inhatc/inhatc    18 2019-08-30 14:30 home/inhatc/.bash_logout
-rw-r--r-- inhatc/inhatc   141 2019-08-30 14:30 home/inhatc/.bash_profile
-rw-r--r-- inhatc/inhatc   312 2019-08-30 14:30 home/inhatc/.bashrc
-rw------- inhatc/inhatc 38143 2020-06-02 23:45 home/inhatc/.bash_history
-rw-rw-r-- inhatc/inhatc 43180 2020-04-10 14:40 home/inhatc/201644091_20200401.txt
drwxrwxr-x inhatc/inhatc     0 2020-04-15 16:20 home/inhatc/work0417/
drwxrwxr-x inhatc/inhatc     0 2020-04-14 22:50 home/inhatc/work0417/Jupiter/
drwxrwxr-x inhatc/inhatc     0 2020-04-14 22:50 home/inhatc/work0417/Venus/
-rwx------ inhatc/inhatc     0 2020-04-14 22:52 home/inhatc/work0417/Mars
lrwxrwxrwx inhatc/inhatc     0 2020-04-14 22:59 home/inhatc/work0417/hosts -> /etc/hosts
-rw-rw-r-- inhatc/inhatc  6064 2020-04-15 11:45 home/inhatc/work0417/Earth
drwxrwxr-x inhatc/inhatc     0 2020-05-05 14:16 home/inhatc/work0508/
-rw-r--r-- inhatc/inhatc 141445 2020-05-05 14:16 home/inhatc/work0508/IMG_3137.JPG
drwxr-xr-x root/root          0 2020-05-20 15:59 home/inhatc/devtest/
-rw-r--r-- root/root         70 2020-05-11 16:30 home/inhatc/devtest/HelloWorld.c
-rw-r--r-- root/root         72 2020-05-11 16:30 home/inhatc/devtest/HelloWorld.cpp
-rw-r--r-- root/root        110 2020-05-11 16:30 home/inhatc/devtest/HelloWorld.java
-rw-r--r-- root/root        426 2020-05-11 16:30 home/inhatc/devtest/HelloWorld.class
-rwxr-xr-x root/root      16672 2020-05-11 16:33 home/inhatc/devtest/Hello_c
-rwxr-xr-x root/root      17168 2020-05-11 16:34 home/inhatc/devtest/a.out
-rwxr-xr-x root/root      17168 2020-05-11 16:36 home/inhatc/devtest/Hello_cpp
-rw-r--r-- root/root         90 2020-05-20 15:59 home/inhatc/devtest/argtest.sh
drwxrwxr-x inhatc/inhatc      0 2020-05-20 17:13 home/inhatc/work0522/
-rwxrwxrwx inhatc/inhatc    159 2020-05-20 16:26 home/inhatc/work0522/argtest.sh
-rwxrwxrwx inhatc/inhatc    289 2020-05-20 16:49 home/inhatc/work0522/datetest.sh
-rwxrwxrwx inhatc/inhatc    231 2020-05-20 16:26 home/inhatc/work0522/functest.sh
-rwxrwxrwx inhatc/inhatc    332 2020-05-20 17:13 home/inhatc/work0522/array.sh
-rw-rw-r-- inhatc/inhatc  35382 2020-05-27 22:57 home/inhatc/Trycommand
drwxrwxr-x inhatc/inhatc      0 2020-06-03 16:34 home/inhatc/scripts/
-rwx------ root/root        152 2020-06-03 16:34 home/inhatc/scripts/backup.sh
~~~