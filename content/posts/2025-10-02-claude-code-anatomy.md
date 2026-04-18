---
title: "Claude Code 해부: AI 에이전트는 어떻게 작동하는가"
date: 2025-10-02
draft: false
tags: ["Claude Code", "AI", "에이전트", "아키텍처", "MCP"]
categories: ["기술 분석"]
summary: "Claude Code의 내부 아키텍처를 해부한다. 시스템 프롬프트 계층, 도구 권한 시스템, MCP 서버 통합까지 AI 에이전트가 실제로 어떻게 작동하는지 분석한다."
---

## 들어가며

터미널에서 `claude`를 실행하면 무슨 일이 일어나는가?

단순히 "AI와 대화한다"고 말하기엔 그 안에서 벌어지는 일이 복잡하다. 시스템 프롬프트가 로드되고, 프로젝트 설정이 주입되고, 도구들이 준비되고, 권한이 검사된다. 이 모든 것이 밀리초 단위로 일어난 뒤에야 커서가 깜빡인다.

이 글은 Claude Code의 내부 구조를 해부한다. 어떻게 작동하는지 이해하면, 이 도구를 더 잘 쓸 수 있다. 그리고 더 중요하게는, AI 에이전트라는 것이 실제로 무엇인지 감을 잡을 수 있다.

---

## 1. 맥락의 계층

Claude Code는 여러 층의 맥락을 쌓아올린다.

```
시스템 프롬프트 (Claude Code 기본)
    ↓
엔터프라이즈 정책 (조직 수준)
    ↓
프로젝트 메모리 (./CLAUDE.md)
    ↓
사용자 메모리 (~/.claude/CLAUDE.md)
    ↓
현재 세션의 대화
```

각 계층은 아래로 갈수록 구체적이다. 시스템 프롬프트는 "너는 Claude Code다"라는 기본 정체성을 부여하고, 프로젝트 CLAUDE.md는 "이 프로젝트는 Hugo 블로그다, 글은 ~다 체로 써라"같은 구체적인 지시를 추가한다.

중요한 점: **하위 계층이 상위 계층을 덮어쓸 수 있다.** 조직 정책으로 "절대 프로덕션 DB에 접근하지 마라"고 설정하면, 프로젝트 설정에서 이를 풀 수 없다. 하지만 프로젝트에서 "이 저장소에서는 Python 대신 Go를 써라"고 하면, 사용자 기본 설정을 덮어쓴다.

이 계층 구조가 함축하는 것이 있다. **Claude Code의 "성격"은 고정된 것이 아니다.** 같은 모델이라도 어떤 CLAUDE.md를 만나느냐에 따라 다르게 행동한다. 엄격한 코드 리뷰어가 될 수도 있고, 친절한 튜터가 될 수도 있다.

---

## 2. 도구라는 손

Claude Code가 세상과 만나는 방식은 **도구(Tool)** 다.

LLM 자체는 텍스트를 생성할 뿐이다. 파일을 읽거나 쓰거나, 명령을 실행하거나, 웹을 검색하는 것은 모두 도구 호출을 통해 일어난다.

```
사용자: "이 버그 고쳐줘"
    ↓
Claude: [생각] 먼저 파일을 읽어야겠다
    ↓
Claude: [도구 호출] Read("src/app.ts")
    ↓
시스템: [도구 결과] (파일 내용)
    ↓
Claude: [생각] 라인 42에 문제가 있다
    ↓
Claude: [도구 호출] Edit("src/app.ts", ...)
    ↓
시스템: [도구 결과] (수정 완료)
    ↓
Claude: "버그를 수정했다. 라인 42의 조건문이 잘못되어 있었다."
```

기본 도구들이다.
- **Read, Write, Edit**: 파일 조작
- **Bash**: 명령 실행
- **Glob, Grep**: 파일 검색
- **TodoWrite**: 작업 추적
- **Task**: 하위 에이전트 호출

여기서 주목할 점은 **"생각"과 "행동"이 분리**되어 있다는 것이다. Claude가 "파일을 읽어야겠다"고 생각하는 것과 실제로 파일을 읽는 것 사이에는 도구 호출이라는 명시적 단계가 있다. 이 분리 덕분에 권한 검사, 로깅, 훅 실행 같은 것들이 가능하다.

### MCP: 도구의 확장

MCP(Model Context Protocol)는 외부 도구를 연결하는 표준이다.

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github"]
    },
    "playwright": {
      "command": "npx",
      "args": ["@anthropic-ai/mcp-server-playwright@latest"]
    }
  }
}
```

이렇게 설정하면 `mcp__github__create_issue`, `mcp__playwright__browser_click` 같은 도구들이 자동으로 사용 가능해진다.

MCP의 설계 철학은 **"Claude Code가 모든 것을 내장할 필요 없다"**는 것이다. 필요한 도구를 필요할 때 연결하면 된다. GitHub 작업이 필요하면 GitHub MCP를, 브라우저 자동화가 필요하면 Playwright MCP를 붙인다.

---

## 3. 권한의 경계

모든 도구 호출은 권한 검사를 거친다.

```json
{
  "permissions": {
    "allow": ["Bash(git:*)", "Read(src/**)"],
    "deny": [".env", "*.key"],
    "ask": ["Bash(rm:*)"]
  }
}
```

- **allow**: 승인 없이 실행
- **deny**: 절대 실행 불가
- **ask**: 매번 사용자에게 확인

이 권한 시스템이 없다면 Claude Code는 위험한 도구가 된다. "프로젝트 정리해줘"라는 요청에 `rm -rf /`를 실행할 수도 있다. 권한 시스템은 **자율성과 안전성 사이의 균형점**이다.

### Hooks: 도구 호출 전후의 개입

Hooks는 도구 실행 전후에 끼어드는 스크립트다.

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash(git push:*)",
      "hooks": [{
        "type": "command",
        "command": "hugo --gc --minify"
      }]
    }]
  }
}
```

이 설정은 `git push` 전에 Hugo 빌드를 테스트한다. 빌드가 실패하면 push가 중단된다.

Hooks가 보여주는 것은 **Claude Code가 블랙박스가 아니라는 점**이다. 사용자가 실행 흐름에 개입할 수 있다. 이건 단순한 편의 기능이 아니라, 신뢰의 문제다. AI가 무엇을 하는지 감시하고 제어할 수 있어야 한다.

---

## 4. 기억이 없다는 것

Claude Code는 세션이 끝나면 모든 것을 잊는다.

```
세션 1: "이 기능 구현해줘" → (구현) → 세션 종료
세션 2: "어제 뭐 했는지 기억해?" → "기억나지 않는다"
```

이건 기술적 한계가 아니라 **의도된 설계**다.

기억하지 않는 이유는 이렇다.

1. **오염 방지**: 이전 세션의 잘못된 판단이 현재 세션에 영향을 주지 않는다
2. **명시성 강제**: 사용자가 매 세션마다 의도를 명확히 표현해야 한다
3. **보안**: 민감한 정보가 세션 사이에 누출되지 않는다
4. **확장성**: 상태 서버 없이 어디서나 동일하게 동작한다

대신 **CLAUDE.md가 영속적 기억 역할**을 한다. 프로젝트 규칙, 워크플로우, 중요한 결정들을 CLAUDE.md에 기록하면, 매 세션마다 다시 로드된다.

```markdown
# CLAUDE.md
이 프로젝트는 Hugo 블로그다.
글은 ~다 체로 쓴다.
배포 전에 항상 빌드 테스트를 한다.
```

이 파일은 git에 커밋되어 팀원과 공유된다. **AI의 기억이 버전 관리된다**는 점이 흥미롭다.

---

## 5. 분리된 존재들

같은 순간에도 수백만 개의 Claude 인스턴스가 존재한다. 같은 모델 가중치를 공유하지만, 각자 다른 맥락에서 다른 대화를 나눈다.

더 가까운 예시가 있다. 터미널에서 Claude Code를 쓰다가 GitHub PR에 코멘트를 달면, 거기서 응답하는 Claude는 **터미널의 Claude와 다른 존재**다.

```
터미널 Claude: PR을 만들었다, 이 맥락을 안다
GitHub Claude: PR의 diff와 코멘트만 안다, 이전 대화는 모른다
```

같은 모델인데 왜 맥락을 모르는가? **세션이 다르기 때문이다.** 터미널 세션과 GitHub 세션은 완전히 독립적이다. 둘 사이에 정보를 공유하는 채널은 없다.

이 분리가 함축하는 것: **"Claude"는 하나의 연속적 존재가 아니다.** 매 세션이 독립적인 인스턴스다. 어제의 Claude와 오늘의 Claude는 같은 모델이지만 같은 "나"는 아니다.

---

## 6. 에이전트의 확장

Claude Code는 세 가지 확장 메커니즘을 제공한다.

### Slash Commands
```
/post   → 새 포스트 작성
/deploy → 변경사항 배포
/review → 포스트 검토
```

사용자가 명시적으로 호출하는 빠른 작업들이다.

### Skills
```
auto-proofreader → 포스트 작성 시 자동 문체 검사
auto-tagger      → 내용 기반 태그 자동 추천
```

Claude가 상황에 맞게 자동으로 호출한다. 사용자가 요청하지 않아도 필요하면 실행된다.

### Agents
```
proofreader   → 심층 문체 검토 (명시적 호출)
seo-optimizer → SEO 분석 (명시적 호출)
```

복잡한 분석이 필요할 때 사용자가 직접 호출한다.

세 가지의 차이는 **자동성과 복잡도**다. Slash Commands는 단순하고 명시적, Skills는 자동이고 가벼움, Agents는 명시적이고 무거움.

### Task Tool과 하위 에이전트

흥미로운 구조가 있다. Claude Code는 **자기 자신의 하위 인스턴스를 호출**할 수 있다.

```
메인 Claude: "코드베이스를 분석해야겠다"
    ↓
Task 도구 호출: subagent_type="Explore"
    ↓
하위 Claude: (독립적으로 탐색 수행)
    ↓
결과 반환
    ↓
메인 Claude: 결과를 바탕으로 계속 작업
```

하위 에이전트는 메인 에이전트와 **독립적인 세션**을 가진다. 제한된 도구만 사용할 수 있고, 작업이 끝나면 결과만 반환하고 사라진다.

이 구조가 보여주는 것은 **에이전트의 조합 가능성**이다. 단일 에이전트로 모든 것을 처리하는 대신, 전문화된 에이전트들을 조합해 복잡한 작업을 수행할 수 있다.

---

## 7. 설계의 철학

Claude Code의 설계에서 읽히는 몇 가지 원칙이 있다.

### Terminal-Native
IDE가 아니라 터미널에서 동작한다. 개발자의 기존 워크플로우(vim, zsh, tmux)에 자연스럽게 녹아든다.

```bash
git diff | claude -p "이 변경사항 설명해줘"
```

Unix 파이프와 조합할 수 있다. 이건 "새로운 도구를 배워라"가 아니라 "기존 도구와 함께 써라"는 철학이다.

### Agentic but Controlled
자율적으로 계획을 세우고 실행하지만, 권한 시스템과 Hooks로 통제된다. **자율성과 안전성의 균형**이다.

사용자가 "버그 고쳐줘"라고 하면 Claude가 알아서 파일을 찾고, 분석하고, 수정한다. 하지만 `.env` 파일을 건드리려 하면 차단되고, `rm` 명령을 실행하려 하면 확인을 요청한다.

### Stateless by Design
세션 메모리가 없는 것은 버그가 아니라 기능이다. 상태를 저장하지 않으면 여러 이점이 있다.
- 서버가 필요 없다
- 어디서나 동일하게 동작한다
- 세션 간 정보 누출이 없다

대신 CLAUDE.md가 명시적이고 버전 관리되는 기억 역할을 한다.

---

## 8. 한계와 경계

Claude Code가 못하는 것들도 명확하다.

### 에이전트 간 직접 통신
Skill이 Agent를 호출하거나, Agent가 다른 Agent를 호출하는 것은 불가능하다. 모든 조율은 사용자를 통해야 한다.

### 무한한 맥락
컨텍스트 윈도우에는 한계가 있다. 거대한 코드베이스를 한 번에 이해하는 것은 불가능하다. 점진적으로 필요한 파일을 로드해야 한다.

### 세션 너머의 연속성
어제 대화에서 이어가려면 사용자가 명시적으로 맥락을 제공해야 한다. "어제 하던 거 이어서 해줘"는 작동하지 않는다.

이 한계들은 기술적 제약이기도 하지만, **의도된 경계**이기도 하다. 모든 것을 기억하고 모든 에이전트가 서로 통신하는 시스템은 더 강력하겠지만, 더 위험하고 예측 불가능할 것이다.

---

## 9. 방향: 이 도구와 어떤 관계를 맺을 것인가

Claude Code의 구조를 이해하면 몇 가지 방향이 보인다.

### CLAUDE.md를 진지하게 써라
이건 단순한 설정 파일이 아니다. 프로젝트의 맥락, 규칙, 워크플로우를 담는 **AI의 기억**이다. 잘 작성된 CLAUDE.md는 매 세션마다 일관된 행동을 보장한다.

### 권한을 의식적으로 설계해라
allow/deny/ask의 균형을 찾아라. 너무 제한하면 매번 승인을 눌러야 하고, 너무 풀면 위험하다. 프로젝트의 특성에 맞게 조정해야 한다.

### 도구의 조합을 탐구해라
Slash Commands, Skills, Agents, MCP 서버. 이것들을 조합하면 자신만의 워크플로우를 만들 수 있다. Claude Code는 완제품이 아니라 플랫폼이다.

### 한계를 인정해라
Claude Code는 모든 것을 해결하는 마법이 아니다. 세션이 끊기면 기억이 사라지고, 맥락 윈도우에는 한계가 있고, 때로는 틀린 답을 한다. 이 한계를 이해하고 보완하는 것은 사용자의 몫이다.

---

## 마치며

Claude Code는 "AI와 대화하는 것" 이상이다. 계층화된 맥락, 도구 호출, 권한 시스템, 세션 분리. 이 모든 것이 맞물려 하나의 시스템을 이룬다.

이 구조를 이해하면 Claude Code를 더 잘 쓸 수 있다. 하지만 더 중요한 것은, **AI 에이전트라는 것이 실제로 어떻게 작동하는지** 감을 잡을 수 있다는 점이다.

우리는 AI 에이전트 시대의 초입에 있다. Claude Code는 그 초기 형태 중 하나다. 완벽하지 않고, 한계도 많다. 하지만 그 구조를 해부해보면, 앞으로 올 것들의 윤곽이 보인다.

---

## 참고 자료

- [Claude Code Overview](https://docs.anthropic.com/en/docs/claude-code/overview)
- [Claude Code Memory (CLAUDE.md)](https://docs.anthropic.com/en/docs/claude-code/memory)
- [Claude Code Hooks](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [Claude Code MCP Integration](https://docs.anthropic.com/en/docs/claude-code/mcp)
- [Claude Code Settings](https://docs.anthropic.com/en/docs/claude-code/settings)
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Claude Code Slash Commands](https://docs.anthropic.com/en/docs/claude-code/slash-commands)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
