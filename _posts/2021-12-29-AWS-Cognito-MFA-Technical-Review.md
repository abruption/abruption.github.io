---
title: [AWS Cognito] MFA 인증 방식 기술검토 경과 정리
author: abruption
date: 2021-12-28 10:30:00 +0900
categories: [Programming, AWS]
tags: [Cognito, 2FA, MFA, Authentication]
---

# [AWS Cognito] MFA 인증 방식 기술검토 경과 정리

## [Cognito UserPool 인증 흐름](https://docs.aws.amazon.com/ko_kr/cognito/latest/developerguide/amazon-cognito-user-pools-authentication-flow.html)

현대식 인증 흐름에는 사용자의 자격 증명을 확인하기 위해 암호 외에도 새로운 챌린지 유형이 통합되어 있습니다. 인증은 두 가지 공통 단계로 일반화되며, 이러한 단계는 두 API 작업인 `InitiateAuth` 및 `RespondToAuthChallenge`를 통해 구현됩니다.

인증이 실패하거나 사용자에게 토큰이 발행될 때까지 사용자는 연속 챌린지에 응답하여 인증합니다. Amazon에서는 서로 다른 챌린지를 포함하기 위해 반복 실행되는 이러한 두 단계를 통해 사용자 지정 인증 흐름을 지원할 수 있습니다.

![UserPool_Flow](https://docs.aws.amazon.com/ko_kr/ko_kr/cognito/latest/developerguide/images/cognito-user-pool-auth-flow-srp.png)

> 로그인 시도의 실패는 5회까지 허용되며 그 후의 각 실패에 대해서는 1초부터 시작하여 약 15분까지 두 배씩 기하급수적으로 증가한 시간만큼 임시 잠금 상태가 됩니다. 임시 잠금 시간 동안의 시도는 무시됩니다. 임시 잠금 시간이 지난 후 다음 시도가 실패하면 새 임시 잠금 시간은 마지막 시간의 두 배로 시작됩니다. 시도하지 않고 15분 정도 기다리면 임시 잠금도 재설정됩니다. 이 동작은 변경될 수 있습니다.

<br>

### 클라이언트측 인증 흐름
[AWS Mobile SDK for Android, AWS Mobile SDK for iOS 또는 JavaScript용 AWS SDK for JavaScript](http://aws.amazon.com/mobile/resources/)를 사용하여 생성된 최종 사용자 클라이언트 측 앱에서 사용자 풀 인증의 작동 방식은 다음과 같습니다.

1. 사용자가 사용자 이름 및 암호를 앱에 입력합니다.
2. 앱이 사용자의 사용자 이름 및 SRP 세부 정보를 사용하여 `InitiateAuth` 작업을 호출합니다. 이 메서드는 인증 파라미터를 반환합니다.

> 앱이 Android, iOS 및 JavaScript SDK에서 Amazon Cognito SRP 지원을 사용하여 SRP 세부 정보를 생성합니다.

3. 앱이 `RespondToAuthChallenge` 작업을 호출합니다. 호출에 성공하면 메서드가 사용자 토큰을 반환하고 인증 흐름이 완료됩니다. 또 다른 챌린지가 필요한 경우 토큰이 반환되지 않습니다. 대신 `RespondToAuthChallenge`를 호출하면 세션이 반환됩니다.
4. `RespondToAuthChallenge`가 세션을 반환하는 경우 이번에는 앱이 세션 및 챌린지 응답(예: MFA 코드)과 함께 `RespondToAuthChallenge`를 다시 호출합니다.

<br />

### 기본 제공 인증 흐름 및 챌린지
Amazon Cognito에는 표준 인증을 위한 AuthFlow 및 ChallengeName 값이 기본으로 제공되어 `Secure Remote Password(SRP)` 프로토콜을 통해 사용자 이름과 암호가 확인됩니다. 이 흐름은 Amazon Cognito용 iOS, Android 및 JavaScript SDK에 내장되어 있습니다. 상위 수준에서는 `USER_SRP_AUTH`에 있는 `AuthFlow` 및 `InitiateAuth` 값과 함께 USERNAME를 SRP_A로 `AuthParameters`에 보내면서 흐름이 시작됩니다. `InitiateAuth` 호출이 성공하면 `PASSWORD_VERIFIER`가 챌린지 파라미터의 ChallengeName 및 SRP_B로 응답에 포함됩니다. 그러면 앱이 `RespondToAuthChallenge` `PASSWORD_VERIFIER` 및 ChallengeName의 필수 파라미터를 사용하여 `ChallengeResponses`를 호출합니다. `RespondToAuthChallenge` 호출이 성공하고 사용자가 로그인하면 토큰이 반환됩니다. 사용자에 대해 MFA(Multi-Factor Authentication)가 활성화된 경우 ChallengeName의 SMS_MFA이 반환되고 앱은 `RespondToAuthChallenge`를 또 호출하여 필요한 코드를 제공할 수 있습니다.

<br><br>

## Cognito - RespondToAuchChallenge의 종류
- `SMS_MFA`: 이 Challenge는 SMS를 통해 전달되는 SMS_MFA_CODE를 제공합니다.
- `SOFTWARE_TOKEN_MFA`: 이 Challenge는 Google Authenticator 같은 OTP 앱을 통해 전달되는 MFA 토큰을 제공합니다.
- `SELECT_MFA_TYPE`: MFA 유형을 선택합니다. 유효한 MFA 옵션은 Text SMS MFA의 경우 `SMS_MFA`, TOTP 소프트웨어 토큰 MFA의 경우 `SOFTWARE_TOKEN_MFA`입니다.
- `MFA_SETUP`: MFA가 필요한 경우, MFA 수단 중 하나 이상을 설정하지 않은 사용자는 Challenge를 요청받게 되며, 사용자가 인증을 계속하려면 MFA 유형 중 하나 이상을 설정해야 합니다.
- `PASSWORD_VERIFIER`: 이 Challenge는 클라이언트 측 SRP 계산 후에 `PASSWORD_CLAIM_SIGNATURE`, `PASSWORD_CLAIM_SECRET_BLOCK` 및 `TIMESTAMP`를 제공하는 것입니다.
- `CUSTOM_CHALLENGE`: 이 Challenge는 사용자 지정 인증 흐름에서 토큰이 발급되기 전에 사용자가 다른 Challenge를 통과해야 한다고 판단된 경우 반환됩니다.
- `DEVICE_SRP_AUTH`: 사용자 풀에서 장치 추적이 활성화되고 이전 Challenge가 통과된 경우 이 Challenge가 반환되어 Amazon Cognito가 이 장치 추적을 시작할 수 있습니다.
- `DEVICE_PASSWORD_VERIFIER`: 이 Challenge는 `PASSWORD_VERIFIER`와 유사하지만 장치 전용입니다.
- `ADMIN_NO_SRP_AUTH`: `USERNAME`과 `PASSWORD`로 직접 인증해야 될 경우 반환됩니다. 이 흐름을 사용하려면 앱 클라이언트가 활성화되어 있어야 합니다.
- `NEW_PASSWORD_REQUIRED`: 최초 로그인에 성공한 후 비밀번호를 변경해야 하는 사용자를 위한 것입니다. 이 Challenge는 `NEW_PASSWORD` 및 기타 필수 속성과 함께 전달되어야 합니다.


## Cognito MFA 설정 이후 유의사항
- 로그인 시도 실패 5회 이상 발생 시 Cognito 임시 잠금 Trigger 발동
- OTP가 등록되어 있는 스마트폰을 분실하였거나, 앱을 재설치하여 OTP를 재등록해야되는 경우  
→ Admin 계정 또는 유저 마이페이지에서 초기화(또는 재설정) 할 수 있는 방안이 필요함