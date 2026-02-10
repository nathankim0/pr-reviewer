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

## 사용 예시

### review-pr — 3차 재리뷰

```
❯ 이제 이거 PR 198에 3차 리뷰 다시 해봐
```

**Step 1: PR 정보 수집**

```
PR: fix(mobile): DynamicHeader 제스처/터치 복구 및 키보드 dismiss 처리
작성자: somangoi
브랜치: b2c-4023 → stage
변경: 8 파일, +188/-102
```

**Step 1.5: 기존 리뷰 확인 및 resolve**

이전 리뷰 스레드를 GraphQL API로 조회하여 반영 여부를 확인합니다. 반영 완료된 스레드는 자동 resolve 처리합니다.

```
┌───┬────────────────────────────────┬───────────────────────────────────────────┬──────────────────────────┐
│ # │              파일              │            이전 코멘트 (요약)             │           상태           │
├───┼────────────────────────────────┼───────────────────────────────────────────┼──────────────────────────┤
│ 1 │ MainDrawerNavigator.tsx        │ collapsed 상태 터치 오버레이 미확인       │ Resolved (이전)          │
│ 2 │ DynamicHeaderStateProvider.tsx │ useMemo deps 누락                        │ Resolved (이전)          │
│ 3 │ MainDrawerNavigator.tsx        │ as const 불필요                          │ Resolved (이전)          │
│ 4 │ DynamicHeaderStateProvider.tsx │ headerMode useEffect deps 무한 되돌림    │ Resolved (이전)          │
│ 5 │ DynamicHeaderStateProvider.tsx │ useMemo deps 불필요                      │ Resolved (이전)          │
│ 6 │ HealthPlatformScreen.tsx       │ 매직 넘버 10                             │ Resolved (방금) - 반영됨 │
└───┴────────────────────────────────┴───────────────────────────────────────────┴──────────────────────────┘
```

**Step 2–3: 워크트리 생성 → 병렬 코드 리뷰**

3개 에이전트가 병렬로 코드를 분석합니다.

```
├─ Agent A: 버그/로직 오류 탐지
├─ Agent B: 성능/보안 이슈 탐지
└─ Agent C: 코드 품질/스타일 탐지
```

**Step 4: 결과 정리**

```
총 2개 이슈 발견: Bug 0 · Warning 0 · Minor 1 · Nit 1

Minor — MainDrawerNavigator.tsx:90
  높이 계산이 DynamicHeaderStateProvider와 중복. 추후 불일치 위험.

Nit — MainDrawerNavigator.tsx:212
  zIndex: 9999 매직 넘버. 상수로 분리하면 의도가 명확해짐.
```

**Step 5: 사용자 승인**

코멘트 게시 여부와 리뷰 액션(Approve / Request Changes / Comment)을 선택합니다.

---

## 설치

> **사전 요구사항**: [GitHub CLI (`gh`)](https://cli.github.com/) 설치 및 `gh auth login` 인증 완료

```
/install-plugin https://github.com/nathankim0/pr-reviewer
```

---

## 라이선스

[MIT](./LICENSE)
