---
title: Google Analytics 4 (GA4) 기술검토 분석 및 정리
author: abruption
date: 2022-02-09 17:45:00 +0900
categories: [Programming, Google Analytics]
tags: [Analytics, Google, DataStudio]
---

# Google Analytics 4 (GA4) 기술검토 분석 및 정리

## Google Analytics 버전
- 기존 Page 데이터 중심으로 구성되어 있는 UA(Universal Analytics)와 Event 데이터 중심으로 구성되어 있는 GA4(Google Analytics 4)가 존재
- 기존 UA와 다르게 GA4는 Web과 App 모두 추적하기 위해 Page보다는 Event/Segment 중심으로 데이터를 수집 
- App과 Web의 데이터를 통합해서 보고 쉽게 관리하려는 목적으로는 GA4가 적합  
→ UA를 사용할 경우 App의 데이터는 Firebase를 통해 데이터를 확인하고 별도의 서비스를 통해 데이터를 통합하는 작업이 필요

<br>

## Google Analytics API 종류
- Data API (Preview): GA4 보고서 데이터에 프로그래밍 방식으로 액세스 할 수 있는 API (Ex. 지난 주 내 Android 앱의 일일 활성 사용자는 몇 명인지에 대한 보고서 생성)
- Admin API (Preview): 새 계정 프로비저닝, 계정 관리, GA4 속성 관리 등을 제어할 수 있는 API
- Core Reporting API: 복잡한 보고 작업을 자동화, GA 데이터를 다른 비즈니스 애플리케이션과 통합 등의 기능을 제공하는 API
- Management API: 사용자의 모든 계정, 속성과 권한 관리 등의 기능을 제공하는 API
- Embed API: 타 웹사이트에서 대시보드를 쉽게 만들고 포함할 수 있는 JavaScript Library
- User Deletion API: 고객이 지정한 사용자 식별자와 연결된 데이터 삭제를 처리할 수 있는 API
- Multi-Channel Funnels Reporting API: 인증된 사용자에 대한 다중 채널 유입경로 데이터를 요청할 수 있는 API
- Real Time Reporting API: 인증된 사용자에 대한 실시간 데이터를 요청할 수 있는 API
- Metadata API: Google Analytics Report API에 노출된 열의 목록과 속성을 반환하는 API입니다. 반환되는 속성에는 UI 이름, 설명, 세그먼트 지원 등이 포함됩니다.

<br>

### Google API Console 상 분류
1. Google Analytics Reporting API
   - Google Analytics Reporting API v4
     - GetReports
     - SearchUserActivity
     - GetDiscovery
     - GetDiscoveryRest

   - Google Analytics Data API
     - BatchGetNextCompatibleFields
     - BatchGetReports
     - DescribeMetaData
     - StreamBatchGetReports
     - RunFunnelReport
     - BatchRunPivotReports
     - BatchRunReports
     - CheckCompatbility
     - GetMetadata
     - RunPivotReport
     - RunRealtimeReport
     - RunReport


2. Google Analytics API
   - Management API v3
   - Core Reporting API v3
   - Real Time Reporting API v3
   - User Deletion API v3


<br><br>

## Quota and Limit
### API Key 발급 (Google Cloud Platform)
- API Key는 기본적으로 (호출) 무제한 상태로 발급 처리되며 해당 API Key로 Analytics API 호출
→ API 키를 사용할 수 있는 웹사이트, IP주소 또는 앱을 지정하는 것을 권고 (API 키별로 제한사항 유형 1개만 설정 가능)
- 애플리케이션 제한사항은 HTTP 리퍼러, IP주소, Android 앱, iOS 앱 중 1가지 선택
    - HTTP 리퍼러를 선택할 경우 웹사이트 제한사항 지정 (미기재 시 모든 웹사이트 요청 수락)
    - 특정 URL 허용 / 단일 하위 도메인의 모든 URL 허용 / 단일 도메인에서 모든 하위 도메인 URL 허용 (와일드카드 사용 가능)

### 일반 할당량 한도
> 다음 할당량은 `Management API`, `Core Reporting API v3`, `Multi-Channel Funnels Reporting API`, `Metadata API`, `User Deletion API`, `Real Time Reporting API`에 적용됩니다.
- 프로젝트별 일일 요청 50,000건 (요청 상향 가능)
- IP 주소 별 초당 쿼리(QPS) 10건

### Reporting API
> 다음 할당량은 `Core Reporting API v3`, `Analytics Reporting API v4`, `Real Time API v3`, `Multi-Channel Funnel API v3`에 적용됩니다.
- 일일 보기(프로필)당 요청 10,000건 (요청 상향 불가)
- 보기(프로필)당 동시 요청 10건 (요청 상향 불가) 

### Reporting API
> 다음 할당량은 `Analytics Reporting API v4`에만 적용됩니다.
- 프로젝트당 일일 요청 수 50,000건  
- 각 프로젝트의 100초 간의 요청 수 2,000건  
- 각 프로젝트의 사용자별 100초 간의 요청 수 100건 (Google API Console에서 1,000건까지 상향 가능)
    → 프로젝트는 GCP Console에서 생성한 프로젝트를 말 함.

- 일일 보기(프로필)당 요청 10,000건 (요청 상향 불가)  
    → 프로젝트는 GCP Console에서 생성한 프로젝트를 말 함.

### 추가 할당량 요청
> 다음 목록만 상향 요청할 수 있습니다.
- 프로젝트별 일일 읽기 요청건 수 50,000건 (기본 값)
- 프로젝트별 일일 쓰기 요청건 수 50건 (기본 값)

## 보고서 작성 관련 Guide [(Google Analytics Data API (GA4) Creating a Report)](https://developers.google.com/analytics/devguides/reporting/data/v1/basics#dimensions)
### Dimension
Dimension은 웹사이트 또는 앱에 대한 이벤트 데이터를 설명하고 그룹화합니다. 예를 들어 `city` Dimension은 각 이벤트가 발생한 도시를 나타냅니다.  
보고서 요청에서 0개 이상의 Dimension을 지정할 수 있으며 요청은 최대 9개 Dimension까지 허용됩니다.

#### Dimension Filter
보고서 요청을 제출할 때 특정 Dimension 값에 대한 데이터만 반환하도록 요청할 수 있습니다. Dimension을 필터링하려면 요청 본문에서 dimensionFilter 필드에 [FilterExpression](https://developers.google.com/analytics/devguides/reporting/data/v1/rest/v1beta/FilterExpression)을 지정하면 됩니다.

### Metrics
Metrics는 웹사이트 또는 앱에 대한 이벤트 데이터에 대한 정량적 측정이며, 보고서 요청에서 하나 이상의 Metrics을 지정할 수 있습니다.

### Pagination
기본적으로 보고서 응답에는 최대 처음 10,000개의 이벤트 데이터 행이 포함됩니다. 10,000~100,000개의 행이 있는 보고서의 경우 RunReportRequest "limit": 10000에 포함하여 최대 100,000개의 행을 검색할 수 있습니다.

100,000개 이상의 행이 있는 보고서의 경우 행을 통해 페이징 요청 시퀀스를 보내야합니다.

### Multiple Date Ranges
하나의 보고서 요청으로 여러 dateRanges에 대한 데이터를 검색할 수 있습니다. 해당 요청에 대한 응답 값은 `date_range_0`, `date_range_1`의 형식입니다z.

## GA4 Query Explorer
Google Analytics 이벤트 데이터의 맞춤 보고서를 작성할 수 있는 사이트로 JSON 형태로 작성하는 것보다 간편하고 직관적으로 설정할 수 있으며,
수집된 데이터에 대한 결과 값을 JSON 형태 또는 Pivot 데이터로 반환받아 확인할 수 있다.  

<br><br>

## References
- [Data API](https://developers.google.com/analytics/devguides/reporting/data/v1)
- [Admin API](https://developers.google.com/analytics/devguides/config/admin/v1)
- [Reporting API v4](https://developers.google.com/analytics/devguides/reporting/core/v4)
- [Management API](https://developers.google.com/analytics/devguides/config/mgmt/v3)
- [Embed API](https://developers.google.com/analytics/devguides/reporting/embed/v1)
- [User Deletion API](https://developers.google.com/analytics/devguides/config/userdeletion/v3)
- [Multi-Channel Funnels Reporting API](https://developers.google.com/analytics/devguides/reporting/mcf/v3)
- [Metadata API](https://developers.google.com/analytics/devguides/reporting/metadata/v3)
- [Real Time Reporting API](https://developers.google.com/analytics/devguides/reporting/realtime/v3)
- [Google Analytics Compare](https://marketingplatform.google.com/about/analytics/compare/)
- [Google Cloud API 키 사용 문서](https://cloud.google.com/docs/authentication/api-keys?hl=ko)
- [Google Analytics 4 Dev-Tool](https://ga-dev-tools.web.app/ga4/dimensions-metrics-explorer/)