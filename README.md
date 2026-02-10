# PR Reviewer - Claude Code Plugin

GitHub PR을 자동으로 리뷰하고, 리뷰 코멘트에 응답하는 Claude Code 플러그인입니다.

## 기능

### review-pr - PR 코드 리뷰
- PR diff를 분석하여 버그, 성능/보안 이슈, 코드 품질 문제를 탐지
- 병렬 에이전트로 빠른 리뷰 수행
- 심각도별 분류 (Bug, Warning, Minor, Nit)
- 사용자 승인 후 GitHub PR에 인라인 코멘트 자동 게시

### resolve-reviews - 리뷰 코멘트 해결
- PR에 달린 미해결 리뷰 코멘트를 자동으로 수집 및 분석
- 코멘트 유형별 분류 (코드 변경, 질문, 이미 해결, 동의하지 않음, 불명확)
- GitHub suggestion 블록 자동 감지 및 적용
- 코드 수정 후 GitHub에 답글 자동 게시

## 설치

Claude Code에서 다음 명령어를 실행하세요:

```
/install-plugin https://github.com/nathankim0/pr-reviewer
```

## 사전 요구사항

- [GitHub CLI (`gh`)](https://cli.github.com/) 설치 및 인증
- `gh auth login`으로 GitHub 계정 인증 완료

## 사용법

### PR 코드 리뷰

```
/pr-reviewer:review-pr <PR_URL>
```

**예시:**

```
/pr-reviewer:review-pr https://github.com/owner/repo/pull/123
```

### 리뷰 코멘트 해결

```
/pr-reviewer:resolve-reviews [PR_URL 또는 PR_번호]
```

**예시:**

```
/pr-reviewer:resolve-reviews https://github.com/owner/repo/pull/123
/pr-reviewer:resolve-reviews 123
/pr-reviewer:resolve-reviews                    # 현재 브랜치의 PR 자동 감지
```

## review-pr 워크플로우

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

## resolve-reviews 워크플로우

1. **입력 파싱 및 환경 검증** - PR 식별, 브랜치 확인
2. **리뷰 코멘트 수집** - GraphQL로 미해결 스레드 조회
3. **코멘트 분석 및 분류** - 5가지 카테고리로 분류
4. **사용자 1차 승인** - 처리 방식 선택
5. **파일별 코드 변경** - suggestion 적용, 코드 수정
6. **변경 사항 요약 및 2차 승인** - 답글 미리보기 확인
7. **GitHub 답글 게시** - 답글 게시 및 결과 보고

## 코멘트 분류 기준

### review-pr 심각도

| 심각도 | 아이콘 | 설명 |
|--------|--------|------|
| Bug | 🐛 | 버그 또는 로직 오류 |
| Warning | ⚠️ | 성능/보안 이슈 |
| Minor | 📝 | 코드 품질 개선 |
| Nit | 💅 | 스타일/사소한 개선 |

### resolve-reviews 카테고리

| 카테고리 | 아이콘 | 설명 |
|----------|--------|------|
| code_change | 🔧 | 코드 변경 필요 |
| question | 💬 | 질문 답변 필요 |
| already_done | ✅ | 이미 반영됨 |
| disagree | 🤔 | 동의하지 않음 |
| unclear | ❓ | 의도 불명확 |

## 라이선스

MIT License
