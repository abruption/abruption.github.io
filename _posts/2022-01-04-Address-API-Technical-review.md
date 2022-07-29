---
title: [Open API] 도로명주소 API 관련 정보 정리 (2022.01.04 기준)
author: abruption
date: 2022-01-04 17:16:00 +0900
categories: [Programming, OpenAPI]
tags: [Kakao, Daum, OpenAPI]
---

# [Open API] 도로명주소 API 관련 정보 정리 (2022.01.04 기준)

## 주소검색솔루션
- 프로그램 설치 방식으로 Windows와 Linux OS 지원
- 내장 스케줄러 지정으로 최신 주소 자동으로 갱신
- 도로명주소(영문, 국문), 사서함주소, 우편번호 검색 등의 다양한 주소검색방식 지원
- 도로명주소 자동완성, 검색화면 색상변경, 주서검색창 타이틀 및 로고 변경 등의 부가적인 편의기능 제공

## Open API
1. 서비스 용도
- 운영: 본인인증 필요 O, 승인키는 만료기간없이 사용 가능
- 개발: 본인인증 필요X, 승인키는 선택한 기간(7일, 30일, 9일)동안 사용 가능

<br />

2. Open API 제약조건
- ~~일일 요청건수 제한은 없으나 5초에 10건의 제한 조건이 존재~~   
→ [도로명주소 API의 호출건수 제한 X](https://www.juso.go.kr/addrlink/DevCenterQABoardDetail.do?mgtSn=96602&currentPage=1&searchType=&keyword=)
- 차단이 되었을 경우 국가정보자원관리원에 유선연결하여 외부 IP의 차단여부 확인 후 차단해제 요청
- API 호출 시 검색어에 특수문자나 SQL문(SELECT, OR, DELETE 등)이 포함되어 SQL Injection 공격으로 간주해 차단이 될 수 있으므로 검색어에 특수문자 등을 필터링할 수 있도록 권고
- 팝업API와 검색API는 승인키를 같이 사용할 수 없으며 각각 신청하여 사용해야 함
- URL, IP가 다수인 경우 '활용시스템 리스트' 양식을 다운로드하여 다수의 URL, IP를 기재 후 첨부하면 각각 URL, IP별로 승인 키를 발행함
- 승인 키는 URL(또는 IP) 기준으로 부여되므로 다수의 사이트(또는 Application)에서 사용할 경우 각각 부여된 승인키를 사용해야 함


## 카카오 우편번호 서비스 API
- Access Key 발급 X, 사용량 제한 X
- 행정안전부에서 제공하는 도로명 주소 DB를 직접 업데이트 받아 사용중이므로 도로명주소 API와 동일
- CSS 및 API 스크립트 임의 수정 사용 불가


## References
- [도로명주소 안내시스템 개발자센터](https://www.juso.go.kr/addrlink/main.do?cPath=99MM)
- [도로명주소 검색 솔루션](https://www.juso.go.kr/addrlink/jusoSearchSolutionIntroduce.do)
- [도로명주소 Open API](https://www.juso.go.kr/addrlink/devAddrLinkRequestWrite.do?returnFn=write&cntcMenu=URL)
- [[오픈API]좌표제공API 일일 건수 제한 문의입니다.](https://www.juso.go.kr/addrlink/DevCenterQABoardDetail.do?mgtSn=96307&currentPage=1&searchType=subject&keyword=%EC%9D%BC%EC%9D%BC)
- [[오픈API]일일 최대 사용량 문의드립니다.](https://www.juso.go.kr/addrlink/DevCenterQABoardDetail.do?mgtSn=90436&currentPage=1&searchType=subject&keyword=%EC%9D%BC%EC%9D%BC)
- [오픈API 호출 제한수가 있나요?](https://www.juso.go.kr/addrlink/DevCenterQABoardDetail.do?mgtSn=37180&currentPage=1&searchType=subject&keyword=%EC%A0%9C%ED%95%9C)
- [[오픈API]도로명 API PHP파일 개인 블로그 및 GITHUB 등록 문의](https://www.juso.go.kr/addrlink/DevCenterQABoardDetail.do?mgtSn=88745&currentPage=1&searchType=subject&keyword=%EA%B0%9C%EC%9D%B8)