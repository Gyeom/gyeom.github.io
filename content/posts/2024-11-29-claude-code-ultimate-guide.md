---
title: "Claude Code 활용법 - 입문부터 고급까지"
date: 2024-11-29
draft: false
tags: ["Claude Code", "AI", "개발도구", "자동화", "MCP", "가이드", "Hugo", "GitHub Pages"]
categories: ["개발환경"]
summary: "Claude Code의 모든 기능을 정리한 완벽 가이드. 기본 설치와 사용법부터 CLAUDE.md 설정, 커스텀 슬래시 명령어, Skills/Agents 자동화, MCP 서버 연동, GitHub Actions 통합까지 실전 예제와 함께 다룬다."
---

## 개요

Claude Code는 Anthropic의 공식 CLI 기반 AI 코딩 어시스턴트다. 터미널에서 직접 코드를 읽고, 수정하고, 명령어를 실행한다.

이 가이드에서는 기본 사용법부터 실전 프로젝트에 적용하는 방법까지 모두 다룬다. 마지막에는 Hugo 블로그에 모든 기능을 적용한 Best Practice 사례를 소개한다.

**이 가이드에서 다루는 내용**
- 기본 사용법과 단축키
- 프로젝트 설정 (CLAUDE.md, settings.json)
- 커스텀 슬래시 명령어
- Skills와 Agents
- Hooks로 자동화
- MCP 서버 연동
- GitHub 연동
- 실전 적용 사례

---

## Part 1: 기본 사용법

### 1. 설치 및 시작

```bash
# npm으로 설치
npm install -g @anthropic-ai/claude-code

# 또는 brew (macOS)
brew install claude-code
```

### 첫 실행

```bash
claude              # 대화형 모드
claude "할 일"      # 단일 작업 실행
claude -p "질문"    # 결과만 출력 (파이프용)
```

### 세션 관리

```bash
claude -c           # 최근 세션 이어서
claude -r <id>      # 특정 세션 복원
```

---

### 2. 필수 단축키

| 단축키 | 기능 |
|--------|------|
| `Tab` | Extended Thinking 토글 (심화 분석) |
| `Shift+Tab` | Plan 모드 토글 (읽기 전용 분석) |
| `Ctrl+C` | 현재 작업 취소 |
| `Ctrl+L` | 화면 초기화 |
| `Ctrl+R` | 이전 명령어 검색 |
| `Ctrl+B` | 백그라운드에서 명령 실행 |
| `Esc` 2회 | 코드 변경 되돌리기 (/rewind) |
| `#` | 메모리에 빠르게 추가 |
| `@` | 파일 경로 자동완성 |
| `!` | Bash 명령어 직접 실행 |

```bash
/vim    # Vim 키바인딩 활성화
```

---

### 3. 모델 선택

| 모델 | 용도 |
|------|------|
| `sonnet` | 일반 코딩 (기본값, 빠름) |
| `opus` | 복잡한 분석 (느리지만 강력) |
| `haiku` | 간단한 작업 (가장 빠름) |
| `opusplan` | Opus로 계획 → Sonnet으로 실행 |
| `sonnet[1m]` | 100만 토큰 컨텍스트 (대형 프로젝트) |

```bash
claude --model opus   # CLI 옵션
/model sonnet         # 세션 중 변경
```

복잡한 문제는 `Tab`을 눌러 Extended Thinking을 활성화한다.

---

## Part 2: 프로젝트 설정

### 4. 디렉토리 구조 이해

Claude Code는 두 곳에 파일을 저장한다.

**전역 디렉토리 (`~/.claude/`)**

```
~/.claude/
├── CLAUDE.md              # 전역 메모리 (모든 프로젝트 공통)
├── settings.json          # 전역 설정
├── settings.local.json    # 로컬 전용 설정 (API 키 등)
├── history.jsonl          # 대화 히스토리
├── projects/              # 프로젝트별 메타데이터
├── todos/                 # 작업 목록 저장
├── plans/                 # Plan 모드 결과물
├── session-env/           # 세션 환경 정보
├── file-history/          # 파일 변경 이력 (되돌리기용)
└── debug/                 # 디버그 로그
```

**프로젝트 디렉토리 (`.claude/`)**

```
.claude/
├── settings.json          # 프로젝트 설정 (Git 공유)
├── settings.local.json    # 로컬 설정 (Git 무시)
├── commands/              # 커스텀 슬래시 명령어
│   ├── post.md
│   ├── deploy.md
│   └── ...
├── skills/                # 자동 호출 기능
│   ├── auto-proofreader.md
│   └── auto-tagger.md
└── agents/                # 명시적 호출 기능
    ├── proofreader.md
    └── seo-optimizer.md
```

**프로젝트 루트 파일**

| 파일 | 용도 |
|------|------|
| `CLAUDE.md` | 프로젝트 메모리 (빌드 명령어, 코드 스타일 등) |
| `.mcp.json` | MCP 서버 설정 |

**Git에 포함할 것 / 제외할 것**

```gitignore
# .gitignore
.claude/settings.local.json  # API 키, 개인 설정
```

팀과 공유할 설정은 `settings.json`에, 개인 설정은 `settings.local.json`에 분리한다.

### Config.json (선택)

멀티 프로젝트나 복잡한 설정이 필요하면 `.claude/config.json`을 사용한다.

```json
{
  "project": {
    "name": "my-project",
    "description": "프로젝트 설명"
  },
  "phases": {
    "parallelExecution": true,
    "requireCodeReview": true
  },
  "testing": {
    "coverageThreshold": 80
  }
}
```

단순 프로젝트에서는 불필요하다. CLAUDE.md만으로 충분하다.

---

### 5. CLAUDE.md - 프로젝트 메모리

프로젝트 루트에 `CLAUDE.md`를 만들면 Claude가 세션 시작 시 자동으로 읽는다.

```markdown
# My Project

## 빌드 명령어
- Build: `npm run build`
- Test: `npm test`
- Lint: `npm run lint`

## 코드 스타일
- TypeScript 사용
- 함수명은 camelCase
- 컴포넌트는 PascalCase

## 규칙
- console.log 대신 logger 사용
- any 타입 금지
```

### 파일 임포트

`@` 문법으로 다른 파일을 참조한다.

```markdown
## 글쓰기 규칙

@.claude/writing-guide.md
```

최대 5단계까지 재귀적으로 참조한다.

### 메모리 계층

1. `~/.claude/CLAUDE.md` - 전역 (모든 프로젝트)
2. `./CLAUDE.md` - 프로젝트 (팀 공유)
3. 세션 중 `#`으로 추가한 내용

### Memory 외부화 전략

전역과 프로젝트 메모리를 분리하면 효율적이다.

**`~/.claude/CLAUDE.md` (전역)**
```markdown
# 개인 설정

## 선호 스타일
- 코드 주석은 영어로
- 커밋 메시지는 한글로
- 테스트 먼저 작성

## 자주 쓰는 명령어
- Tab: Extended Thinking
- Ctrl+B: 백그라운드 실행
```

**`./CLAUDE.md` (프로젝트)**
```markdown
# 프로젝트명

## 빌드
npm run build

## 규칙
@.claude/coding-standards.md
```

전역에는 개인 워크플로우, 프로젝트에는 팀 규칙을 둔다.

---

### 6. 권한 설정

Claude Code는 **보안**을 위해 기본적으로 도구 사용 전 사용자 승인을 요청한다. 파일 수정, 명령어 실행 등이 의도치 않은 결과를 초래할 수 있기 때문이다.

### 승인 방식 선택

| 방식 | 설명 |
|------|------|
| **Yes** | 이번만 허용 |
| **Always allow** | 세션 동안 같은 유형 자동 허용 |
| **No** | 거부 |

### 자동 승인 설정

매번 승인하기 번거로우면 설정 파일에서 자동 허용할 도구를 지정한다.

`.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm:*)",
      "Bash(docker-compose:*)",
      "Read(src/**)",
      "Write(src/**)"
    ],
    "ask": [
      "Bash(rm:*)",
      "Bash(chmod:*)",
      "Bash(git reset:*)",
      "Bash(git push --force:*)"
    ],
    "deny": [
      "Read(**/.env)"
    ]
  }
}
```

### 권한 3단계

| 권한 | 동작 | 용도 |
|------|------|------|
| **allow** | 자동 승인 | 안전한 반복 작업 |
| **ask** | 실행 전 확인 | 위험하지만 필요한 명령어 |
| **deny** | 차단 | 절대 실행 안 함 |

### 권한 규칙 문법

```
Tool(action:pattern)

- Bash(npm:*)        # npm 관련 모든 명령어 허용
- Bash(git add:*)    # git add로 시작하는 명령어
- Read(src/**)       # src 하위 모든 파일 읽기 허용
- Write(*.md)        # 마크다운 파일만 쓰기 허용
- Edit(content/**)   # content 하위 파일 편집 허용
```

### MCP 도구 권한

MCP 서버의 도구도 개별적으로 권한 설정이 가능하다.

```json
{
  "permissions": {
    "allow": [
      "mcp__github__list_issues",
      "mcp__github__create_issue",
      "mcp__playwright__browser_navigate",
      "mcp__playwright__browser_take_screenshot",
      "mcp__playwright__browser_click",
      "mcp__fetch__fetch"
    ]
  }
}
```

이렇게 설정하면 GitHub 이슈 조회/생성, Playwright 브라우저 조작, URL 내용 가져오기가 자동 승인된다.

### 완전 자동 실행 모드

**주의: 위험한 옵션이다.**

```bash
claude --dangerously-skip-permissions
```

모든 권한 확인을 건너뛴다. 신뢰할 수 있는 환경에서만 사용해야 한다.

### 설정 파일 위치

| 파일 | 범위 |
|------|------|
| `~/.claude/settings.json` | 전역 |
| `.claude/settings.json` | 프로젝트 (Git 공유) |
| `.claude/settings.local.json` | 로컬 (Git 무시) |

---

### 7. 커스텀 슬래시 명령어

`.claude/commands/review.md`:

```markdown
---
description: 코드 리뷰를 수행한다
allowed-tools:
  - Read
  - Grep
argument-hint: <파일경로>
---

$ARGUMENTS 파일의 코드를 리뷰한다.

다음을 확인한다.
- 보안 취약점
- 성능 이슈
- 코드 스타일
```

```bash
/review src/auth.ts
```

### 인자 접근

- `$ARGUMENTS` - 전체 인자
- `$1`, `$2` - 위치별 인자

---

## Part 3: 자동화

### 8. Skills vs Agents

둘 다 특화된 기능을 정의하지만 호출 방식이 다르다.

| 유형 | 호출 방식 | 용도 |
|------|----------|------|
| **Skill** | 자동 (Claude가 판단) | 반복적인 기본 작업 |
| **Agent** | 명시적 (사용자가 호출) | 심층 분석 |

### Skill 예시

`.claude/skills/auto-proofreader.md`:

```markdown
---
name: auto-proofreader
description: 포스트 작성 시 자동으로 기본 문체를 검사한다
allowed-tools:
  - Read
---

포스트 작성 완료 후 자동으로 검사한다.

## 검사 항목

- : 로 끝나는 문장 → 마침표로 수정
- ~입니다, ~합니다 → ~이다, ~한다로 수정
```

### Agent 예시

`.claude/agents/security-reviewer.md`:

```markdown
---
name: security-reviewer
description: 보안 취약점을 찾는다
allowed-tools:
  - Read
  - Grep
system-prompt: |
  당신은 보안 전문가입니다.
  OWASP Top 10을 기준으로 취약점을 찾습니다.
---

보안 관점에서 코드를 분석한다.
```

```bash
security-reviewer 에이전트로 src/auth/ 폴더 검토해줘
```

---

### 9. Subagent 패턴

복잡한 작업을 독립된 컨텍스트에서 병렬로 실행한다.

### 왜 Subagent인가?

- 메인 컨텍스트가 오염되지 않음
- 병렬 실행으로 시간 단축 (30-40% 비용/시간 절감)
- 각 에이전트가 전문 영역에 집중

### 병렬 vs 순차 실행

**중요**: Task를 **같은 응답**에서 호출해야 병렬로 실행된다.

```
# 병렬 실행 (권장) - 동시에 완료
Task 1: proofreader → 문체 검토
Task 2: seo-optimizer → SEO 검토
(두 Task를 하나의 응답에서 호출)

# 순차 실행 - 시간 2배 소요
응답 1: Task(proofreader)
응답 2: Task(seo-optimizer)
```

### 모델별 작업 매핑

Task tool의 `model` 파라미터로 비용과 성능을 최적화한다.

| 모델 | 적합한 작업 | 비용 |
|------|-------------|------|
| `haiku` | 파일 찾기, 태그 검색 | 최저 |
| `sonnet` | 분석, 리뷰 (기본값) | 중간 |
| `opus` | 복잡한 추론, 창작 | 최고 |

단순 검색에 haiku를 쓰면 비용을 크게 줄일 수 있다.

### 사용 예시

```markdown
# /review 명령어에서 병렬 검토

1. 포스트 찾기
2. **Task tool로 병렬 실행**:
   - proofreader 에이전트 (문체 검토)
   - seo-optimizer 에이전트 (SEO 검토)
3. 결과 종합
```

Task tool의 `subagent_type` 파라미터로 에이전트를 지정한다. 여러 Task를 동시에 호출하면 병렬로 실행된다.

---

### 10. Hooks

특정 이벤트에서 자동으로 실행되는 스크립트다.

### 사용 가능한 이벤트

| 이벤트 | 시점 | 용도 |
|--------|------|------|
| `PreToolUse` | 도구 실행 전 | 검증, 차단 |
| `PostToolUse` | 도구 실행 후 | 후처리, 알림 |
| `Notification` | 이벤트 알림 시 | 로깅, 모니터링 |
| `Stop` | Claude 종료 시 | 정리 작업 |
| `SubagentStop` | Subagent 종료 시 | 결과 처리 |
| `PreCompact` | 컨텍스트 압축 전 | 중요 정보 보존 |
| `UserPromptSubmit` | 사용자 입력 후 | 입력 추적 |

### 설정 예시

`.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matchers": ["Edit"],
        "type": "command",
        "command": "prettier --write $FILE_PATH"
      }
    ],
    "PreToolUse": [
      {
        "matchers": ["Bash(git commit:*)"],
        "type": "command",
        "command": "./scripts/check-sensitive.sh"
      },
      {
        "matchers": ["Bash(git push:*)"],
        "type": "command",
        "command": "npm test || exit 2"
      }
    ]
  }
}
```

### Security Hook 예시

커밋 전 민감 정보를 검사하여 실수로 API 키나 토큰이 노출되는 것을 방지한다.

```bash
#!/bin/bash
# scripts/check-sensitive.sh

STAGED=$(git diff --cached --name-only | grep "\.md$")

PATTERNS=(
  'sk-[a-zA-Z0-9]{20,}'           # OpenAI API key
  'ghp_[a-zA-Z0-9]{36}'           # GitHub PAT
  'AKIA[0-9A-Z]{16}'              # AWS Access Key
)

for file in $STAGED; do
  for pattern in "${PATTERNS[@]}"; do
    if grep -qE "$pattern" "$file"; then
      echo "⚠️ 민감 정보 의심: $file" >&2
      exit 2
    fi
  done
done
```

exit 2 반환 시 커밋이 차단된다.

### 반환값

- `0` - 성공
- `2` - 작업 차단
- 기타 - 경고 후 계속 진행

---

## Part 4: 외부 연동

### 10. MCP 서버

Model Context Protocol. 외부 도구와 데이터 소스를 Claude에 연결한다.

### 서버 추가

```bash
# Fetch - URL 내용 가져오기
claude mcp add fetch -- uvx mcp-server-fetch

# GitHub - 저장소 관리
claude mcp add github -- npx -y @modelcontextprotocol/server-github
```

### 관리 명령어

```bash
claude mcp list          # 설치된 서버 목록
claude mcp get <name>    # 상세 정보
claude mcp remove <name> # 제거
```

### 설정 파일 (.mcp.json)

```json
{
  "mcpServers": {
    "fetch": {
      "command": "uvx",
      "args": ["mcp-server-fetch"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### 활용 예시

```bash
"이 URL 내용 요약해서 정리해줘"
"dev-notes 저장소 이슈 확인해줘"
"배포 실패한 Actions 로그 분석해줘"
```

### Playwright MCP - 브라우저 자동화

웹 페이지 스크린샷, UI 테스트, 자동화 작업에 활용한다.

```bash
# 설치
claude mcp add playwright -- npx @anthropic-ai/mcp-server-playwright
```

**주요 도구**

| 도구 | 기능 |
|------|------|
| `browser_navigate` | URL 이동 |
| `browser_snapshot` | 접근성 스냅샷 (상호작용용) |
| `browser_take_screenshot` | 스크린샷 캡처 |
| `browser_click` | 요소 클릭 |
| `browser_type` | 텍스트 입력 |
| `browser_resize` | 뷰포트 크기 조정 |

**스크린샷 캡처 예시**

```bash
"GitHub 이슈 목록 캡처해줘"
"배포된 블로그 메인 페이지 스크린샷 찍어줘"
```

불필요한 공백을 줄이려면 캡처 전 viewport를 조정한다.

```javascript
// viewport 조정 후 캡처
browser_resize({ width: 1200, height: 500 })
browser_take_screenshot({ filename: "issues.png" })

// 특정 요소만 캡처
browser_take_screenshot({
  element: "main content",
  ref: "main",
  filename: "content.png"
})
```

권장 viewport 높이다.
- 목록 페이지: 400-500px
- 상세 페이지: 600-700px
- 전체 페이지: `fullPage: true`

---

### 11. GitHub 연동

이슈나 PR에서 `@claude`를 멘션하면 Claude가 응답한다.

### 워크플로우 설정

`.github/workflows/claude.yml`:

```yaml
name: Claude AI Assistant

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  claude-response:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm install -g @anthropic-ai/claude-code

      - name: Run Claude
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PROMPT=$(echo "${{ github.event.comment.body }}" | sed 's/.*@claude//')
          RESPONSE=$(claude -p "$PROMPT" --max-turns 5)
          gh issue comment ${{ github.event.issue.number }} --body "$RESPONSE"
```

### API 키 설정

```bash
gh secret set ANTHROPIC_API_KEY --repo USERNAME/REPO
```

### 사용 예시

```
@claude 이 버그 수정해줘
@claude 코드 리뷰해줘
@claude README 개선 제안해줘
```

---

### 12. Headless Mode (CI/CD)

대화 없이 명령어를 실행하고 결과만 받는다.

```bash
# 기본 사용
claude -p "README.md 요약해줘"

# 출력 형식 지정
claude -p "버그 분석해줘" --output-format json

# 최대 턴 수 제한 (무한 루프 방지)
claude -p "테스트 실행하고 결과 알려줘" --max-turns 10
```

### CI/CD 활용

```yaml
# GitHub Actions에서
- name: Code Review
  run: |
    RESULT=$(claude -p "PR 변경사항 리뷰해줘" --max-turns 5)
    echo "$RESULT" >> $GITHUB_STEP_SUMMARY
```

### 권장 max-turns

| 작업 | max-turns |
|------|-----------|
| 단순 질문 | 3-5 |
| 코드 리뷰 | 10-15 |
| 버그 수정 | 15-20 |
| 복잡한 리팩토링 | 30+ |

---

## Part 5: 실전 적용 - Hugo 블로그 Best Practice

모든 기능을 Hugo 블로그에 적용한 사례다.

> 📦 **GitHub 저장소**: [Gyeom/dev-notes](https://github.com/Gyeom/dev-notes)
>
> 실제 사용 중인 설정 파일들을 확인할 수 있다.

### 프로젝트 구조

```
dev-notes/
├── CLAUDE.md                    # 프로젝트 메모리
├── .mcp.json                    # MCP 서버 설정
├── .claude/
│   ├── settings.json            # 권한 + Hooks
│   ├── writing-guide.md         # 글쓰기 규칙
│   ├── commands/                # 슬래시 명령어
│   │   ├── post.md
│   │   ├── draft.md
│   │   ├── publish.md
│   │   ├── edit.md
│   │   ├── list.md
│   │   ├── stats.md
│   │   ├── review.md
│   │   ├── deploy.md
│   │   └── preview.md
│   ├── skills/                  # 자동 호출
│   │   ├── auto-proofreader.md
│   │   ├── auto-tagger.md
│   │   └── post-sync.md
│   ├── agents/                  # 명시적 호출
│   │   ├── proofreader.md
│   │   ├── seo-optimizer.md
│   │   └── post-reviewer.md
│   └── knowledge/               # 참조 데이터
│       └── post-index.md
├── .github/workflows/
│   ├── deploy.yml               # Hugo 자동 배포
│   └── claude.yml               # @claude 멘션
├── content/posts/
├── scripts/
└── themes/PaperMod/
```

### CLAUDE.md

```markdown
# Dev Notes

Hugo 기반 개발 블로그. 마크다운 작성 후 push하면 자동 배포된다.

## 빌드 및 배포

hugo --gc --minify        # 빌드
hugo server -D            # 로컬 미리보기
git push                  # 배포

## 슬래시 명령어

| 명령어 | 설명 |
|--------|------|
| /post | 새 포스트 작성 및 배포 |
| /draft | 초안 작성 |
| /list | 최근 포스트 목록 |
| /review | 문체/SEO 검토 |

## 글쓰기 규칙

@.claude/writing-guide.md
```

### settings.json (권한 + Hooks)

실제 프로젝트에서 사용 중인 설정이다.

```json
{
  "permissions": {
    "allow": [
      "Bash(hugo:*)",
      "Bash(hugo server:*)",
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(git push:*)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git pull:*)",
      "Bash(./scripts/*)",
      "Bash(gh issue create:*)",
      "Bash(gh issue list:*)",
      "Bash(gh pr list:*)",
      "Bash(gh pr merge:*)",
      "Read(content/**)",
      "Write(content/**)",
      "Edit(content/**)",
      "mcp__github__list_issues",
      "mcp__github__create_issue",
      "mcp__playwright__browser_navigate",
      "mcp__playwright__browser_take_screenshot",
      "mcp__playwright__browser_click",
      "mcp__playwright__browser_wait_for",
      "mcp__playwright__browser_close",
      "mcp__playwright__browser_resize",
      "mcp__fetch__fetch"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git push:*)",
        "hooks": [
          {
            "type": "command",
            "command": "hugo --gc --minify > /dev/null 2>&1 || exit 2"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write(content/posts/*)",
        "hooks": [
          {
            "type": "command",
            "command": "echo '새 포스트가 작성되었습니다. /preview로 미리보기하거나 /deploy로 배포하세요.'"
          }
        ]
      }
    ]
  }
}
```

**권한 설정 포인트**
- Hugo, Git, GitHub CLI 명령어 자동 허용
- 포스트 파일(`content/**`) 읽기/쓰기/편집 자동 허용
- MCP 도구(GitHub, Playwright, Fetch) 자동 허용
- 승인 없이 블로그 작업에 집중할 수 있다

**Hook 설정 포인트**
- `PreToolUse`: `git push` 전 Hugo 빌드 테스트 → 실패 시 push 차단
- `PostToolUse`: 포스트 작성 후 안내 메시지 출력

### Knowledge Base

`.claude/knowledge/` 디렉토리에 참조 데이터를 저장한다.

```
.claude/knowledge/
└── post-index.md    # 포스트 메타데이터 인덱스
```

**post-index.md 예시**

```markdown
## 전체 포스트 (5개)

| 파일 | 제목 | 태그 |
|------|------|------|
| claude-code-anatomy.md | Claude Code 해부 | Claude Code, AI |
| github-claude-automation.md | 이슈 기반 자동 포스팅 | GitHub Actions |

## 태그별 분류

### Claude Code
- claude-code-anatomy.md
- claude-code-ultimate-guide.md
```

`/index` 명령어로 인덱스를 갱신한다. `post-sync` Skill이 이 인덱스를 참조하여 관련 포스트를 찾는다.

### 자동 인덱싱

포스트 작성/수정 시 Hook으로 인덱스가 자동 갱신된다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write(content/posts/*)",
        "hooks": [
          { "command": "python3 scripts/update-index.py" }
        ]
      }
    ]
  }
}
```

### 검색 고도화 로드맵

포스트가 많아지면 검색 방식을 단계적으로 업그레이드한다.

```
.claude/knowledge/
├── post-index.md   # 마크다운 (사람용)
├── posts.json      # JSON (기계용)
└── schema.md       # 벡터 검색 스키마
```

| 단계 | 포스트 수 | 검색 방식 | 토큰 절감 |
|------|----------|----------|----------|
| Phase 1 | ~50개 | 키워드/태그 검색 | 기본 |
| Phase 2 | 50~100개 | SQLite FTS5 전문 검색 | ~20% |
| Phase 3 | 100개+ | 벡터 임베딩 (sqlite-vec) | ~40% |

Phase 3에서는 Semantic Search MCP를 도입하여 유사 컨텐츠 검색을 지원한다. 스키마는 `.claude/knowledge/schema.md`에 미리 정의해 둔다.

### 역할 분담

| 유형 | 이름 | 역할 |
|------|------|------|
| Skill | auto-proofreader | 포스트 작성 시 자동 문체 검사 |
| Skill | auto-tagger | 내용 기반 태그 자동 추천 |
| Skill | post-sync | 작업 완료 후 관련 포스트 업데이트 확인 |
| Agent | proofreader | 심층 문체 검토 |
| Agent | seo-optimizer | SEO 분석 |
| Agent | post-reviewer | 종합 검토 (문체 + SEO + 코드) |

### 사용 예시

```bash
# 포스트 작성
"이 내용 정리해서 포스팅해줘"

# 슬래시 명령어
/post "제목" 태그1,태그2
/list
/stats

# 에이전트 호출
"proofreader로 최근 포스트 검토해줘"

# MCP 활용
"이 URL 내용 요약해서 TIL로 올려줘"

# GitHub에서
@claude 이 이슈 해결해줘
```

### 적용 결과

| 기능 | 효과 |
|------|------|
| CLAUDE.md | 프로젝트 컨텍스트 자동 로드 |
| Skills | 포스트 작성 시 자동 검사 |
| Hooks | 빌드 실패 방지 |
| MCP | 외부 URL/저장소 연동 |
| GitHub App | 이슈에서 바로 작업 |

---

## CLI 옵션 총정리

### 실행 모드

```bash
claude                    # 대화형
claude "작업"             # 단일 실행
claude -p "질문"          # 출력만 (파이프용)
claude -c                 # 최근 세션 계속
claude -r <id>            # 세션 복원
```

### 모델 및 도구

```bash
--model sonnet              # 모델 지정
--allowedTools "Bash,Read"  # 허용 도구
--disallowedTools "Write"   # 제한 도구
--max-turns 10              # 최대 턴 수
```

### 출력 포맷

```bash
--output-format text        # 텍스트 (기본)
--output-format json        # JSON
--output-format stream-json # 스트림 JSON
```

---

## 자주 쓰는 슬래시 명령어

| 명령어 | 기능 |
|--------|------|
| `/help` | 도움말 |
| `/model <name>` | 모델 변경 |
| `/compact` | 대화 압축 |
| `/clear` | 히스토리 초기화 |
| `/memory` | CLAUDE.md 편집 |
| `/vim` | Vim 모드 |
| `/rewind` | 변경 되돌리기 |
| `/agents` | 에이전트 관리 |
| `/hooks` | Hook 관리 |
| `/status` | 현재 상태 확인 |

---

## 성능 최적화 팁

### 1. 모델 선택 전략

| 상황 | 추천 모델 | 이유 |
|------|----------|------|
| 간단한 질문, 파일 찾기 | `haiku` | 빠르고 저렴 |
| 일반 코딩 | `sonnet` | 균형 잡힌 성능 |
| 복잡한 아키텍처 설계 | `opus` | 깊은 추론 |
| 대형 코드베이스 분석 | `sonnet[1m]` | 100만 토큰 컨텍스트 |

**실전 팁**: 대부분 sonnet으로 충분하다. opus는 "왜 이렇게 설계해야 하는가" 같은 깊은 고민이 필요할 때만 사용한다.

### 2. max-turns 설정

자동화 스크립트나 CI에서 무한 루프를 방지한다.

```bash
claude -p "버그 수정해줘" --max-turns 10
```

| 작업 유형 | 권장 max-turns |
|----------|---------------|
| 단순 질문 | 3-5 |
| 버그 수정 | 10-15 |
| 기능 구현 | 20-30 |
| 복잡한 리팩토링 | 50+ |

너무 낮으면 작업 중간에 멈추고, 너무 높으면 삽질을 계속한다.

### 3. 컨텍스트 관리

긴 대화는 토큰을 소모하고 응답이 느려진다.

```bash
/compact    # 대화 압축 (핵심만 유지)
/clear      # 히스토리 초기화 (새 주제 시작할 때)
```

**언제 /compact를 쓰는가?**
- 같은 주제를 계속 다룰 때
- "컨텍스트가 길다"는 경고가 뜰 때

**언제 /clear를 쓰는가?**
- 완전히 다른 작업을 시작할 때
- 이전 대화가 방해될 때

### 4. 병렬 처리

```bash
Ctrl+B    # 현재 명령을 백그라운드로
```

테스트나 빌드를 돌리면서 다른 파일 작업을 계속할 수 있다.

```bash
# 예시: 테스트 돌리면서 다른 작업
"npm test 백그라운드로 실행해줘"
"그 동안 README 업데이트해줘"
```

### 5. 효과적인 프롬프트

**구체적으로 요청한다**

```
❌ "버그 수정해줘"
✅ "src/auth.ts:45에서 발생하는 TypeError 수정해줘"

❌ "성능 개선해줘"
✅ "getUsers 함수가 1000건 이상에서 느려지는데, 페이지네이션 추가해줘"
```

**파일 경로를 명시한다**

```
❌ "로그인 함수 수정해줘"
✅ "@src/services/auth.ts의 login 함수 수정해줘"
```

`@`를 누르면 파일 경로 자동완성이 된다.

### 6. Extended Thinking 활용

복잡한 문제는 `Tab`을 눌러 Extended Thinking을 활성화한다.

```
# 이런 상황에서 유용하다
- 여러 파일에 걸친 리팩토링
- 버그의 근본 원인 분석
- 아키텍처 설계 결정
- "왜 안 되지?" 싶을 때
```

### 7. Plan Mode

복잡한 작업 전 계획을 먼저 세운다.

```bash
Shift+Tab    # Plan 모드 토글
```

### 워크플로우

```
1. Plan 모드 진입 (Shift+Tab)
2. Claude가 계획 제시
3. 사용자가 검토/수정
4. 승인 후 실행
```

### 언제 사용하는가?

- 대규모 리팩토링
- 여러 파일에 걸친 기능 추가
- 아키텍처 변경
- "이거 어떻게 하지?" 싶을 때

Plan 모드에서는 코드를 수정하지 않고 계획만 세운다. 사용자 승인 후 실행되므로 실수를 방지할 수 있다.

### 8. 세션 재사용

```bash
claude -c              # 마지막 세션 이어서
claude -r <session-id> # 특정 세션 복원
```

같은 프로젝트에서 작업을 이어갈 때 컨텍스트를 다시 설명할 필요 없다.

---

## Troubleshooting

### Hook 실행 실패

```
Hook이 exit 2를 반환했습니다
```

→ `PreToolUse` Hook에서 스크립트가 실패함. Hook에 설정된 명령어를 수동 실행하여 오류를 확인한다.

### MCP 연결 오류

```
MCP server 'xxx' failed to start
```

→ MCP 서버 설정 확인. `.mcp.json`의 command 경로와 환경변수가 올바른지 확인한다.

### 권한 오류

```
Permission denied for tool: Bash(rm -rf:*)
```

→ `ask` 또는 `deny` 권한에 해당 명령어가 있다. 정말 필요하면 `settings.json`에서 `allow`로 이동한다.

### 세션 복원 실패

```
Session not found: <id>
```

→ 세션 히스토리가 만료됨. `claude -c`로 가장 최근 세션을 시도한다.

---

## 마무리

Claude Code는 단순한 코드 생성 도구가 아니다.

- **CLAUDE.md**로 프로젝트 컨텍스트를 공유
- **Skills/Agents**로 반복 작업 자동화
- **Hooks**로 실수 방지
- **MCP**로 외부 시스템 연동
- **GitHub App**으로 이슈/PR에서 바로 작업

이 기능들을 조합하면 진짜 AI 페어 프로그래머가 된다.

**핵심 파일**
- `CLAUDE.md` - 프로젝트 메모리
- `.claude/settings.json` - 권한 + Hooks
- `.claude/commands/` - 커스텀 명령어
- `.claude/skills/` - 자동 호출 기능 (post-sync 등)
- `.claude/agents/` - 심층 분석 기능 (post-reviewer 등)
- `.claude/knowledge/` - 참조 데이터 (post-index 등)
- `.mcp.json` - MCP 서버 설정

**필수 단축키**
- `Tab` - Extended Thinking
- `Ctrl+B` - 백그라운드 실행
- `Esc` 2회 - 되돌리기
- `#` - 메모리 추가
