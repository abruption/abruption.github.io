---
title: MySQL 쉘스크립트 작성 시 비밀번호 경고 처리하기 (Warning :Using a password on the command line interface can be insecure)
author: abruption
date: 2021-01-17 21:15:00 +0900
categories: [Programming, Ubuntu]
tags: [Ubuntu, MySQL]
---

MySQL 5.6 이후 버전에서 쉘 스크립트를 실행하거나 mysqldump를 사용하면 `Warning: Using a password on the command line interface can be insecure` 오류가 발생한다.   
검색해보니 무시해도 상관이 없다고 하지만, 뭔가 오류가 뜨는게 거슬리다보니 해결방법을 찾던 중 까먹을까봐 블로그에도 올려본다.

~~~bash
mysql_config_editor set --login-path=root --host=localhost --user=root --password --port=3306
Enter password : [비밀번호 입력]
~~~

- --login-path : ID&PW를 담고 있는 일종의 접속명을 정의한다 (ex. root)
- --host : DB서버의 IP주소 (ex. 127.0.0.1, 외부IP 또는 도메인)
- --user : DB계정
- --password : 별도 기재 없이 차후에 해당 계정의 비밀번호 입력
- --port : 생략해도 되지만 기본포트가 아닌 다른 포트일 경우 기재필요

해당 명령어를 작성한 뒤, 쉘스크립트나 mysqldump를 사용할 때 `-u [계정명] -p [비밀번호]` 대신 위에서 작성한 `--login-path=[접속명]`을 넣으면 별도의 경고문구 없이 사용 가능하다.