---
title: [AWS Lambda] AWS S3 Event Notifications 기능을 이용한 이미지 리사이징 Lambda 개발 검토 정리
author: abruption
date: 2021-11-30 10:41:00 +0900
categories: [Programming, AWS]
tags: [Lambda, Serverless, Node.js, npm, Library, S3]
---

# S3 Event Notifications을 트리거로 사용하는 ImageResize Lambda [(프로세스 다이어그램)](https://drive.google.com/file/d/1o3O1WqjwSj5z_5ddm-FnVNbDxn9pk3Sm/view?usp=sharing)

### 이슈사항
- 테넌트가 증가하고, 각 환경이 늘어날 때마다 버킷에 이벤트 알림을 각각 추가해야 함 [(AWS 콘솔 및 AWS SDK putBucketNotificationConfiguration로 추가가능)](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html)
- 사용자 정의 메타데이터의 최대 크기는 2KB이며, PUT 요청 헤더의 크기는 8KB 이하여야 함
- 사용자 정의 메타데이터의 Key는 대소문자 상관없이 무조건 소문자로 저장됨
- 동일 객체의 사용자 정의 메타데이터의 이름은 중복될 수 없음 (403 Forbidden 오류 발생)
- 사용자 정의 메타데이터의 Value 값은 String 형식만 가능함
- 객체 metadata는 객체가 업로드 된 후에는 수정할 수 없으며, 복사 후에 수정해야 함
- S3 이벤트 알림은 버킷 당 100개로 제한됨
- Prefix에 대한 와일드카드 설정 불가 (Ex. `**/temp`)
- Suffix에 파일 확장자 다중 지정 불가 (S3 이벤트 알림 각각 생성해야 함)

### 문제사항
- 이미지 삭제도 신규 저장과 동일한 S3 이벤트 알림으로 설정 및 구현하려 하였으나 버킷의 각 환경별, 테넌트별 폴더별로 이벤트 알림을 설정해야 하므로 사실상 불가능   
- 메타데이터 값을 가지고 있는 원본 데이터를 copyObject로 복사하였을 경우 메타 정보가 삭제됨   
- S3 이벤트 알림의 경우 1번 이상 전송되도록 설계되었으므로 Lambda가 동일 이미지에 대해 1회 이상 실행될 수 있음 [(Amazon S3 이벤트 알림)](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/NotificationHowTo.html)
- S3의 이벤트 알림을 받아 이미지 리사이징이 비동기적으로 처리되는 중에 SaveData Lambda가 먼저 완료될 경우 썸네일 이미지를 가져올 때 404 오류가 발생할 수 있음  
- 현재 이미지 업로드 시 회전 값을 Resize Lambda에 넘겨 처리하도록 되어있는데 S3 Event를 사용할 경우 Lambda에 직접적으로 값을 전달할 수 없음  
    → S3 버킷에 이미지를 업로드할 경우 사용자 정의 메타데이터에 회전 값을 설정하여 넘겨주고, Lambda에서 해당 객체의 메타정보를 읽어서 처리하는 로직으로 처리

- S3 이벤트 알림이 Lambda에게 2회 전송되는 문제 발생  
    →  테스트 Lambda 소스코드 상의 문제로 원본 이미지의 회전 후 저장 경로 변경으로 해결


### References
- [S3 객체 메타데이터 작업](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/UsingMetadata.html)
- [Amazon S3 콘솔에서 객체 메타데이터 편집](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/add-object-metadata.html)
- [PutObject](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/API/API_PutObject.html)
- [POST Object](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/API/RESTObjectPOST.html)

### 사용자 정의 메타데이터 사용 사례
- [Example: Browser-Based Upload using HTTP POST (Using AWS Signature Version 4)](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/API/sigv4-post-example.html)
- [업로드 예제(AWS 서명 버전 2)](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/HTTPPOSTExamples.html)
- [Browser Based Uploads to Amazon S3?](https://stackoverflow.com/questions/1236688/browser-based-uploads-to-amazon-s3)
- [Amazon AWS S3 browser-based upload using POST](https://pretagteam.com/question/amazon-aws-s3-browserbased-upload-using-post)