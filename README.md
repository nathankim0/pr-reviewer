<p align="center">
  <h1 align="center">PR Reviewer</h1>
  <p align="center">
    <strong>Claude Code Plugin for GitHub PR Review</strong>
  </p>
  <p align="center">
    PR을 리뷰하고 &middot; 리뷰에 응답하고 &middot; 코드를 수정합니다
  </p>
</p>

<p align="center">
  <a href="#설치">설치</a> &nbsp;&bull;&nbsp;
  <a href="#커맨드">커맨드</a> &nbsp;&bull;&nbsp;
  <a href="#워크플로우">워크플로우</a> &nbsp;&bull;&nbsp;
  <a href="#라이선스">라이선스</a>
</p>

---

## 커맨드

### `review-pr` — PR 코드 리뷰

> PR diff를 분석하여 이슈를 탐지하고, GitHub에 인라인 코멘트를 게시합니다.

```
/pr-reviewer:review-pr <PR_URL>
```

```bash
# 예시
/pr-reviewer:review-pr https://github.com/owner/repo/pull/123
```

| 심각도 | 설명 |
|--------|------|
| 🐛 **Bug** | 버그 또는 로직 오류 |
| ⚠️ **Warning** | 성능 / 보안 이슈 |
| 📝 **Minor** | 코드 품질 개선 |
| 💅 **Nit** | 스타일 / 사소한 개선 |

### `resolve-reviews` — 리뷰 코멘트 해결

> PR에 달린 미해결 리뷰 코멘트를 분석하고, 코드를 수정하고, GitHub에 답글을 게시합니다.

```
/pr-reviewer:resolve-reviews [PR_URL | PR_번호]
```

```bash
# 예시
/pr-reviewer:resolve-reviews https://github.com/owner/repo/pull/123
/pr-reviewer:resolve-reviews 123
/pr-reviewer:resolve-reviews          # 현재 브랜치의 PR 자동 감지
```

| 분류 | 설명 | 액션 |
|------|------|------|
| 🔧 **code_change** | 코드 변경 필요 | 코드 수정 + 답글 |
| 💬 **question** | 질문 답변 필요 | 답글 |
| ✅ **already_done** | 이미 반영됨 | 답글 |
| 🤔 **disagree** | 동의하지 않음 | 답글 (근거 제시) |
| ❓ **unclear** | 의도 불명확 | 사용자에게 확인 |

---

## 워크플로우

<table>
<tr>
<th width="50%">review-pr</th>
<th width="50%">resolve-reviews</th>
</tr>
<tr>
<td>

1. PR 정보 수집
2. 워크트리 생성
3. 병렬 코드 리뷰 (3 에이전트)
4. 결과 정리 및 표시
5. 사용자 승인
6. GitHub 인라인 코멘트 게시
7. 정리

</td>
<td>

1. 입력 파싱 및 환경 검증
2. 미해결 리뷰 코멘트 수집
3. 코멘트 분석 및 분류
4. 사용자 1차 승인
5. 파일별 코드 변경
6. 변경 사항 요약 및 2차 승인
7. GitHub 답글 게시

</td>
</tr>
</table>

---

## 설치

> **사전 요구사항**: [GitHub CLI (`gh`)](https://cli.github.com/) 설치 및 `gh auth login` 인증 완료

```
/install-plugin https://github.com/nathankim0/pr-reviewer
```

---

## 라이선스

[MIT](./LICENSE)
