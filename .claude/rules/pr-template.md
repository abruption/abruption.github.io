# PR 생성 규칙

## 자동 Close 방지 (필수)

이 레포에는 `.github/workflows/issue-pr-interceptor.yml`이 설정되어 있어, PR body가 템플릿 패턴에 맞지 않으면 **자동으로 Close**된다.

**검증 패턴** (정규식): `\[x\] Bug|\[x\] New feat|\[x\] Break|\[x\] Doc`

즉, "Type of change" 섹션에서 **최소 1개 항목이 `[x]`로 체크**되어 있어야 한다. 체크하지 않으면 즉시 Close된다.

## PR 템플릿

`.github/PULL_REQUEST_TEMPLATE.md`의 포맷을 **반드시** 따라야 한다. `gh pr create` 시 아래 구조를 body에 포함할 것:

```markdown
## Description

{변경 내용 요약}

## Type of change

- [ ] Bug fix (non-breaking change which fixes an issue)
- [x] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## How has this been tested

- [ ] I have run `bash ./tools/test.sh --build` (at the root of the project) locally and passed
- [x] I have tested this feature in the browser

### Test Configuration

- Browser type & version: {브라우저}
- Operating system: {OS}
- Bundler version: {N/A if not applicable}
- Ruby version: {N/A if not applicable}
- Jekyll version: {N/A if not applicable}

### Checklist

- [x] My code follows the [Google style guidelines](https://google.github.io/styleguide/)
- [x] I have performed a self-review of my own code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [x] I have made corresponding changes to the documentation
- [x] My changes generate no new warnings
- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] Any dependent changes have been merged and published in downstream modules
```

## Type of change 선택 기준

| 변경 유형 | 체크 항목 |
|---------|---------|
| 버그 수정, 오류 정정 | `Bug fix` |
| 새 포스트, 새 기능, 설정 추가 | `New feature` |
| 기존 기능이 깨지는 변경 | `Breaking change` |
| README, 문서만 변경 | `Documentation update` |

## gh pr create 사용 시

```bash
gh pr create \
  --title "{type}: {description}" \
  --body "$(cat <<'EOF'
{위 템플릿 전체를 여기에 포함}
EOF
)" \
  --base master
```

- `--base master` 필수 (배포 브랜치)
- title은 커밋 메시지와 동일한 Conventional Commits 형식
- body는 반드시 HEREDOC으로 전달하여 포맷 보존
