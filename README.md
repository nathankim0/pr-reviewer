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
  <a href="#사용-예시">사용 예시</a> &nbsp;&bull;&nbsp;
  <a href="#라이선스">라이선스</a>
</p>

---

## 설치

> **사전 요구사항**: [GitHub CLI (`gh`)](https://cli.github.com/) 설치 및 `gh auth login` 인증 완료

```
/install-plugin https://github.com/nathankim0/pr-reviewer
```

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
| **Bug** | 버그 또는 로직 오류 |
| **Warning** | 성능 / 보안 이슈 |
| **Minor** | 코드 품질 개선 |
| **Nit** | 스타일 / 사소한 개선 |

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
| **code_change** | 코드 변경 필요 | 코드 수정 + 답글 |
| **question** | 질문 답변 필요 | 답글 |
| **already_done** | 이미 반영됨 | 답글 |
| **disagree** | 동의하지 않음 | 답글 (근거 제시) |
| **unclear** | 의도 불명확 | 사용자에게 확인 |

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
2. 리뷰 관점 선택
3. 워크트리 생성 + 기존 리뷰 확인
4. 병렬 코드 리뷰 (3 에이전트)
5. 결과 정리 및 라인 검증
6. 사용자 승인 + 리뷰 액션 선택
7. GitHub 리뷰 게시
8. 정리

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

## 사용 예시

### review-pr

```
> /pr-reviewer:review-pr https://github.com/owner/repo/pull/42
```

**PR 정보 수집**

```
PR: fix: 로그인 세션 만료 시 리다이렉트 누락 수정
작성자: dev-kim
브랜치: fix/session-redirect → main
변경: 5 파일, +74/-31
```

**기존 리뷰 확인 및 resolve**

이전 리뷰 스레드를 GraphQL API로 조회하여 반영 여부를 확인합니다. 반영 완료된 스레드는 자동 resolve 처리합니다.

```
| # | 파일               | 이전 코멘트 (요약)         | 상태                     |
|---|--------------------|----------------------------|--------------------------|
| 1 | AuthProvider.tsx    | useEffect 의존성 누락      | Resolved (이전)          |
| 2 | SessionGuard.tsx    | 에러 바운더리 미처리       | Resolved (방금) - 반영됨 |
| 3 | api/auth.ts         | 토큰 갱신 경쟁 조건       | 미반영                   |
```

**워크트리 생성 + 병렬 코드 리뷰**

3개 에이전트가 병렬로 코드를 분석합니다.

```
Agent A: 버그/로직 오류 탐지
Agent B: 성능/보안 이슈 탐지
Agent C: 코드 품질/스타일 탐지
```

**결과 정리**

```
총 3개 이슈 발견: Bug 1 · Warning 1 · Minor 0 · Nit 1

Bug — api/auth.ts:45
  refreshToken 호출이 동시에 여러 번 실행될 수 있음. 중복 요청 방지 필요.

Warning — SessionGuard.tsx:28
  만료된 토큰을 localStorage에 평문 저장 중. httpOnly 쿠키 사용 권장.

Nit — AuthProvider.tsx:63
  SESSION_TIMEOUT 매직 넘버. 상수로 분리하면 의도가 명확해짐.
```

**사용자 승인 + GitHub 게시**

코멘트 게시 여부와 리뷰 액션(Approve / Request Changes / Comment)을 선택하면 GitHub에 인라인 코멘트가 게시됩니다.

```
PR #42 리뷰 완료!
- 리뷰 액션: Changes Requested
- 기존 코멘트 3개 중 1개 resolved
- 새 코멘트 3개 게시
```

### resolve-reviews

```
> /pr-reviewer:resolve-reviews 42
```

**PR 정보 확인**

```
PR #42: fix: 로그인 세션 만료 시 리다이렉트 누락 수정
작성자: dev-kim
브랜치: fix/session-redirect
```

**미해결 리뷰 코멘트 수집**

```
미해결 리뷰 코멘트: 2개

| # | 파일             | 라인 | 작성자     | 내용 (요약)                    |
|---|------------------|------|------------|--------------------------------|
| 1 | api/auth.ts      | 45   | dev-park   | refreshToken 중복 요청 방지    |
| 2 | AuthProvider.tsx  | 63   | dev-park   | SESSION_TIMEOUT 상수화         |
```

**코멘트 분석 및 분류**

```
| # | 분류        | 파일                 | 계획                              |
|---|-------------|----------------------|-----------------------------------|
| 1 | code_change | api/auth.ts:45       | mutex 패턴으로 중복 요청 방지     |
| 2 | code_change | AuthProvider.tsx:63   | SESSION_TIMEOUT 상수 분리         |
```

**코드 수정 + 답글 게시**

사용자 승인 후 코드를 수정하고 GitHub에 답글을 게시합니다.

```
처리 완료
- 코드 변경: 2건
- 답글 게시: 2건 (성공 2 / 실패 0)
```

---

## 라이선스

[MIT](./LICENSE)
