# PR 리뷰 코멘트 해결

PR에 달린 미해결 리뷰 코멘트를 분석하고, 코드를 수정하고, GitHub에 답글을 게시합니다.

## 사용법

```
/pr-reviewer:resolve-reviews [PR_URL 또는 PR_번호]
```

**예시**:
- `/pr-reviewer:resolve-reviews https://github.com/owner/repo/pull/123`
- `/pr-reviewer:resolve-reviews 123`
- `/pr-reviewer:resolve-reviews` (현재 브랜치의 PR 자동 감지)

---

allowed-tools: Bash(gh *), Bash(git status *), Bash(git log *), Bash(git diff *), Bash(git branch *), Bash(git rev-parse *), Bash(git fetch *), Read, Edit, Write, Grep, Glob, Task, AskUserQuestion

---

## 워크플로우

아래 단계를 순서대로 수행하세요. **모든 출력은 한글로** 작성합니다.

### Step 0: 입력 파싱 및 환경 검증

사용자 인자: `$ARGUMENTS`

**PR 식별**:
1. `$ARGUMENTS`가 GitHub PR URL (`https://github.com/{owner}/{repo}/pull/{number}`)이면 → `owner`, `repo`, `number` 추출
2. `$ARGUMENTS`가 숫자면 → PR 번호로 사용, `owner`/`repo`는 현재 레포에서 추출:
   ```bash
   gh repo view --json owner,name --jq '.owner.login + "/" + .name'
   ```
3. `$ARGUMENTS`가 비어있으면 → 현재 브랜치의 PR 자동 감지:
   ```bash
   gh pr view --json number,url --jq '.number'
   ```

**환경 검증** (병렬 실행):
1. 현재 브랜치 확인:
   ```bash
   git rev-parse --abbrev-ref HEAD
   ```
2. PR 상세 정보 조회:
   ```bash
   gh pr view {number} --repo {owner}/{repo} --json state,headRefName,author,title
   ```

**검증 조건**:
- PR이 `OPEN` 상태인지 확인. 아니면 "이 PR은 이미 닫혔거나 머지되었습니다."라고 안내하고 중단
- 현재 브랜치가 PR의 `headRefName`과 일치하는지 확인. 불일치 시 경고 표시 후 AskUserQuestion으로 계속 진행 여부 확인

검증 통과 시 PR 기본 정보를 표시하세요:
```
## PR #{number}: {title}
- 작성자: {author}
- 브랜치: {headRefName}
```

### Step 1: 리뷰 코멘트 수집

GraphQL API로 미해결 리뷰 스레드를 조회합니다:

```bash
gh api graphql -f query='
query($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      author { login }
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          startLine
          diffSide
          comments(first: 20) {
            nodes {
              id
              databaseId
              author { login }
              body
              createdAt
              path
              line
              startLine
              diffSide
            }
          }
        }
      }
    }
  }
}' -F owner='{owner}' -F repo='{repo}' -F number={number}
```

**필터링 규칙**:
1. `isResolved === true`인 스레드는 제외
2. 스레드의 첫 번째 코멘트 작성자가 PR 작성자(`author.login`)와 같으면 제외 (자기 자신의 코멘트)
3. `isOutdated === true`인 스레드는 별도 표시 (코드가 이미 변경됨)

**Outdated 스레드 diff 검증**:
`isOutdated === true`인 스레드에 대해서는 해당 파일의 변경 이력을 확인하여 실제로 반영되었는지 판단합니다:
```bash
git diff {base_branch}...HEAD -- {file_path}
```
- diff에서 코멘트가 지적한 라인이 이미 수정되었으면 → Step 2에서 자동으로 `already_done`으로 분류
- diff에서 해당 라인이 변경되지 않았으면 → 여전히 미해결로 간주하여 정상 분류 진행

미해결 코멘트가 없으면 "미해결 리뷰 코멘트가 없습니다! 모든 리뷰가 해결되었습니다." 라고 안내하고 종료하세요.

파일별로 그룹핑하여 사용자에게 보여주세요:
```
## 미해결 리뷰 코멘트: {N}개

| # | 파일 | 라인 | 작성자 | 내용 (요약) | 오래됨 |
|---|------|------|--------|-------------|--------|
| 1 | path/to/file.ts | 42 | reviewer1 | 코멘트 요약... | - |
| 2 | path/to/file.ts | 58 | reviewer2 | 코멘트 요약... | ⚠️ |
```

### Step 2: 코멘트 분석 및 분류

각 코멘트를 분석하여 5가지 카테고리로 분류하세요. 분류 시 다음을 고려합니다:
- 코멘트 내용과 맥락
- 현재 코드 상태 (파일을 Read하여 확인)
- Step 1에서 수행한 diff 검증 결과 (`isOutdated` + diff 확인으로 이미 반영된 코멘트는 자동 `already_done`)
- GitHub suggestion 블록 포함 여부

**카테고리**:

| 카테고리 | 아이콘 | 설명 | 액션 |
|----------|--------|------|------|
| `code_change` | 🔧 | 코드 변경이 필요 | 코드 수정 + 답글 |
| `question` | 💬 | 질문에 답변 필요 | 답글만 |
| `already_done` | ✅ | 이미 반영되었거나 해결됨 | 답글만 |
| `disagree` | 🤔 | 동의하지 않음 (근거 필요) | 답글만 |
| `unclear` | ❓ | 의도가 불명확 | 사용자에게 확인 필요 |

분류 결과를 테이블로 표시:
```
## 코멘트 분석 결과

| # | 분류 | 파일 | 내용 (요약) | 계획 |
|---|------|------|-------------|------|
| 1 | 🔧 code_change | file.ts:42 | 변수명 변경 필요 | rename 수행 |
| 2 | 💬 question | file.ts:58 | 이 로직의 이유? | 설계 의도 설명 |
| 3 | ✅ already_done | util.ts:10 | null 체크 추가 | 이미 반영됨을 안내 |
```

`unclear` 카테고리가 있으면 AskUserQuestion으로 사용자에게 해당 코멘트의 의도를 확인하세요.

### Step 3: 사용자 1차 승인

AskUserQuestion을 사용하여 진행 방식을 확인하세요:

**질문**: "위 분석 결과대로 리뷰 코멘트를 처리할까요?"

**옵션**:
1. "전체 진행" - 코드 변경 + 답글 모두 처리
2. "코드 변경만" - 코드 수정만 수행, 답글은 게시하지 않음
3. "중단" - 아무것도 하지 않고 종료

사용자가 "중단"을 선택하면 종료하세요.

### Step 4: 파일별 코드 변경

`code_change`로 분류된 코멘트들을 파일별로 그룹핑하여 처리합니다.

**GitHub suggestion 블록 처리**:
코멘트 본문에 다음 형식의 suggestion이 포함되어 있으면 해당 내용을 그대로 적용하세요:
````
```suggestion
수정된 코드 내용
```
````

**파일별 병렬 처리** (Task 에이전트 사용):

각 파일 그룹에 대해 Task 에이전트를 생성하세요. 에이전트에게 제공할 정보:
- 파일 경로
- 해당 파일의 코멘트 목록 (라인, 내용, suggestion 포함 여부)
- 수행할 작업 설명

각 에이전트의 작업:
1. Read로 해당 파일의 현재 내용 확인
2. 각 코멘트에 대해:
   - suggestion 블록이 있으면 → 해당 라인을 suggestion 내용으로 교체
   - suggestion이 없으면 → 코멘트 의도를 파악하여 적절히 코드 수정
3. Edit 도구로 코드 수정 적용
4. 수정 내역과 각 코멘트에 대한 답글 초안을 반환

**주의사항**:
- 코멘트의 의도를 정확히 파악하여 수정하세요
- 수정 범위를 최소화하세요 (요청된 부분만 변경)
- 수정으로 인한 부작용이 없는지 주변 코드를 확인하세요

### Step 5: 변경 사항 요약 및 2차 승인

코드 변경이 완료되면 변경 내역과 답글 미리보기를 표시하세요:

```
## 변경 사항 요약

### 코드 변경
| 파일 | 변경 내용 |
|------|-----------|
| file.ts | 변수명 `foo` → `bar` 변경 (L42) |

### 답글 미리보기
| # | 파일 | 답글 내용 |
|---|------|-----------|
| 1 | file.ts:42 | 변수명을 `bar`로 변경했습니다. |
| 2 | file.ts:58 | 이 로직은 ~한 이유로 설계되었습니다. |
```

AskUserQuestion으로 확인:

**질문**: "답글을 GitHub에 게시할까요?"

**옵션**:
1. "답글 게시" - 모든 답글을 GitHub에 게시
2. "수정 필요" - 답글 내용을 수정하고 싶음
3. "게시하지 않음" - 답글은 게시하지 않고 코드 변경만 유지

사용자가 "수정 필요"를 선택하면 어떤 답글을 수정할지 확인하고 수정 후 다시 미리보기를 표시하세요.

### Step 6: GitHub 답글 게시

승인된 답글을 GitHub에 게시합니다.

각 스레드의 **첫 번째 코멘트의 `databaseId`**를 사용하여 답글을 게시하세요:

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_database_id}/replies \
  --method POST -f body='{reply_body}'
```

**답글 형식 규칙**:
- `code_change` 답글: 수정 내용을 구체적으로 설명
- `question` 답글: 질문에 대한 답변
- `already_done` 답글: 이미 반영되었음을 안내
- `disagree` 답글: 동의하지 않는 이유를 정중하게 설명

**모든 답글 끝에 푸터 추가**:
```
_Resolved by Claude Code_ 🤖
```

**오류 처리**:
- 개별 답글 게시 실패 시 해당 답글만 실패로 표시하고 나머지는 계속 진행
- 실패한 답글 목록을 마지막에 표시

### Step 7: 결과 보고

처리 결과를 요약하여 보고하세요:

```
## 처리 완료

### 결과 요약
- 🔧 코드 변경: {n}건
- 💬 답글 게시: {n}건 (성공 {n} / 실패 {n})
- ✅ 이미 해결: {n}건
- 🤔 동의하지 않음: {n}건

### 다음 단계
변경된 코드를 커밋하려면 `/commit` 명령어를 사용하세요.
```
