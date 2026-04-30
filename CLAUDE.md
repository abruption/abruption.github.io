# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

Jekyll 기반 개인 기술 블로그 ([abruption.github.io](https://abruption.github.io)). Chirpy 테마를 사용하며, GitHub Actions를 통해 `master` 브랜치 push 시 자동으로 빌드 및 `gh-pages` 브랜치에 배포된다.

## 주요 명령어

```bash
# 의존성 설치 (최초 1회)
bundle install

# 로컬 개발 서버 실행 (http://localhost:4000)
bundle exec jekyll s

# 프로덕션 빌드
JEKYLL_ENV=production bundle exec jekyll b
```

## 아키텍처

- **`_config.yml`** — 사이트 전역 설정 (url, author, timezone, Disqus, Google Analytics 등)
- **`_posts/`** — 블로그 포스트 (파일명 규칙: `YYYY-MM-DD-제목.md`)
- **`_tabs/`** — 사이드바 탭 페이지 (About, Archives, Categories, Tags)
- **`_data/`** — 사이트 전역 데이터 파일 (label, contact, share, date_format 등)
- **`assets/`** — 정적 자산 (이미지, CSS, JS)
- **`.github/workflows/pages-deploy.yml`** — GitHub Actions CI/CD 파이프라인

## 포스트 작성 규칙

포스트 파일명: `_posts/YYYY-MM-DD-제목.md`

Front matter 필수 항목:
```yaml
---
title: 포스트 제목
author: abruption
date: YYYY-MM-DD HH:MM:SS +0900
categories: [상위카테고리, 하위카테고리]
tags: [태그1, 태그2]
---
```

- `categories`는 최대 2단계 계층 구조
- `tags`는 소문자 권장
- 이미지는 `assets/img/` 하위에 배치

## 배포

`master` 브랜치에 push하면 GitHub Actions가 자동으로 빌드 후 `gh-pages` 브랜치에 배포한다. 별도의 수동 배포 작업은 필요 없다.
