---
title: AWS Lambda란 무엇인가?
author: abruption
date: 2021-09-03 14:25:00 +0900
categories: [Programming, AWS]
tags: [Lambda, Serverless, AWS]
---

## AWS Lambda란?
`AWS Lambda`는 2014 Re:invent 콘퍼런스 기조연설에서 발표한 서버리스 컴퓨팅 서비스이다.
지원하는 언어는 2021.08 기준 C#, PowerShell, Go, Java, Node.js, Python, Ruby이며 할당한 리소스(Lambda의 메모리 자원은 128MB부터 10,240MB까지 설정할 수 있으며 메모리 크기에 비례하여 CPU 용량과 기타 리소스 크기가 증가하는 방식을 취함)와 실행시간(1ms당)에 따라 요금이 부과되는 방식을 택하고 있다.  

Lambda는 기본적으로 `Event-Driven` 방식으로 동작하는데 API 게이트웨이와 엘라스틱 로드 밸런서를 이용하여 HTTP(S) 요청을 처리할 수 있으며, S3 Object, DynamoDB, Kinesis 등에서 발생하는 이벤트를 트리거로 실행하는 것도 가능하다.  

AWS Lambda와 유사한 컴퓨팅 서비스로는 Microsoft Azure의 `Azure Functions`, GCP(Google Cloud Platform)의 `Cloud Functions`, `Cloud Run` 등이 있으며, Kubernetes 위에서 Severless 실행 환경을 직접 구축할 수 있는 `Knative` 등이 있다.

<br />

### Lambda의 장점
- 빠른 개발   
    → 오직 비즈니스 로직에만 신경쓰면서 코드만 Lambda에 업로드하면 되므로 개발 속도와 서비스 출시를 앞당길 수 있다.
- 자동 확장  
	 → AWS에서 자동적으로 Auto-Scaling 하기때문에 유입되는 요청에 갯수에 따라 서버를 관리할 필요가 없다.
- 쉬운 운영 관리  
    → 자동 확장의 연장선상으로 AWS에서 자동으로 관리하기때문에 서버 운영에 대한 부담이 없다.
- 비용 절감  
    → 이벤트 기반 실행이므로 사용한 시간만 비용을 지불하면 되기때문에 서버를 운영하는 것보다 저렴하다.
- 쉬운 유지보수  
    → Lambda는 기능 단위로 함수를 작성하기때문에 코드 유지보수나 추가 기능 개발이 용이하다.

<br />

### Lambda의 단점
- 상태 제한   
	→ Lambda는 내부적으로 사용하지 않는 리소스는 제거하기 때문에 Running 상태를 계속해서 유지하지 않는다.
- DoS(Denial of Service, 서비스 거부)   
	→ Lambda는 동시 실행 횟수는 1,000*으로 제한하므로 동일한 계정에서 Load Test는 Lambda의 실행 횟수를 높여 DoS 공격을 만들 수 있다.
- 제한된 실행 시간   
	→ Lambda는 설정한 실행 시간이 넘어가면 Timeout이 발생하여 실행이 중단된다.
- Cold Start   
	→ Lambda는 처음 실행하거나, 일정시간 경과 후 실행 시에 내부적으로 Code Download, 부트스트래핑 등의 절차를 거치므로 리소스를 구성하는 시간이 필요하다.
- 모니터링 및 로그   
	→ AWS의 CloudWatch를 통해 확인할 수 있으나 접근성과 가독성 측면에서 불편한 측면이 있다.

> 동시 실행 횟수는 Region에 따라 상이 [Developer Guide: Lambda function scaling](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/invocation-scaling.html)

<br />

### Lambda의 동작 순서
![](https://miro.medium.com/max/700/0*ajxqAxlS9CcOtZqa.png)
1. Lambda Function Write & Deploy
2. Load Code as .ZIP or S3
3. Start new container
4. Bootstrap Code(Compile, load code to container)
5. Run Code
> 1~3까지는 Cold Start, 4~5까지는 Warm Start에 해당된다.

<br />

### Cold Start 
이벤트가 (처음) 호출되어 Lambda가 실행될 때 실행할 컨테이너가 준비되어 있지 않은 상태   
    → 실행 콘텍스트 설정 및 부트스트래핑 실행을 위한 과정  

비유하자면 꺼져있는, 완전히 식어버린 컴퓨터를 새로 킨다는 의미

<br />

### Warm Start 
일정 시간 대기 중에 호출이 발생하면 대기 중인 컨테이너가 바로 처리하는 것

<br />

### Cold Start에 대한 해결방법 
- Lambda의 사양(Memory)를 올린다.	  
	→ Memory와 CPU 사양은 비례하기때문
- AWS의 Provisioned Concurrency를 사용한다.   
	→ 일반적인 함수 호출과 동일한 요율의 비용이 부과
- 주기적으로(5~10분 이내) Lambda를 CloudWatch Event 사용 등의 방법으로 호출   
	→ 단, 호출에 따른 비용이 발생할 수 있으므로 주의
- Container가 계속 활성화 되어 있도록 일정량의 요청이 꾸준히 들어오는 것   
	→ 요청이 일정 수준 이상이라면 EC2가 더 유리할 수도 있다

<br />

### 언어마다 다른 Cold Start
![](https://theburningmonk.com/wp-content/uploads/2017/06/lambda-coldstart-04.png)

이미지에는 나오지 않는 Go와 Python, Nodejs는 빠른 Cold Start Time을 가지고 있지만, Java와 C#은 느린 Cold Start을 확인할 수 있다.

→ Go는 컴파일 언어이지만 컴파일러의 속도가 매우 빨라 인터프리터 언어처럼 쓸 수 있다는 장점때문에 다른 인터프리터 언어인 JavaScript나 Python과 같이 Cold Start Time이 짧다. 그 외 Java나 C#은 컴파일 언어로 컴파일 시간이 추가되기때문에 Cold Start Time이 타 언어보다 길 다.


<br />

### 메모리마다 다른 Cold Start
![](https://theburningmonk.com/wp-content/uploads/2017/06/lambda-coldstart-07-1024x800.png)

설정한 메모리에 따라서도 Cold Start Time이 다르며, 1024로 설정했을 때가 가장 좋은 효율을 보여주고 있다. 

→ 메모리 크기를 낮게 잡으면 실행시간이 길어져 타임아웃이 발생할 수 있고, 메모리를 높게 잡으면 비용이 늘어나 고 효율이 좋지 않기때문에 코드에 알맞는 타임아웃과 메모리를 할당하는 것이 중요하다.

<br />

## References
- [AWS Lambda 동작 순서 (feat. Warm start vs Cold start)](https://velog.io/@milkcoke/Lambda-Cold-start)
- [AWS Lambda를 시작하기 전에 알았으면 좋았을것들](https://medium.com/harrythegreat/aws-lambda를-시작하기-전-알았으면-좋았을것들-788bd3b3bdd2)
- [Serverless Architecture AWS Lambda: 2 Comprehensive Criteria](https://hevodata.com/blog/serverless-architecture-aws-lambda/)
- [aws lambda – compare coldstart time with different languages, memory and code sizes](https://theburningmonk.com/2017/06/aws-lambda-compare-coldstart-time-with-different-languages-memory-and-code-sizes/)