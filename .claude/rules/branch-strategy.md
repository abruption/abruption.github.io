# 브랜치 전략

## 기본 브랜치

| 브랜치 | 용도 |
|--------|------|
| `master` | 프로덕션 배포 브랜치. push 시 GitHub Actions가 자동 빌드 → `gh-pages`에 배포 |
| `gh-pages` | GitHub Actions가 관리하는 빌드 산출물 브랜치. 직접 수정 금지 |

## 작업 브랜치 네이밍

`master`에서 분기하여 작업 브랜치를 생성한다. **직접 `master`에 커밋하지 않는다.**

| 접두사 | 용도 | 예시 |
|--------|------|------|
| `feature/` | 새 기능, 새 포스트, 설정 변경 | `feature/custom-domain-blog`, `feature/krisflyer-mile-strategy` |
| `fix/` | 버그 수정, 오류 정정 | `fix/airasia-airline-code`, `fix/security-hardening` |
| `chore/` | 유지보수, 설정 정리 | `chore/replace-naver-verification-file`, `chore/update-site-title` |
| `docs/` | 문서 전용 변경 | `docs/update-readme` |

## 워크플로우

```
1. git checkout master && git pull origin master
2. git checkout -b {prefix}/{descriptive-name}
3. 작업 → 커밋 (Conventional Commits)
4. git push -u origin {branch-name}
5. gh pr create (PR 템플릿 필수 준수)
6. 리뷰 → 머지 → 자동 배포
```

## 커밋 메시지

Conventional Commits 형식을 따른다:

```
{type}({scope}): {description}

# 예시
feat: add custom domain blog.abruption.top
post: claude -p headless 과금 정책과 PTY+JSONL Watch 채널 재설계 회고
fix(airline): correct AirAsia airline code
docs(travels): align Trips count with immigration records
chore: update site title
```

| type | 용도 |
|------|------|
| `feat` | 새 기능 |
| `post` | 새 블로그 포스트 |
| `fix` | 버그 수정 |
| `docs` | 문서 변경 |
| `chore` | 유지보수 |
| `refactor` | 리팩토링 |
