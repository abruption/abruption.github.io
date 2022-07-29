---
title: AWS Lambda를 이용한 이미지 리사이징 방법 검토 정리
author: abruption
date: 2021-11-24 17:05:00 +0900
categories: [Programming, AWS]
tags: [Lambda, Serverless, Node.js, npm, Library]
---

# AWS Lambda를 이용한 이미지 리사이징 방법 검토


1. [Lambda@Edge](https://aws.amazon.com/ko/lambda/edge/)
   - S3에 데이터를 저장하는 것은 기존과 동일하나, 이미지 리사이징을 처리하는 방식을 Lambda@Edge로 사용
   - Querystring으로 리사이징 사이즈를 지정할 수 있으므로 사이즈가 Fix된 ResizeLambda 대비 확장성이 높다
   - 다만, 일반 Lambda보다 3배 이상 높은 요금  
   - 미국 리전에서만 생성 가능하다 (단, CloudFront를 통해 전세계 리전에 배포되므로 크게 관계 없음)
   - 기본 24시간 캐싱, 최대 100년까지 캐싱 가능하지만 사용자가 이미지를 수정 및 삭제하면 캐싱을 삭제하는 별도 처리가 필요하다 (1,000개 이후의 무효화 요청은 과금)
   - Lambda@Edge는 VPC에 접근할 수 없어 DB 정보를 조회하고 수정, 삭제하는 로직 처리 불가
   - 사용자가 이미지 파일을 삭제하였을 경우 Cloudfront의 캐시 무효화 처리를 해야하는 문제가 발생한다. (View Request 부분에 Lambda@Edge로 예외처리 가능하나 비용 발생)

2. **S3 Event를 트리거로 사용하는 ImageResize Lambda**
   - S3 이벤트 알림의 Prefix 경로에 와일드카드 문자는 객체 키 이름으로 사용할 수 있으므로 이벤트 알림이 정상동작하지 않는 문제가 발생한다. [(링크)](https://aws.amazon.com/ko/premiumsupport/knowledge-center/s3-event-notification-filter-wildcard/)
   - S3의 이벤트 알림 생성으로 접두사에 폴더명을 지정할 수 있다.
   - 단, 같은 폴더 내에 리사이징된 이미지 파일을 저장할 경우 재귀함수처럼 무한반복되므로 파일명 중복체크 로직 등이 필요하다.
   - 위와 같은 문제를 개선하기 위해 Original 파일과 Resize 파일의 저장 위치는 버킷 또는 폴더가 다르게 저장되도록 해야 한다.
   - 위에서 반환받은 결과 파일이 존재하지 않다면 기존의 Rollback 처리한다.
   - **특정 환경 Bucket에 이벤트 알림을 설정하여 `/temp` 폴더에 이미지가 업로드되면 App 환경에 맞는 디렉토리 구조의 현재 날짜에 thumb 이미지를 저장하는 방식으로 구현 (Tenent 번호는 DB에 접속하여 split하여 입력하도록)**
   - **이미지 업로드 시에 헤더에 `x-amz-meta-rotate`라는 Key 값을 추가하여 회전 정보를 담아 저장한 뒤, Resize 또는 원본 데이터의 HeadObject로 데이터를 확인하여 회원 수정을 처리하도록 구현**

3. S3 Event와 SQS/SNS 서비스를 이용한 ImageResize Lambda
   - S3 이벤트 알림의 Prefix 경로에 와일드카드 문자는 객체 키 이름으로 사용할 수 있으므로 이벤트 알림이 정상동작하지 않는 문제가 발생한다. [(링크)](https://aws.amazon.com/ko/premiumsupport/knowledge-center/s3-event-notification-filter-wildcard/)
   - 2번과 동일한 방법에 동시에 다량의 이미지 저장 요청이 들어오게 될 경우 비동기적 처리하기 위한 새로운 인스턴스가 계속 올라오게되므로 Cold Start 문제가 발생한다.
   - 이와 같은 문제를 해결하기 위해 Queue 방식을 지원하는 Amazon SQS나 Amazon SNS 서비스를 이용하여 FIFO 방식 등으로 순차적으로 제한을 두어 처리하도록 한다.
   - Amazon SQS는 100만건까지 무료이나 SNS는 1백만건 또는 데이터의 GB당 비용을 청구하는 방식으로 SQS의 FIFO 방식으로 사용하는 것이 효율적

4. [ImageProxy](https://github.com/willnorris/imageproxy)
   - Lambda@Edge와 비슷한 원리로 동작하지만 Golang으로 작성된 프로그램을 실행할 서버가 필요
   - 실행방법은 요청 URL에 리사이징 옵션과 원본 이미지 URL을 입력하면 자동으로 리사이징된 이미지가 나타나고, 캐시된 이미지는 Local 또는 S3 등에 저장할 수 있다.
   - 해당 프로그램을 Lambda로 작성할 경우 기존 Resize Lambda와 별 차이가 없으므로 메리트 낮음 → **AWS Fargate를 사용하여 Serverless Computing Engine 구현**
   - 최초 Resizing 실행 시, 3.3MB → 3.87초 소요 / 15MB → 7초 소요 / 23.9MB → 10.45초 소요 (E2.micro 사양 기준)
   - Resizing된 이미지 파일의 크기는 60~70KB 수준이며, Local Storage 또는 S3에 캐시 이미지 저장 가능 (캐시 Hit 시 최소 48ms, 최대 600ms 소요, 기본 캐시시간은 4시간)
   - 원본 이미지 주소에 대해 S3 URI 사용불가, 객체 URL로만 인식 가능

5. [ImgProxy](https://imgproxy.net/)
   - Go 기반으로 작성된 프로그램이며 Docker, Heroku, Linux, Mac 등에서 사용가능
   - Go로 Local 서버를 띄워 요청을 처리하고 보여주는 방식이므로 상시 구동 가능한 인스턴스가 필요 함
   - Docker Image를 빌드하여 사용할 수 있으므로 AWS Fargate 사용도 고려해 볼 수 있음
   - 기존 로직에서 S3에 Upload만 하고, 해당 경로에 대한 Resize 옵션만 URL에 지정하면 설정할 수 있으므로 처리가 간단해짐
   - 단, ImageProxy와 다르게 캐시 이미지를 저장할 수 있는 방식을 지원하지 않으므로 요청이 들어올 때마다 Resize 처리가 일어남
   - 2MB / 10MB 이미지 파일 Resize 시에 `Source image resolution is too big` 오류 발생 (config.go 파일에서 MaxResolution 수정 시 해결 가능)
   - Local 서버 구동 후 일정시간 동안 요청이 없을경우 자동으로 종료됨 (약 2시간 30분)
   - 원본 이미지 주소에 대해 S3 URI 사용 가능
  
6. [AWS Lambda와 DynamoDB를 이용한 작업 대기열 구현](https://aws.amazon.com/ko/blogs/compute/implementing-a-lifo-task-queue-using-aws-lambda-and-amazon-dynamodb/)
   - LIFO 방식을 사용하는 방법이므로 FIFO 방식이 필요한 해당 방식에서는 제외

### 공통사항
- Image Upload 시 DataType은 jpg, jpeg, png, gif만 가능하도록 Validate 처리하고, gif는 리사이징 제외  
- Image Upload Size Limit은 15MB (Studio에서 최대 설정 값)

### 참고 링크
- [S3 Events to Lambda vs S3 Events to SQS/SNS to Lambda](https://aws.plainenglish.io/system-design-s3-events-to-lambda-vs-s3-events-to-sqs-sns-to-lambda-2d41477d1cc9)
- [USING TERRAFORM TO DEPLOY S3->SQS->LAMBDA INTEGRATION](https://hands-on.cloud/using-terraform-to-deploy-s3-sqs-lambda-integration/)
- [How to send a message to an SQS queue using Lambda when a file is uploaded to an S3 bucket](https://awsbytes.com/how-to-send-a-message-to-an-sqs-queue-using-lambda-when-a-file-is-uploaded-to-an-s3-bucket/)
- [Triggering Lambda function from SQS and store the message in S3 bucket](https://play.whizlabs.com/site/task_details?lab_type=1&task_id=54&quest_id=31)
- [cloudfront-image-proxy](https://github.com/skorfmann/cloudfront-image-proxy)
- [Amazon S3 버킷에 특정 파일 유형만 업로드할 수 있도록 하려면 어떻게 해야 합니까?](https://aws.amazon.com/ko/premiumsupport/knowledge-center/s3-allow-certain-file-types/)
- [S3 객체 메타데이터 작업](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/UsingMetadata.html)