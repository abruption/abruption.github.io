---
title: [AWS Lambda] PDF 템플릿 업로드 & 다운로드 기능을 위한 라이브러리 기술검토 정리
author: abruption
date: 2022-01-06 10:24:00 +0900
categories: [Programming, AWS]
tags: [Lambda, Serverless, npm]
---

# [AWS Lambda] PDF 템플릿 업로드 & 다운로드 기능을 위한 라이브러리 기술검토 

## `pdf2json` 모듈을 사용한 테스트 진행 
- PDF를 바이너리에서 JSON 형식으로 구문 분석하고 변환하는 모듈로 pdf.js 기반으로 작성
- 타 모듈에 비해 폭넓은 정보를 추출하여 제공함 (Page 요소, 스타일, Formatter 유형 등)
- 모듈 내에서 encodeURIComponent 처리를 하고 있으므로 특수문자가 원문 그대로가 아닌 Encoding 되어 내려옴 (Case 특정 불가)  
→ Parsing하여 반환된 JSON 파일 내에 Text Align에 대한 정보가 내려오고는 있으나, 모듈 소스 내에서는 Left로 Fix 되어있음 (좌측, 우측, 가운데 정렬 무시)
- pdf2json의 가장 최신버전(2.0.0)의 경우 Node 14 버전 이상을 요구
  - 가장 최신버전에서도 Text Align 정보 `left`로 고정되어 내려옴
- pdf2json 모듈을 설치하여 테스트 할 경우 `unexpected token '(' ` 오류가 해당 모듈에서 발생  
→ 1.2.5 버전으로 재설치 할 경우 해결 [(참고링크)](https://issueexplorer.com/issue/modesty/pdf2json/250)  

## `pdf-parse` (PDF 정보 추출 모듈)
- 해당 파일의 메타데이터 (PDF Format Version, Title, 작성 프로그램, 작성 OS, 생성&수정일자) 출력 지원
- 해당 PDF 파일의 텍스트 출력 기능 제공 (좌, 우, 가운데 정렬 기능 미제공)  
→ PDF 파일 내의 텍스트 정보 출력은 가능하나 색상과 정렬 정보는 내려오지 않음

## `pdffiller` (PDF 정보 추출 및 변환 모듈)
- PDF 템플릿에 대한 값 설정, 템플릿 초기화, 변환 등의 기능 제공
→ Fillable PDF로 설정되어 있어야만 입력 폼에 해당 값 입력가능함

## `form-pdf2json` (PDF ↔ JSON 모듈)
- 데이터 Convert 기능 (PDF ↔ JSON), 파일 Export 기능 (PDF ↔ JSON) 제공  
→ 데이터 Convert와 파일 Export 기능 모두 작동 X (macOS, Windows 테스트 동일)

## `pdfreader` (PDF 정보 추출 모듈)
- 해당 파일의 메타데이터와 Text 정보(X, Y 좌표와 색상 정보) 등을 제공  
→ 해당 모듈과 `pdf` 모듈을 사용하여 X, Y 좌표와 텍스트 정보를 매개변수로 넘겨 PDF 파일 생성하는 방식 구현 실패

## `pdf-creator-node` (HTML → PDF 전환 모듈)
- HTML 템플릿 파일을 작성해놓거나, 업로드한 뒤 데이터 값을 매개변수로 전달하여 PDF 파일 생성  
→ macOS에서는 테스트 실패, Windows에서 테스트 성공
- HTML ↔ PDF의 양방향 전환이 가능한 Library 검색과 테스트 필요  
→ PDF에서 HTML으로 변환하는 [모듈](#pdf2html-pdf--html-전환-모듈)이 현재 JRE가 필요하므로 Lambda에서 사용 불가함

## `pdf2html` (PDF → HTML 전환 모듈)
- Apache Tika를 사용하는 모듈로 Java가 설치되어 있어야 Dependency 모듈 설치 가능    
→ Node.js 기반의 Lambda에서는 사용불가  
- Text Align, Color 값과 상관없이 HTML 파일 생성되어

## `Puppeteer` (Chromium DevTools 기반의 Utility, HTML → PDF 모듈)
- PDF 형식의 스크린샷 저장 기능, Crawling 기능, Automation 기능, Chrome Extension 테스트 기능 등 제공   
→ 이미 `pdf-creator-node` 모듈로 구현 가능한 기능이므로 참고용 기재

## `pdfminer` (PDF → HTML 전환 모듈, Python 기반)
> pdf2txt.py -o output.html -t html -c utf-8 -Y exact -S ./a.pdf
- PDF를 다른 형식(HTML/XML)으로 변환 가능  
→ 테스트 결과 정상적인 레이아웃이 적용된 HTML 추출 가능 (색상 미적용)

## `pdfkit-table` (PDF Table 생성 모듈)
- 요청 파라미터를 기준으로 Table 생성이 가능한 모듈로써 Text Align, Style, Property 등 지정이 가능함  
→ Title, Subtitle은 Text Align 속성 지정 불가(단, 줄바꿈 가능), 나머지 Table 내의 상세 속성들은 모두 지정 가능  
→ 해당 파일의 내용이 많아질 경우 자동으로 PDF 페이지 증가, GroupTable은 표시 불가    

<br>

## References 
- [pdf-parse](https://www.npmjs.com/package/pdf-parse)
- [pdf2json](https://www.npmjs.com/package/pdf2json)
- [pdffiller](https://github.com/pdffillerjs/pdffiller)
- [form-pdf2json](https://github.com/ClareKang/form-pdf2json)
- [pdfreader](https://www.npmjs.com/package/pdfreader)
- [pdf](https://www.npmjs.com/package/pdf)
- [pdf-creator-node](https://www.npmjs.com/package/pdf-creator-node)
- [pdf2html](https://www.npmjs.com/package/pdf2html)
- [Puppeteer](https://www.npmjs.com/package/puppeteer)
- [pdfminer](https://pypi.org/project/pdfminer/)
- [pdfkit-table](https://www.npmjs.com/package/pdfkit-table)