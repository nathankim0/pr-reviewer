# PR Code Reviewer

자동으로 PR을 리뷰하고 GitHub에 인라인 코멘트를 게시합니다.

## 사용법

```
/pr-reviewer:review-pr <PR_URL>
```

**예시**: `/pr-reviewer:review-pr https://github.com/owner/repo/pull/123`

---

allowed-tools: Bash(gh *), Bash(git worktree *), Bash(git *), Bash(rm -rf /tmp/pr-review-*), Bash(cat *), Bash(wc *), Read, Grep, Glob, Task

## 입력 파싱

사용자가 제공한 인자에서 PR URL을 파싱하세요: `$ARGUMENTS`

PR URL 형식: `https://github.com/{owner}/{repo}/pull/{number}`

이 URL에서 `owner`, `repo`, `number`를 추출하세요. URL이 유효하지 않으면 올바른 형식을 안내하고 중단하세요.

---

## 워크플로우

아래 단계를 순서대로 수행하세요. **모든 출력은 한글로** 작성합니다.

### Step 1: PR 정보 수집

다음 명령어들을 **병렬로** 실행하여 PR 메타데이터를 수집하세요:

1. **PR 기본 정보**:
   ```bash
   gh pr view {number} --repo {owner}/{repo} --json title,body,author,baseRefName,headRefName,headRefOid,additions,deletions,changedFiles
   ```

2. **변경 파일 목록 + diff**:
   ```bash
   gh pr diff {number} --repo {owner}/{repo}
   ```

수집한 정보를 요약하여 사용자에게 보여주세요:
- PR 제목, 작성자, 브랜치 정보
- 변경 파일 수, 추가/삭제 라인 수

### Step 2: 워크트리 생성

PR 코드를 로컬에서 분석하기 위해 워크트리를 생성합니다:

```bash
REVIEW_DIR="/tmp/pr-review-{owner}-{repo}-{number}"
```

- 이미 해당 디렉토리가 존재하면 제거 후 재생성
- `gh repo clone {owner}/{repo} $REVIEW_DIR` 후 `cd $REVIEW_DIR && gh pr checkout {number}`
- 클론 실패 시 사용자에게 알리고 중단

### Step 3: 병렬 코드 리뷰

**중요**: diff에서 변경된 파일들만 리뷰 대상입니다. 변경되지 않은 파일은 리뷰하지 않습니다.

3개의 병렬 에이전트(Task 도구 사용)를 실행하여 코드를 분석합니다. 각 에이전트에게 다음 정보를 제공하세요:
- 리뷰 디렉토리 경로 (`$REVIEW_DIR`)
- 변경된 파일 목록
- PR diff 내용

#### Agent A: 버그/로직 오류 탐지
- 널 참조, 타입 불일치, 잘못된 조건문
- 경쟁 조건, 무한 루프 가능성
- 예외 미처리, 리소스 누수
- API 계약 위반
- diff에서 추가/변경된 코드만 리뷰
- **출력 형식**: JSON 배열 `[{"path": "파일경로", "line": 라인번호, "body": "**[Bug]** 설명", "severity": "bug"}]`

#### Agent B: 성능/보안 이슈 탐지
- N+1 쿼리, 불필요한 재렌더링
- SQL 인젝션, XSS, 인증 우회
- 민감 정보 노출, 안전하지 않은 의존성
- 메모리 누수, 비효율적 알고리즘
- diff에서 추가/변경된 코드만 리뷰
- **출력 형식**: JSON 배열 `[{"path": "파일경로", "line": 라인번호, "body": "**[Warning]** 설명", "severity": "warning"}]`

#### Agent C: 코드 품질/스타일 이슈 탐지
- 네이밍 컨벤션, 코드 중복
- 불필요한 복잡성, 매직 넘버
- 누락된 에러 처리, 미사용 import
- 코드 가독성, 유지보수성
- diff에서 추가/변경된 코드만 리뷰
- **출력 형식**: JSON 배열 `[{"path": "파일경로", "line": 라인번호, "body": "**[Minor]** 또는 **[Nit]** 설명", "severity": "minor|nit"}]`

**각 에이전트 주의사항**:
- `line`은 반드시 **PR의 diff에서 변경된 라인**이어야 합니다 (추가된 라인 기준). 변경되지 않은 라인에 코멘트를 달지 마세요.
- `path`는 레포지토리 루트 기준 상대 경로여야 합니다.
- 사소하지 않은 실제 이슈만 보고하세요. 불필요한 nit-pick은 최소화하세요.
- 반드시 유효한 JSON 배열만 출력하세요.

### Step 4: 결과 정리 및 표시

3개 에이전트의 결과를 병합하고 심각도별로 정리하세요:

| 심각도 | 설명 |
|--------|------|
| Bug | 버그 또는 로직 오류 |
| Warning | 성능/보안 이슈 |
| Minor | 코드 품질 개선 |
| Nit | 스타일/사소한 개선 |

**심각도 표기는 이모지 없이 텍스트(`[Bug]`, `[Warning]`, `[Minor]`, `[Nit]`)로만 표시합니다.**

사용자에게 테이블 형태로 보여주세요:

```
## 리뷰 결과 요약

총 {N}개 이슈 발견:
- Bug: {n}개
- Warning: {n}개
- Minor: {n}개
- Nit: {n}개

### 상세 내용

| 심각도 | 파일 | 라인 | 내용 |
|--------|------|------|------|
| Bug | path/to/file.ts | 42 | 설명... |
| ... | ... | ... | ... |
```

발견된 이슈가 없으면 "리뷰 결과 이슈가 없습니다. 코드가 깔끔합니다!" 라고 알리고, 워크트리를 정리한 뒤 종료하세요.

### Step 5: 사용자 승인

AskUserQuestion 도구를 사용하여 사용자에게 확인하세요:

**질문**: "위 리뷰 결과를 GitHub PR에 인라인 코멘트로 게시할까요?"

**옵션**:
1. "전체 게시" - 모든 이슈를 코멘트로 게시
2. "Bug/Warning만 게시" - Bug과 Warning 심각도만 게시
3. "게시하지 않음" - 코멘트를 게시하지 않고 종료

사용자가 "게시하지 않음"을 선택하면 워크트리를 정리하고 종료하세요.

### Step 6: GitHub 인라인 코멘트 게시

사용자가 승인한 코멘트들을 GitHub PR에 게시합니다.

**head SHA 가져오기**:
Step 1에서 수집한 `headRefOid`를 사용하세요.

**리뷰 본문 (body) 작성 규칙**:

리뷰 본문은 실제 동료 개발자가 코드를 읽고 남기는 리뷰처럼 자연스럽게 작성하세요. 기계적인 카운트 나열이 아니라, **PR의 변경 내용을 이해한 뒤 핵심 피드백을 요약하는 형태**여야 합니다.

좋은 예시:
```
전반적으로 DynamicHeader 터치 처리 개선 방향은 좋습니다. headerPressHandlerRef 패턴으로 zIndex 문제를 우회한 건 깔끔하네요.

다만 DynamicHeaderStateProvider의 useEffect에서 headerMode를 의존성에 넣으면서 의도치 않은 동작이 생길 수 있어 보입니다. 상세 내용은 인라인 코멘트로 남겼습니다.

그 외 사소한 매직 넘버 건도 같이 확인 부탁드립니다.
```

나쁜 예시 (이렇게 쓰지 마세요):
```
## AI 코드 리뷰 결과
총 5개 이슈를 발견했습니다.
- Bug: 1개
- Warning: 2개
```

**작성 가이드라인**:
- PR의 변경 의도를 먼저 인정하거나 언급
- 가장 중요한 이슈 1~2개를 자연어로 요약
- 나머지는 "인라인 코멘트 확인 부탁드립니다" 식으로 안내
- 이모지, 기계적 카운트 나열, "AI 코드 리뷰 결과" 같은 제목 금지
- 존댓말 사용, 동료에게 말하듯 자연스럽게

**리뷰 생성**:
```bash
cat <<'JSON' | gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input -
{
  "commit_id": "{head_sha}",
  "event": "COMMENT",
  "body": "{위 규칙에 따라 작성한 자연스러운 리뷰 본문}",
  "comments": [
    {
      "path": "파일경로",
      "line": 라인번호,
      "body": "**[심각도]** 설명"
    }
  ]
}
JSON
```

**주의사항**:
- `comments` 배열에는 사용자가 승인한 심각도의 코멘트만 포함
- JSON이 유효한지 확인 (특수문자 이스케이프 필요)
- API 호출 실패 시 에러 메시지를 사용자에게 보여주세요

게시 성공 시 PR URL을 다시 안내하세요.

### Step 7: 정리

워크트리를 제거합니다:
```bash
rm -rf /tmp/pr-review-{owner}-{repo}-{number}
```

완료 메시지를 출력하세요:
```
✅ PR #{number} 리뷰 완료! GitHub에 {N}개의 인라인 코멘트가 게시되었습니다.
🔗 {PR_URL}
```
