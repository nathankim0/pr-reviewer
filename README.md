# PR Reviewer - Claude Code Plugin

자동으로 GitHub PR을 리뷰하고 인라인 코멘트를 게시하는 Claude Code 플러그인입니다.

## 기능

- PR diff를 분석하여 버그, 성능/보안 이슈, 코드 품질 문제를 탐지
- 병렬 에이전트로 빠른 리뷰 수행
- 심각도별 분류 (Bug, Warning, Minor, Nit)
- 사용자 승인 후 GitHub PR에 인라인 코멘트 자동 게시

## 설치

Claude Code에서 다음 명령어를 실행하세요:

```
/install-plugin https://github.com/nathankim0/pr-reviewer
```

## 사전 요구사항

- [GitHub CLI (`gh`)](https://cli.github.com/) 설치 및 인증
- `gh auth login`으로 GitHub 계정 인증 완료

## 사용법

```
/pr-reviewer:review-pr <PR_URL>
```

**예시:**

```
/pr-reviewer:review-pr https://github.com/owner/repo/pull/123
```

## 리뷰 워크플로우

1. **PR 정보 수집** - PR 메타데이터, 변경 파일, diff 추출
2. **워크트리 생성** - PR 브랜치를 로컬에 체크아웃
3. **병렬 코드 리뷰** - 3개 에이전트가 동시에 분석
   - Agent A: 버그/로직 오류
   - Agent B: 성능/보안 이슈
   - Agent C: 코드 품질/스타일
4. **결과 정리** - 심각도별 분류하여 표시
5. **사용자 승인** - 게시할 코멘트 선택
6. **인라인 코멘트 게시** - GitHub PR에 리뷰 코멘트 생성
7. **정리** - 임시 워크트리 제거

## 심각도 분류

| 심각도 | 아이콘 | 설명 |
|--------|--------|------|
| Bug | 🐛 | 버그 또는 로직 오류 |
| Warning | ⚠️ | 성능/보안 이슈 |
| Minor | 📝 | 코드 품질 개선 |
| Nit | 💅 | 스타일/사소한 개선 |

## 라이선스

MIT License
