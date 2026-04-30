---
title: AWS Lambda PDF 템플릿 업로드 & 다운로드 기능 기술검토 정리
author: abruption
date: 2024-03-30 14:34:00 +0900
categories: [Programming, AWS]
tags: [Lambda, Serverless, npm]
---

# AWS Lambda PDF 템플릿 업로드 & 다운로드 기능 기술검토 정리

## `pdf-creator-node`와 `html-pdf` 비교
- pdf-creator-node 모듈의 Dependencies가 `html-pdf`임
- pdf-creator-node 모듈의 Dependencies인 `handlebar`는 템플릿을 렌더링 할 수 있도록 지원하는 모듈

## `liquid` 모듈 분석
- Ruby 언어 기반으로 만들어진 모듈이나, Node.js로 Fork 되었음
- JSP에서 사용하는 JSTL처럼 HTML Template를 생성하면 동일하게 동작 함

## `pdf-create-node` 적용
- html-pdf 모듈을 Dependency로 사용하고 있어 동일한 모듈 기반
- 그 외 Dependency인 handlebar는 템플릿 작성을 지원하는 모듈
- 간단한 조건문과 반복문(if, includeZero, unless, each, with, lookup 등) 지원
- Hierarchy 구조에서도 JSTL과 비슷한 문법을 적용하여 사용 가능
  - Object / Array(1차원) 형식 → {{#each Array | Object}} {{/each}}
  - 2차원 이상 Array 형식일 경우 {{#each Array}} {{/each}} 중첩 

### 참고사항
- 이미지 태그를 사용한 외부 이미지 삽입 가능 (Base64 인코딩 필요)
- HTML → PDF 파싱 과정에서 HTML 파일 내의 CSS, JS CDN은 적용되지 않음
 → 내장 스크립트 및 내장 CSS는 적용 가능
- Lambda에서 실행 시, `Error: write EPIPE` 오류 발생  
~~→ Lambda에서 지원하는 다른 언어(Go, Python 등)로 대체할 수 있는지 검토 필요~~

## 이슈사항
- `html-pdf` 모듈 3.0.1 버전 사용 시 Lambda에서 동작하지 않는 문제가 발생  
→ 2.0.1 버전으로 다운그레이드 필요함  

- <img src> 태그 사용 시 이미지 경로가 깨져 정상적으로 표시되지 않는 문제가 발생   
→ 모듈 내 Encoding 과정에서 발생하는 현상으로 Local과 Lamdba Cross Check 필요

- Table 작성 시 Header 부분이 일반 Row와 중첩되어 표시되는 문제 발생  
→ CSS로 break-inside: avoid; 처리하였음에도 발생

- 특수문자 중 ⑯ ~ ⑲번까지 나오지 않던 문제  
→ 1부터 15까지는 거의 모든 한국어 폰트가 지원하지만, 16~20까지는 일부 폰트만 지원하기 떄문에 보이지 않았던 문제로, Lambda 배포시 지원 폰트 추가하여 배포 후 해결 [(참고 링크)](https://wordtips.tistory.com/entry/%EC%9B%8C%EB%93%9C-%EA%B8%B0%ED%98%B8-%EC%9B%90%EB%AC%B8%EC%9E%90-%EC%88%AB%EC%9E%90)

- Lambda를 실행하는 Amazon Linux 2 인스턴스의 폰트 문제로 사용하려는 Font를 직접 넣어줘야 함

## References 
- [html-pdf](https://www.npmjs.com/package/html-pdf)
- [pdf-creator-node](https://www.npmjs.com/package/pdf-creator-node)
- [liquid](https://www.npmjs.com/package/html-pdf)
- [handlebar](https://handlebarsjs.com/guide/)
- [pybars3](https://github.com/wbond/pybars3)