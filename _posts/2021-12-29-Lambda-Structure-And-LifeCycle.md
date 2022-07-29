---
title: [AWS Lambda] Lambda 구조 및 LifeCycle
author: abruption
date: 2021-12-29 09:38:00 +0900
categories: [Programming, AWS]
tags: [JavaScript, Node.js, Lambda, Serverless]
---

## Lambda의 기본 객체
| Param | Type | Description |
| --- | --- | --- |
| event | <code>object</code> | [링크 가기](https://docs.aws.amazon.com/ko_kr/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-set-up-lambda-proxy-integration-on-proxy-resource) |
| context | <code>object</code> | [링크 가기](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/nodejs-prog-model-context.html)|

<br />

## Lambda의 LifeCycle
1. **`Init`**: Lambda로 구성된 리소스로 실행 환경을 만들거나 고정 해제하고, 함수의 모든 계정의 코드를 다운로드하고, 모든 익스텐션을 초기화하고, 런타임을 초기화한 다음 함수의 초기화 코드(기본 핸들러 외부의 코드)를 실행한다. `Init` 단계는 첫 번째 호출 중에 발생하거나, [프로비저닝된 동시성](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/provisioned-concurrency.html)을 사용하도록 설정한 경우 함수 전에 발생한다.
   - `Extension Init`: 모든 익스텐션 시작
   - `Runtime Init`: 런타임 부트스트랩
   - `Function Init`: 함수의 정적 코드 실행  

 - `Init` 단계는 10초로 제한되며, 세 작업이 모두 10초 이내에 완료되지 않으면 Lambda는 첫 번째 함수 호출 시 `Init` 단계를 다시 시도한다.  
 
2. **`Invoke`**: Lambda는 함수 핸들러를 호출한다. 함수의 실행이 완료된 후 Lambda는 다른 함수 호출을 처리할 준비를 한다.
   - 함수의 제한 시간 설정은 전체 `Invoke` 단계의 기간을 제한한다. 기간은 모든 호출 시간 (런타임 + 익스텐션)의 합계이며 함수와 모든 익스텐션의 실행이 완료될 때까지 계산되지 않는다.  

   - `Invoke` 단계 중에 Lambda 함수가 충돌되거나 시간이 초과되면 Lambda는 실행 환경을 재설정한다. 재설정은 `Shutdown` 이벤트처럼 동작한다.   

3. **`Shutdown`**: Lambda 함수가 일정 기간 동안 호출을 받지 않으면 이 단계가 트리거된다. `Shutdown` 단계에서는 Lambda는 런타임을 종료하고 익스텐션이 완전히 중지되도록 알림을 보낸 다음 환경을 제거한다. Lambda는 각 익스텐션에 `Shutdown` 이벤트를 보내, 환경이 곧 종료됨을 익스텐션에게 알린다.
   - `Runtime Shutdown`
   - `Extension Shutdown`
   - Lambda는 런타임을 종료하려고 할 때 Shutdown 이벤트를 런타임에 전송한 다음 각 외부 익스텐션에 전송한다. 익스텐션은 이 시간 동안 최종 정리 작업을 수행할 수 있다.  
  
   - **기간**: 전체 `Shutdown` 단계는 2초로 제한된다. 런타임 또는 익스텐션이 응답하지 않는 경우, Lambda는 신호(`SIGKILL`)를 통해 이를 종료한다.  

    - 함수와 모든 익스텐션이 완료된 후 Lambda는 다른 함수 호출을 예상하여 일정 시간 동안 실행 환경을 유지한다. 실제로 Lambda는 실행 환경을 재개합니다. 함수가 다시 호출되면 Lambda는 재사용을 위해 환경을 재개한다. 실행 환경을 재사용하는 것은 다음과 같은 의미를 가진다.

        - 함수의 핸들러 메서드 외부에서 선언된 객체는 함수가 다시 호출될 때 추가로 최적화가 되도록 초기화된 상태로 유지된다. 예를 들어 Lambda 함수가 연결을 재설정하는 대신 데이터베이스 연결을 설정하면 원래 연결이 후속 호출에 사용된다. 코드에 로직을 추가하여 연결을 새로 생성하기에 앞서 연결이 이미 존재하는지 확인하는 것이 좋다.
        - 각 실행 환경은 `/tmp` 디렉터리에 512MB의 디스크 공간을 제공한다. 디렉터리 콘텐츠는 실행 환경이 일시 중지되어도 그대로 유지되기 때문에 일시적인 캐시를 여러 호출에 사용할 수 있다. 코드를 추가하여 캐시에 저장된 데이터가 포함되어 있는지 확인할 수 있다. 배포 크기에 대한 자세한 내용은 [Lambda 할당량](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/gettingstarted-limits.html) 단원을 참조
        - Lambda 함수에 의해 초기화되었고 함수 종료 시 완료되지 않은 백그라운드 프로세스 또는 콜백 Lambda가 실행 환경을 재사용하는 경우에 재개된다. 코드가 종료되려면 먼저 코드의 백그라운드 프로세스 또는 콜백이 완료되어야 한다.

