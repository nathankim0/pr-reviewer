# CLAUDE.md

**중요: 반드시 한글로 대답하시오.**

## 플러그인 수정 후 배포 절차

파일 수정 시 반드시 아래 절차를 **자동으로** 수행할 것:

1. `git add` → `git commit` → `git push origin main`
2. `claude plugin marketplace update pr-reviewer`
3. `claude plugin uninstall pr-reviewer@pr-reviewer` → `claude plugin install pr-reviewer@pr-reviewer`

> 버전이 동일하면 `update`가 동작하지 않으므로 반드시 `uninstall` → `install`로 재설치한다.
