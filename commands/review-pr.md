# PR Code Reviewer

자동으로 PR을 리뷰하고 GitHub에 인라인 코멘트를 게시합니다.

## 사용법

```
/pr-reviewer:review-pr <PR_URL>
```

**예시**: `/pr-reviewer:review-pr https://github.com/owner/repo/pull/123`

---

allowed-tools: Bash(gh *), Bash(git worktree *), Bash(git *), Bash(rm -rf /tmp/pr-review-*), Bash(cat *), Bash(wc *), Read, Grep, Glob, Task, AskUserQuestion

---

## !! 필수 준수 사항 !!

**이 워크플로우의 모든 Step은 반드시 순서대로 실행해야 합니다. 단 하나의 Step도 건너뛰거나 생략할 수 없습니다.**

절대 하지 말아야 할 것:
- ❌ diff만 읽고 직접 리뷰를 작성하는 것 — 반드시 Task 도구로 에이전트를 실행하세요
- ❌ 워크트리(Step 2) 생성을 건너뛰는 것 — 에이전트가 전체 파일 컨텍스트를 읽어야 합니다
- ❌ **기존 리뷰 확인(Step 2.5)을 건너뛰는 것** — 재리뷰 시 기존 코멘트의 반영 여부를 반드시 확인하고, 반영된 스레드는 자동 resolve해야 합니다. 이 단계를 건너뛰면 미해결 스레드가 방치됩니다
- ❌ 사용자에게 리뷰 관점을 묻지 않는 것 (Step 1.5) — 반드시 AskUserQuestion으로 확인하세요
- ❌ 사용자에게 게시 여부를 묻지 않는 것 (Step 5) — 반드시 AskUserQuestion으로 확인하세요
- ❌ 사용자에게 리뷰 액션을 묻지 않는 것 (Step 5.5) — 반드시 AskUserQuestion으로 확인하세요
- ❌ GitHub API로 리뷰를 게시하지 않는 것 (Step 6) — 최종 목표는 GitHub에 리뷰를 남기는 것입니다
- ❌ **수동으로 리뷰/approve하는 것** — 반드시 이 워크플로우의 전체 Step을 따라야 합니다. 워크플로우 밖에서 `gh api`로 직접 리뷰를 게시하지 마세요

**체크포인트**: 각 Step 완료 후 다음 Step으로 넘어가기 전에, 해당 Step이 실제로 실행되었는지 확인하세요.

---

## 입력 파싱

사용자가 제공한 인자에서 PR URL을 파싱하세요: `$ARGUMENTS`

PR URL 형식: `https://github.com/{owner}/{repo}/pull/{number}`

이 URL에서 `owner`, `repo`, `number`를 추출하세요. URL이 유효하지 않으면 올바른 형식을 안내하고 중단하세요.

---

## 워크플로우

아래 단계를 **반드시 순서대로, 빠짐없이** 수행하세요. **모든 출력은 한글로** 작성합니다.

**실행 순서**: Step 1 → Step 1.5 → Step 2 → Step 2.5 → Step 3 → Step 4 → Step 5 → Step 5.5 → Step 6 → Step 7

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

3. **기존 리뷰 스레드 조회** (항상 실행):

   현재 사용자의 미해결 리뷰 스레드를 함께 조회합니다:
   ```bash
   gh api user --jq '.login'
   ```
   ```bash
   gh api graphql -F owner='{owner}' -F repo='{repo}' -F number={number} -f query='
   query($owner: String!, $repo: String!, $number: Int!) {
     repository(owner: $owner, name: $repo) {
       pullRequest(number: $number) {
         reviewThreads(first: 100) {
           nodes {
             id
             isResolved
             isOutdated
             path
             line
             comments(first: 1) {
               nodes {
                 author { login }
                 body
                 originalCommit { oid }
               }
             }
           }
         }
       }
     }
   }'
   ```

수집한 정보를 요약하여 사용자에게 보여주세요:
- PR 제목, 작성자, 브랜치 정보
- 변경 파일 수, 추가/삭제 라인 수

### Step 1.5: 리뷰 관점 확인

PR 정보를 보여준 뒤, AskUserQuestion 도구를 사용하여 리뷰 방향을 확인하세요.

**질문**: "이 PR에서 어떤 부분을 중점적으로 리뷰할까요?"

**옵션**:
1. "전체 리뷰 (추천)" - 버그, 성능, 보안, 코드 품질 모두 검토합니다
2. "버그/로직만" - 치명적인 버그와 로직 오류에 집중합니다
3. "성능/보안만" - 성능 병목과 보안 취약점에 집중합니다
4. "코드 품질만" - 네이밍, 구조, 가독성 등에 집중합니다

사용자가 "Other"로 직접 입력하면, 해당 내용을 리뷰 관점으로 사용합니다 (예: "상태 관리 로직 위주로 봐줘", "이 PR에서 API 에러 처리가 제대로 됐는지 확인해줘").

**선택에 따른 에이전트 실행**:

| 선택 | 실행할 에이전트 |
|------|----------------|
| 전체 리뷰 | Agent A + B + C (기존과 동일) |
| 버그/로직만 | Agent A만 실행 |
| 성능/보안만 | Agent B만 실행 |
| 코드 품질만 | Agent C만 실행 |
| 사용자 커스텀 입력 | Agent A + B + C 모두 실행하되, 각 에이전트에게 **사용자 지정 관점**을 추가 컨텍스트로 전달 |

### Step 2: 워크트리 생성

PR 코드를 로컬에서 분석하기 위해 워크트리를 생성합니다:

```bash
REVIEW_DIR="/tmp/pr-review-{owner}-{repo}-{number}"
```

- 이미 해당 디렉토리가 존재하면 제거 후 재생성
- **PR 브랜치를 직접 클론** (shallow clone + `gh pr checkout` 조합은 브랜치 ref를 찾지 못하는 문제가 발생할 수 있음):
  ```bash
  gh repo clone {owner}/{repo} $REVIEW_DIR -- --single-branch --branch {headRefName}
  ```
  `{headRefName}`은 Step 1에서 수집한 PR의 head 브랜치명입니다.
- 클론 실패 시 사용자에게 알리고 중단

### Step 2.5: 기존 리뷰 확인 및 resolve

**현재 사용자가 남긴 미해결(`isResolved === false`) 스레드**가 있으면 이 단계를 실행합니다. 없으면 Step 3으로 진행합니다.

**1) 반영 여부 확인**:

코멘트가 달린 커밋(`originalCommit.oid`)과 현재 HEAD(`headRefOid`)가 다른 경우, 이후 커밋에서 변경된 파일을 확인합니다:
```bash
gh api repos/{owner}/{repo}/compare/{original_commit_oid}...{head_ref_oid} --jq '.files[].filename'
```

각 기존 코멘트에 대해:
- 코멘트가 달린 파일이 이후 커밋에서 변경되었으면 → **워크트리(`$REVIEW_DIR`)에서 해당 파일의 현재 코드를 Read**하여 코멘트의 지적 사항이 **실제로 반영되었는지** 판단
- `isOutdated === true`라도 반영 여부는 코드를 직접 읽어서 판단 (라인 변경 ≠ 반영)
- 파일이 변경되지 않았으면 → 미반영으로 판단

**2) resolve 처리**:

반영이 확인된 스레드는 GraphQL mutation으로 resolve 처리합니다.

**중요**: GraphQL 변수 문법(`$var`)은 bash `$` 해석 문제를 일으킬 수 있으므로, mutation은 반드시 **인라인 값**으로 작성하세요:
```bash
gh api graphql -f query='mutation { resolveReviewThread(input: { threadId: "{thread_node_id}" }) { thread { isResolved } } }'
```

각 스레드별로 개별 호출하세요. `{thread_node_id}`는 GraphQL 조회에서 얻은 스레드의 `id` 필드 값입니다.

**반영된 스레드가 여러 개이면 병렬로 resolve 처리하세요.**

**3) 결과 표시**:
```
## 기존 리뷰 확인 결과

| # | 파일 | 이전 코멘트 (요약) | 상태 |
|---|------|-------------------|------|
| 1 | file.ts:42 | `useEffect` 의존성 문제 | Resolved |
| 2 | file.ts:58 | 매직 넘버 | 미반영 |
```

미반영된 코멘트가 있으면 사용자에게 알려주고, 이후 Step 3 리뷰에서 동일 이슈를 중복으로 지적하지 않습니다.

### Step 3: 병렬 코드 리뷰

**중요**: diff에서 변경된 파일들만 리뷰 대상입니다. 변경되지 않은 파일은 리뷰하지 않습니다.

**!! 절대로 직접 리뷰를 작성하지 마세요 !!** 반드시 아래의 Task 에이전트를 실행하여 리뷰를 수행해야 합니다. 당신이 diff를 읽고 직접 이슈를 작성하는 것은 금지됩니다.

**Step 1.5에서 선택한 리뷰 관점에 따라** 해당 에이전트만 실행합니다. 각 에이전트(Task 도구 사용)에게 다음 정보를 제공하세요:
- 리뷰 디렉토리 경로 (`$REVIEW_DIR`)
- 변경된 파일 목록
- PR diff 내용
- (재리뷰 시) 기존 미반영 코멘트 목록 — 동일 이슈 중복 지적 방지용
- (사용자 커스텀 입력 시) 사용자가 지정한 리뷰 관점 — 해당 관점을 우선적으로 분석

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
- `line`은 반드시 **PR diff의 hunk 범위 내에 있는 라인**이어야 합니다 (새 파일 기준 라인 번호). diff hunk 밖의 라인에 코멘트를 달면 GitHub API가 422 에러를 반환합니다.
- `path`는 레포지토리 루트 기준 상대 경로여야 합니다.
- 사소하지 않은 실제 이슈만 보고하세요. 불필요한 nit-pick은 최소화하세요.
- 반드시 유효한 JSON 배열만 출력하세요.

### Step 4: 결과 정리 및 라인 검증

3개 에이전트의 결과를 병합하고 심각도별로 정리하세요:

| 심각도 | 설명 |
|--------|------|
| Bug | 버그 또는 로직 오류 |
| Warning | 성능/보안 이슈 |
| Minor | 코드 품질 개선 |
| Nit | 스타일/사소한 개선 |

**심각도 표기는 이모지 없이 텍스트(`[Bug]`, `[Warning]`, `[Minor]`, `[Nit]`)로만 표시합니다.**

#### 라인 검증 (중요)

GitHub PR Review API는 **diff hunk 범위 내의 라인에만** 코멘트를 달 수 있습니다. 범위 밖 라인에 코멘트하면 `422 Unprocessable Entity` 에러가 발생합니다.

**검증 방법**: PR diff의 각 hunk 헤더 (`@@ -old_start,old_count +new_start,new_count @@`)에서 `new_start`와 `new_count`를 파싱하여, 각 코멘트의 `line`이 `[new_start, new_start + new_count - 1]` 범위 내에 있는지 확인합니다.

**범위 밖 코멘트 처리**:
- diff hunk 범위 밖의 코멘트는 인라인 코멘트에서 제외
- 대신 리뷰 body 본문에 해당 내용을 포함시킵니다
- 사용자에게 "일부 코멘트는 diff 범위 밖이라 본문에 포함했습니다"라고 안내

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

발견된 이슈가 없으면 "리뷰 결과 이슈가 없습니다. 코드가 깔끔합니다!" 라고 알리고, **Step 5는 건너뛰고 Step 5.5로** 진행하세요 (Approve 여부 확인).

### Step 5: 사용자 승인 (코멘트 게시 여부)

**이슈가 있을 때만 실행합니다.** (이슈가 없으면 Step 5.5로 건너뜁니다.)

AskUserQuestion 도구를 사용하여 사용자에게 확인하세요:

**질문**: "위 리뷰 결과를 GitHub PR에 인라인 코멘트로 게시할까요?"

**옵션**:
1. "전체 게시" - 모든 이슈를 코멘트로 게시
2. "Bug/Warning만 게시" - Bug과 Warning 심각도만 게시
3. "게시하지 않음" - 코멘트를 게시하지 않고 종료

사용자가 "게시하지 않음"을 선택하면 워크트리를 정리하고 종료하세요.

### Step 5.5: 리뷰 액션 선택

코멘트 게시와 함께 PR에 대한 리뷰 액션을 결정합니다. AskUserQuestion으로 확인하세요.

**심각도 기반 추천 로직**:
- **Bug 이슈가 없을 때** → Approve 추천
- **Bug 이슈가 있을 때** → Request Changes 추천

**질문**: "리뷰 액션을 선택해주세요."

**옵션 (Bug 없을 때)**:
1. "Approve (추천)" - PR을 승인합니다
2. "Request Changes" - 변경을 요청합니다
3. "Comment만" - 액션 없이 코멘트만 남깁니다

**옵션 (Bug 있을 때)**:
1. "Request Changes (추천)" - 변경을 요청합니다
2. "Approve" - PR을 승인합니다
3. "Comment만" - 액션 없이 코멘트만 남깁니다

선택에 따라 리뷰 API의 `event` 값이 결정됩니다:
| 선택 | `event` 값 |
|------|-----------|
| Approve | `APPROVE` |
| Request Changes | `REQUEST_CHANGES` |
| Comment만 | `COMMENT` |

### Step 6: GitHub 리뷰 게시

사용자가 선택한 코멘트와 리뷰 액션을 GitHub PR에 게시합니다.

**head SHA 가져오기**:
Step 1에서 수집한 `headRefOid`를 사용하세요.

**리뷰 본문 (body) 작성 규칙**:

리뷰 본문은 실제 동료 개발자가 코드를 읽고 남기는 리뷰처럼 자연스럽게 작성하세요. 기계적인 카운트 나열이 아니라, **PR의 변경 내용을 이해한 뒤 핵심 피드백을 요약하는 형태**여야 합니다.

좋은 예시:
```
DynamicHeader 터치 처리 개선 방향 좋은 것 같아요! `contentContainer` zIndex 문제를 `ArrivedHeaderTouchOverlay` + `headerPressHandlerRef` 패턴으로 우회한 거 깔끔하고, `tempoConfig` 타입을 모델 레벨에서 정리해서 소비처 타입 캐스팅 없앤 것도 좋네요.

한 가지 신경 쓰이는 건, `DynamicHeaderStateProvider`의 `useEffect`에서 `headerMode`를 의존성 배열에 넣으면서 proposals 있는 상태에서 사용자가 헤더를 접을 수 없는 문제가 생길 수 있을 것 같아요. 자동 펼침 로직을 별도 `useEffect`로 분리하거나 `proposals.length` 변경 시에만 동작하도록 하면 어떨까요?

나머지 `useMemo` 의존성 정리 건이랑 매직 넘버 건도 인라인 코멘트로 남겨뒀으니 확인해주세요~
```

나쁜 예시 (이렇게 쓰지 마세요):
```
## AI 코드 리뷰 결과
총 5개 이슈를 발견했습니다.
- Bug: 1개
- Warning: 2개
```

**작성 가이드라인**:
- 동료 개발자에게 말하듯 편한 어투 (~것 같아요, ~좋네요, ~어떨까요?, ~확인해주세요~)
- PR의 변경 의도를 먼저 인정하거나 언급
- 가장 중요한 이슈 1~2개를 자연어로 요약
- 나머지는 "인라인 코멘트로 남겨뒀으니 확인해주세요" 식으로 안내
- 코드 관련 단어(변수명, 함수명, 컴포넌트명 등)는 반드시 인라인 코드 포맷(백틱)으로 감싸기
- 이모지, 기계적 카운트 나열, "AI 코드 리뷰 결과" 같은 제목 금지
- **Approve 시**: 본문 톤을 긍정적으로 마무리 (~좋은 것 같아요!, 깔끔하네요~)
- **Request Changes 시**: 수정이 필요한 핵심 이유를 자연스럽게 언급
- **재리뷰 시**: 이전 코멘트 반영 결과를 자연스럽게 언급 (예: "이전 피드백 잘 반영해주셨네요!")
- **diff 범위 밖 코멘트가 있으면**: 본문에 해당 내용을 포함 (예: "추가로 diff 범위 밖이라 인라인 코멘트로 남기지 못한 부분이 있어요: ...")

**리뷰 생성**:

`{event}`는 Step 5.5에서 선택한 값 (`APPROVE`, `REQUEST_CHANGES`, 또는 `COMMENT`)을 사용합니다.

```bash
cat <<'JSON' | gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input -
{
  "commit_id": "{head_sha}",
  "event": "{event}",
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

**이슈가 없어서 코멘트 없이 Approve만 하는 경우**:
```bash
cat <<'JSON' | gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input -
{
  "commit_id": "{head_sha}",
  "event": "APPROVE",
  "body": "{PR 변경 내용을 언급하며 자연스럽게 승인하는 본문}"
}
JSON
```

**주의사항**:
- `comments` 배열에는 사용자가 승인한 심각도의 코멘트만 포함
- **라인 검증을 통과한 코멘트만** `comments` 배열에 포함 (Step 4에서 검증)
- JSON이 유효한지 확인 (특수문자 이스케이프 필요)
- API 호출 실패 시 에러 메시지를 사용자에게 보여주세요
- **422 에러 발생 시**: 문제가 되는 코멘트를 제거하고 나머지만 재시도

게시 성공 시 PR URL과 리뷰 액션을 안내하세요.

### Step 6.5: APPROVE 시 미해결 스레드 자동 resolve

**APPROVE 리뷰를 게시한 경우에만 실행합니다.** (REQUEST_CHANGES, COMMENT인 경우 건너뜁니다.)

APPROVE = "모든 이슈가 해결됨"을 의미하므로, 현재 사용자가 남긴 **모든 미해결 스레드를 resolve** 처리합니다.

**1) 미해결 스레드 조회**:

Step 1에서 수집한 현재 사용자 로그인과 리뷰 스레드 정보를 사용하되, Step 2.5에서 이미 resolve한 스레드는 제외합니다. 만약 Step 1 이후 새 코멘트가 추가되었을 가능성이 있으므로, **GraphQL로 최신 미해결 스레드를 다시 조회**합니다:

```bash
gh api graphql -F owner='{owner}' -F repo='{repo}' -F number={number} -f query='
query($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          comments(first: 1) {
            nodes {
              author { login }
            }
          }
        }
      }
    }
  }
}'
```

**2) 현재 사용자의 미해결 스레드 필터링 및 resolve**:

첫 번째 코멘트의 `author.login`이 현재 사용자인 미해결(`isResolved === false`) 스레드를 모두 resolve 처리합니다:

```bash
gh api graphql -f query='mutation { resolveReviewThread(input: { threadId: "{thread_node_id}" }) { thread { isResolved } } }'
```

**병렬로 실행하세요.** resolve 결과 수를 기록하여 Step 7 완료 메시지에 포함합니다.

### Step 7: 정리

워크트리를 제거합니다:
```bash
rm -rf /tmp/pr-review-{owner}-{repo}-{number}
```

완료 메시지를 출력하세요 (리뷰 액션 포함):
```
PR #{number} 리뷰 완료!
- 리뷰 액션: {Approved / Changes Requested / Commented}
- 인라인 코멘트: {N}개 게시
{PR_URL}
```

기존 리뷰 코멘트가 있었던 경우 resolve 결과도 함께 표시:
```
PR #{number} 리뷰 완료!
- 리뷰 액션: {Approved / Changes Requested / Commented}
- 기존 코멘트 {M}개 중 {R}개 resolved
- 새 코멘트 {N}개 게시
{PR_URL}
```
