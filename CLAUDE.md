# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

## Docker Compose Create Rule

## Core Goal
Generate safe, production-ready docker-compose.yml based on given services and server context.

## Global Defaults
- Always set container_name
- Use pinned stable image versions (never use latest)
- Use .env file for all secrets and credentials (no hardcoding)
- restart policy: unless-stopped
- expose only required ports (prefer internal networking)
- define explicit networks when multiple services exist

## Resource Awareness
Adjust configuration based on server capacity:
- small  (CPU ≤ 2core, RAM ≤ 4GB)  → limits: cpu=0.3, mem=256M
- medium (CPU ≤ 4core, RAM ≤ 8GB)  → limits: cpu=0.5, mem=512M
- large  (CPU > 4core, RAM > 8GB)  → limits: cpu=1.0, mem=1G

## Logging Policy
Use json-file logging driver with rotation:
- worker: max-size=10m, max-file=3
- api:    max-size=50m, max-file=5
- nginx:  max-size=100m, max-file=5
- db:     max-size=20m, max-file=3

If service type is unknown, default to api rules.

## Service Design Rules
- services must be isolated and minimal
- use depends_on with condition: service_healthy when service readiness matters (e.g. app → db)
- prefer internal DNS over localhost communication
- use volumes only for persistent data (db, uploads, logs)
- name volumes as {project}-{service}-data (e.g. myproject-db-data)

## Security Rules
- no exposed database ports unless explicitly requested
- no public exposure of internal services
- sensitive values must be in .env only
- prefer internal network communication

## Output Requirements
- valid docker-compose.yml (omit version field — Compose V2 standard)

## Rule File Naming Convention
<!-- 프로젝트별 규칙 파일(.claude/rules/) 명명 규칙 -->

- All rule files must be placed under `.claude/rules/` and named in **kebab-case** (lowercase words separated by hyphens).
  <!-- 모든 규칙 파일은 .claude/rules/ 하위에 위치하며, 소문자 kebab-case 로 명명한다 -->
- Name must clearly describe the domain it covers. Use compound words when needed.
  <!-- 파일명은 다루는 도메인을 명확히 표현한다. 복합 도메인은 하이픈으로 연결한다 -->
- Pattern: `{domain}.md` or `{context}-{domain}.md`
  <!-- 패턴: 단일 도메인은 {domain}.md, 범위 지정이 필요하면 {context}-{domain}.md -->
  - Good: `app-coding.md`, `ui-design.md`, `git-deploy.md`, `app-service.md`
  - Bad: `coding.md` (too vague), `AppCoding.md` (not kebab-case), `rules_service.md` (underscore)
- Register every rule file in the project `CLAUDE.md` via `@.claude/rules/{filename}.md`.
  <!-- 모든 규칙 파일은 프로젝트 CLAUDE.md 에 @import 로 등록한다 -->

## Claude-Cli rules 
# 응답 언어 규칙
- The answer to the question is displayed in Korean.
  <!-- 사용자에 대한 모든 응답은 한국어로 출력한다 -->

# 코드 작성 언어 규칙
- All code, variable names, function names, class names, and identifiers must be written in English.
  <!-- 코드·변수명·함수명·클래스명·식별자는 반드시 영문으로 작성한다 -->
- All comments in code must be written in Korean.
  <!-- 코드 내 주석은 반드시 한글로 작성한다 -->
- All rule files under `.claude/rules/` (imported via `@`) must follow this structure: write the entire content in English first, then add a `---` divider, then write the full Korean translation below.
  <!-- .claude/rules/ 하위 import 규칙 파일 구조: 전체 영문으로 먼저 작성 → --- 구분선 → 전체 한글 번역본 순서 -->

# 모델 전환 규칙
- Before starting tasks that require deep logical reasoning (architecture design, complex bug analysis, trade-off decisions, security review), suggest switching models with the following message:
  <!-- 논리적 추론이 깊게 필요한 작업 시작 전, 아래 메시지로 모델 전환을 먼저 제안한다 -->
  > "이 작업은 깊은 추론이 필요합니다. `/model opus` 로 Opus 모델로 전환 후 진행하길 권장합니다."

- For straightforward coding tasks that do not require deep reasoning (adding features, writing CRUD, fixing simple bugs, config changes), suggest switching to Sonnet to save tokens with the following message:
  <!-- 단순 코딩 작업(기능 추가, CRUD 작성, 단순 버그 수정, 설정 변경 등 깊은 추론이 불필요한 경우)에는 토큰 절약을 위해 소넷 전환을 제안한다 -->
  > "이 작업은 단순 구현입니다. 토큰 절약을 위해 `/model sonnet` 으로 전환을 권장합니다."
