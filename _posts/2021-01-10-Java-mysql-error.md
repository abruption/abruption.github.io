---
title: JAVA-MySQL 연동 시 Public key retrieval is not allowed 오류 해결법
author: LEE YEONGUK
date: 2021-01-10 23:10:00 +0900
categories: [Programming, Java]
tags: [Java, MySQL]
---

## Public key retrieval is not allowed 오류 해결법
~~~java
Exception in thread "main" 
java.sql.SQLNonTransientConnectionException: Public Key Retrieval is not allowed
~~~

<br/>

MySQL 8.x 버전 이상에서 발생하는 문제로 위와 같은 오류가 발생한다.   
이럴때 JDBC URL에 `allowPublicKeyRetrieval=true&useSSL=false`를 추가한다.

~~~java
String url = "jdbc:mysql://localhost:3306/DB명?&useSSL=false&allowPublicKeyRetrieval=true&useUnicode=true&serverTimezone=UTC"
~~~

