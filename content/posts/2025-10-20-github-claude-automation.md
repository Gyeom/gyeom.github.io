---
title: "AI 자동화 블로그 만들기 (2) - Claude Code로 이슈 기반 자동 포스팅"
date: 2025-10-20
draft: true
tags: ["GitHub Actions", "Claude Code", "자동화", "CI/CD", "AI"]
categories: ["개발환경"]
series: ["AI 자동화 블로그"]
summary: "GitHub 이슈에서 @claude를 멘션하면 Claude가 포스트를 작성하고 PR까지 자동 생성하는 워크플로우를 구축한다. 시리즈의 두 번째 글."
---

> **시리즈**: AI 자동화 블로그 만들기
> 1. [Hugo + GitHub Pages 구축](/dev-notes/posts/2024-11-29-hugo-github-pages-auto-blog/)
> 2. **Claude Code로 이슈 기반 자동 포스팅** ← 현재 글

---

## 개요

[이전 글](/dev-notes/posts/2024-11-29-hugo-github-pages-auto-blog/)에서 Hugo 블로그와 GitHub Pages 자동 배포를 구축했다. 이번에는 GitHub 이슈에서 `@claude`를 멘션하면 Claude가 포스트를 작성하고 PR까지 자동 생성하는 워크플로우를 추가한다.

코드 작성, 문서 생성, 버그 수정 등 다양한 작업을 이슈 하나로 요청할 수 있다. 리뷰 프로세스를 유지하면서도 반복적인 작업을 자동화할 수 있어 생산성이 크게 향상된다.

## 완성된 워크플로우

최종 흐름은 다음과 같다.

```mermaid
flowchart TD
    A[이슈에 @claude 멘션] --> B[GitHub Actions 트리거]
    B --> C[Claude Code 실행]
    C --> D{파일 변경?}
    D -->|Yes| E[브랜치 생성 + 커밋]
    D -->|No| F[결과 코멘트만 작성]
    E --> G[PR 자동 생성]
    G --> H[리뷰 후 머지]
    H --> I[GitHub Pages 배포]
    I --> J[이슈 자동 Close]
```

---

## 1. 사전 준비

### Anthropic API 키 등록

Claude API를 사용하려면 Anthropic API 키가 필요하다. [Anthropic Console](https://console.anthropic.com/)에서 발급받을 수 있다.

GitHub 저장소 Settings > Secrets and variables > Actions에서 `ANTHROPIC_API_KEY`를 등록한다.

```bash
gh secret set ANTHROPIC_API_KEY --repo username/repo-name
```

### Actions 권한 설정

GitHub Actions가 PR을 생성하려면 쓰기 권한이 필요하다. 기본적으로 읽기 전용이므로 별도로 설정해야 한다.

```bash
gh api repos/username/repo-name/actions/permissions/workflow \
  -X PUT \
  -f default_workflow_permissions="write" \
  -F can_approve_pull_request_reviews=true
```

---

## 2. 워크플로우 파일 작성

핵심이 되는 워크플로우 파일이다. 이슈 이벤트를 감지하고, Claude Code를 설치해 실행한 뒤, 변경사항이 있으면 PR을 생성한다.

`.github/workflows/claude.yml`:

```yaml
name: Claude AI Assistant

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]

# 동일 이슈에서 중복 실행 방지
concurrency:
  group: claude-${{ github.event.issue.number || github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  claude-response:
    if: contains(github.event.comment.body, '@claude') || contains(github.event.issue.body, '@claude')
    runs-on: ubuntu-latest
    timeout-minutes: 10

    permissions:
      contents: write
      issues: write
      pull-requests: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          # PR 코멘트인 경우 PR 브랜치를 checkout
          ref: ${{ github.event.issue.pull_request && format('refs/pull/{0}/head', github.event.issue.number) || github.ref }}

      - name: Setup Git
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run Claude
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          COMMENT_BODY: ${{ github.event.comment.body || github.event.issue.body }}
          IS_PR: ${{ github.event.issue.pull_request && 'true' || 'false' }}
        run: |
          set -euo pipefail

          USER_REQUEST=$(echo "$COMMENT_BODY" | sed 's/.*@claude//')

          # PR인 경우 맥락 정보 수집
          if [ "$IS_PR" = "true" ]; then
            PR_TITLE=$(gh pr view ${ISSUE_NUMBER} --json title --jq '.title')
            PR_BRANCH=$(gh pr view ${ISSUE_NUMBER} --json headRefName --jq '.headRefName')
            PR_DIFF=$(gh pr diff ${ISSUE_NUMBER} --name-only | head -20)

            PROMPT="현재 PR #${ISSUE_NUMBER}에서 작업 중이다. PR 제목: ${PR_TITLE}, 브랜치: ${PR_BRANCH}. 변경된 파일: ${PR_DIFF}. 사용자 요청: ${USER_REQUEST}. 파일을 수정해야 한다면 Edit 도구를 사용해서 직접 수정해라."
          else
            PROMPT="${USER_REQUEST}"
          fi

          # Claude 실행 (CI 환경에서 권한 자동 승인)
          if RESPONSE=$(claude -p "$PROMPT" --max-turns 30 --dangerously-skip-permissions 2>&1); then
            CLAUDE_STATUS="success"
          else
            CLAUDE_STATUS="error"
          fi

          # 변경사항 확인 및 커밋
          if [ "$CLAUDE_STATUS" = "success" ] && [ -n "$(git status --porcelain)" ]; then
            git add .
            git commit -m "feat: Claude가 #${ISSUE_NUMBER} 요청 처리

          🤖 Generated by Claude Code"

            if [ "$IS_PR" = "true" ]; then
              # PR인 경우: 해당 브랜치에 직접 push
              PR_BRANCH=$(gh pr view ${ISSUE_NUMBER} --json headRefName --jq '.headRefName')
              git push origin HEAD:"refs/heads/${PR_BRANCH}"
              RESULT_MSG="✅ PR 브랜치(${PR_BRANCH})에 변경사항을 push했습니다."
            else
              # 일반 이슈인 경우: 새 브랜치 생성 후 PR
              BRANCH_NAME="claude/issue-${ISSUE_NUMBER}-$(date +%Y%m%d%H%M%S)"
              git checkout -b "$BRANCH_NAME"
              git push -u origin "$BRANCH_NAME"

              PR_URL=$(gh pr create \
                --title "Claude: 이슈 #${ISSUE_NUMBER} 요청 처리" \
                --body "Closes #${ISSUE_NUMBER}" \
                --base main \
                --head "$BRANCH_NAME")

              RESULT_MSG="✅ PR이 생성되었습니다: $PR_URL"
            fi
          else
            RESULT_MSG="ℹ️ 파일 변경사항이 없습니다."
          fi

          gh issue comment ${ISSUE_NUMBER} --body "## Claude 응답

          $RESPONSE

          ---
          $RESULT_MSG

          *🤖 Generated by Claude Code*"
```

### 핵심 포인트

| 항목 | 설명 |
|------|------|
| `concurrency` | 동일 이슈에서 중복 실행 방지 |
| `COMMENT_BODY`를 env로 전달 | 셸 이스케이프 문제 방지 |
| `--dangerously-skip-permissions` | CI 환경에서 파일 수정 권한 자동 승인 |
| `--max-turns 30` | 복잡한 작업을 위해 충분한 턴 수 확보 |
| PR 브랜치 checkout | PR 코멘트 시 해당 브랜치에서 작업 |
| PR 맥락 정보 전달 | PR 제목, 브랜치, 변경 파일 목록 제공 |

워크플로우는 `@claude` 멘션이 포함된 이슈나 코멘트에만 반응한다. **PR 코멘트인 경우 PR 브랜치를 checkout하고, 변경사항을 해당 브랜치에 직접 push한다.** 일반 이슈인 경우 새 브랜치를 만들어 PR을 생성한다.

---

## 3. 실제 사용 예시

실제로 이 워크플로우를 사용해 블로그 포스트를 자동 생성한 과정이다.

### 이슈 생성

![이슈 목록](/dev-notes/images/github-claude-automation/01-issues-list.png)
![이슈 상세](/dev-notes/images/github-claude-automation/02-issue-detail.png)

이슈를 생성하고 본문에 `@claude`를 멘션한다. 멘션 뒤에 원하는 작업을 자연어로 설명하면 된다.

```
@claude OpenFGA와 ReBAC(Relationship-Based Access Control)에 대한 포스트를 content/posts/ 폴더에 작성해줘.

주요 내용:
- ReBAC 개념과 기존 RBAC와의 차이점
- OpenFGA 소개 및 아키텍처
- 스키마 정의 방법 (DSL)
- 관계 튜플과 권한 체크 방식
- Spring Boot 연동 예시
- 실제 사용 사례 (Google Drive, GitHub 등)
```

### Claude 응답 및 PR 생성

이슈가 생성되면 GitHub Actions가 트리거되고, Claude Code가 요청을 처리한다. 파일 생성이 완료되면 자동으로 PR이 만들어진다.

![PR 목록](/dev-notes/images/github-claude-automation/03-pr-list.png)

### PR 상세

![PR 상세](/dev-notes/images/github-claude-automation/04-pr-detail.png)

PR 본문에 `Closes #9`가 포함되어 있어 머지 시 원본 이슈가 자동으로 닫힌다. 이렇게 이슈와 PR이 연결되어 작업 추적이 용이하다.

### 워크플로우 실행 로그

![워크플로우 실행](/dev-notes/images/github-claude-automation/06-workflow-run.png)

Actions 탭에서 워크플로우 실행 상태와 로그를 확인할 수 있다. 성공 여부, 실행 시간, 각 단계별 출력을 볼 수 있어 디버깅에 유용하다.

---

## 4. GITHUB_TOKEN 동작 원리

워크플로우에서 `${{ secrets.GITHUB_TOKEN }}`을 별도 설정 없이 사용할 수 있다. 이 토큰이 어떻게 동작하는지 알아보자.

### 자동 생성 메커니즘

1. 워크플로우 실행 시 GitHub이 임시 토큰 발급
2. 해당 저장소에 대한 권한만 부여
3. 워크플로우 종료 시 토큰 자동 폐기

### PAT vs GITHUB_TOKEN

| 구분 | GITHUB_TOKEN | PAT |
|------|--------------|-----|
| 범위 | 해당 저장소만 | 여러 저장소 |
| 수명 | 워크플로우 실행 중만 | 수동 관리 |
| 설정 | 자동 | 수동 등록 필요 |

---

## 5. 활용 사례

### 포스트 작성 요청

```
@claude Docker Compose 로컬 개발 환경 구성 가이드 포스트를 작성해줘.
Spring Boot + PostgreSQL + Redis 조합으로.
```

### 버그 수정 요청

```
@claude 이 파일의 타입 에러 수정해줘.
src/utils/date.ts
```

### 문서 개선 요청

```
@claude README에 설치 방법과 사용법 섹션 추가해줘.
```

---

## 6. 주의사항 및 트러블슈팅

### 셸 이스케이프 문제

코멘트에 한글이나 특수문자가 포함되면 셸에서 명령어로 해석되는 문제가 발생할 수 있다. `COMMENT_BODY`를 `env`로 전달하면 해결된다.

```yaml
# 잘못된 방법 (셸 이스케이프 문제 발생)
COMMENT_BODY="${{ github.event.comment.body }}"

# 올바른 방법 (env로 안전하게 전달)
env:
  COMMENT_BODY: ${{ github.event.comment.body }}
```

### 파일 수정 권한

CI 환경에서 Claude가 파일을 수정하려면 `--dangerously-skip-permissions` 플래그가 필요하다. 이 플래그 없이는 권한 승인 프롬프트에서 대기하다가 타임아웃된다.

### PR 브랜치에서 작업하기

PR 코멘트에서 `@claude`를 호출하면 main 브랜치가 아닌 PR 브랜치에서 작업해야 한다. `actions/checkout`의 `ref` 옵션으로 PR 브랜치를 지정한다.

```yaml
ref: ${{ github.event.issue.pull_request && format('refs/pull/{0}/head', github.event.issue.number) || github.ref }}
```

### 비용

Claude API 호출 비용이 발생한다. 복잡한 요청일수록 토큰 사용량이 증가한다. `Credit balance is too low` 에러가 발생하면 [Anthropic Console](https://console.anthropic.com/)에서 크레딧을 충전해야 한다.

---

## 7. 배포 결과

PR을 머지하면 GitHub Pages 배포 워크플로우가 자동 실행된다. 빌드가 완료되면 블로그에 새 포스트가 게시된다.

![블로그에 포스트 게시됨](/dev-notes/images/github-claude-automation/07-blog-main.png)

Claude가 작성한 "OpenFGA와 ReBAC로 구현하는 관계 기반 권한 제어" 포스트가 블로그에 게시됐다. PR 본문에 `Closes #9`가 포함되어 있었기 때문에 머지와 동시에 원본 이슈도 자동으로 닫힌다.

---

## 결과

이슈에서 `@claude`를 멘션하는 것만으로 코드 생성부터 배포까지 전체 파이프라인이 자동화됐다.

```mermaid
flowchart LR
    A[이슈 생성] --> B[Claude 실행] --> C[PR 생성] --> D[리뷰/머지] --> E[배포] --> F[이슈 Close]
```

**장점**
- 반복적인 작업 자동화
- 코드 리뷰 프로세스 유지
- 이슈 트래킹과 자연스럽게 연동
- 머지만 하면 배포까지 완료

**확장 가능성**
- PR 코멘트에서도 `@claude` 멘션으로 코드 수정 요청
- 라벨 기반 자동 할당
- 특정 조건에서만 Claude 실행

이 워크플로우를 기반으로 다양한 자동화를 구축할 수 있다.

---

## 시리즈 마무리

이 시리즈에서 구축한 전체 시스템 구조다.

```mermaid
flowchart TB
    subgraph "Part 1: 블로그 구축"
        H[Hugo + PaperMod] --> G[GitHub Pages]
        G --> D1[deploy.yml]
    end

    subgraph "Part 2: AI 자동화"
        I[GitHub 이슈] --> C[Claude Code]
        C --> PR[Pull Request]
        PR --> D1
    end

    D1 --> B[블로그 배포]
```

👈 [Part 1: Hugo + GitHub Pages 구축](/dev-notes/posts/2024-11-29-hugo-github-pages-auto-blog/)
