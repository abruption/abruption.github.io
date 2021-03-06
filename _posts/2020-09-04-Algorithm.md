---
title: 알고리즘(표현방법의 종류)이란?
author: abruption
date: 2020-09-04 13:02:00 +0900
categories: [Leature, Data Structure]
tags: [Algorithm, Data Structure]
---

##  알고리즘이란?

  - 알고리즘(Algorithm)은 문제해결방법을 추상화하여 단계적 절차를 논리적으로 기술해 놓은 명세서다.
  - 알고리즘은 다음과 같은 5가지 조건을 만족해야 한다.
    - 입력(input) : 알고리즘 수행에 필요한 자료가 외부에서 입력으로 제공될 수 있어야 한다.
    -   출력(output) : 알고리즘 수행 후 하나 이상의 결과를 출력해야 한다.
    -   명확성(definiteness) : 수행할 작업의 내용과 순서를 나타내는 알고리즘의 명령어들은 명확하게 명세되어야 한다.
    -   유한성(finiteness) : 알고리즘은 수행 뒤에 반드시 종료되어야 한다.
    -   효과성(effectiveness) : 알고리즘의 모든 명령어들은 기본적이며 실행이 가능해야 한다.  

<br/>

## 알고리즘의 표현 방법의 종류
    
1. 자연어를 이용한 서술적 표현 방법
2.  순서도(Flowchart)를 이용한 도식화 표현 방법
    -   장점 : 알고리즘의 흐름 파악이 용이함
    -   단점 : 복잡한 알고리즘의 표현이 어려움
3.  프로그래밍 언어를 이용한 구체화 방법
4.  가상코드(Pseudo-code)를 이용한 추상화 방법
    -   가상코드를 이용한 알고리즘의 표현
    -   가상코드, 즉 알고리즘 기술언어(ADL, Algorithm Description Language)를 사용하여 프로그래밍 언어의 일반적인 형태와 유사하게 알고리즘을 표현
    -   특정 프로그래밍 언어가 아니므로 직접 실행은 불가능
    -   일반적인 프로그래밍 언어의 형태이므로 원하는 특정 프로그래밍 언어로의 변환 용이

<br/>

### 기본 요소

-   기호 : 변수, 자료형 이름, 프로그램 이름, 레코드 필드 명, 문장의 레이블 등을 나타냄
-   자료형 : 정수형과 실수형의 수치 자료형, 문자형, 논리형, 포인터, 문자열 등의 모든 자료형 사용
-   연산자 : 산술연산자, 관계연산자, 논리연산자
-   지정문
    -   사용형식 : 변수 ← 값 ;
    -   지정 연산자(←)의 오른쪽에 있는 값(또는 식의 계산 결과 값이나 변수의 값)을 지정연산자(←)의 왼쪽에 있는 변수에 저장
-   조건문 : 조건에 따라 실행할 명령문이 결정되는 선택적 제어구조를 만든다
    -   if문
        -   형식 : if (조건식) then 명령문;
        -   형식2 : if(조건식) then 명령문1; else 명령문2;
        -   중첩 if문의 형식 : else if (조건식-2) then 명령문2; else 명령문3;
    -   case문
         -   여러 조건식 중에서 당되는 조건을 찾아서 그에 대한 명령문을 수행
         -   중접 if문으로 표현가능
-   반복문 : 일정한 명령을 반복 수행하는 루프(loop) 형태의 제어구조
    -   for문
        -   형식 : for (초기값; 조건식; 증감값) do 명령문 ;
        -   초기값 : 반복문을 시작하는 값
        -   조건식 : 반복 수행 여부를 검사하는 조건식
        -   증감값 : 반복 회수를 계산하기 이해서 반복문을 한번 수행할 때마다 증가 또는 감소시키는 값
    -    while문
        -   형식 : while (조건식) do 명령문;
        -   조건식이 참인 동안 명령문을 반복 수행
    -    do-while문
        -   형식 : do 명령문; while (조건식);
        -   일단 명령문을 한번 실행한 후에 조건식을 검사하여 조건식이 참인 동안 명령문을 반복 수행
- 함수문 : 처리작업 별로 모듈화하여 만든 부프로그램
    -   형식 : 함수이름(매개변수)  
        -   명령문;
        -   …
        -   return 결과값; end
    - 함수의 호출과 실행 및 결과값 반환에 대한 흐름제어

<br/>

## 알고리즘 성능 분석 방법

- 알고리즘 성능 분석 방법
    - 공간 복잡도
        - 알고리즘을 프로그램으로 실행하여 완료하기까지 필요한 총 저장공간의 양
            - 공간 복잡도 = 고정 공간 + 가변 공간
    - 시간 복잡도
        - 알고리즘을 프로그램으로 실행하여 완료하기까지의 총 소요시간
        - 시간 복잡도 = 컴파일 시간 + 실행시간
            - 컴파일 시간 : 프로그램마다 거의 고정적인 시간 소요
            - 실행시간 : 컴퓨터의 성능에 따라 달라질 수 있으므로 실제 실행시간 보다는 명령문의 실행 빈도수에 따라 계산
- 실행 빈도수의 계산
    - 지정문, 조건문, 반복문 내의 제어문과 반환문은 실행시간 차이가 거의 없으므로 하나의 단위시간으로 갖는 기본 명령문으로 취급