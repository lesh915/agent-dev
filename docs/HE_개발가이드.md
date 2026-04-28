# HE-AMS 개발 가이드
## 하네스 엔지니어링 기반 에이전트 관리 시스템 — 기술 스펙 & 아키텍처

**버전:** 0.1.0  
**작성일:** 2026-04-29  
**참조 문서:** [HE_PRD.md](./HE_PRD.md)

---

## 목차

1. [시스템 아키텍처 전체도](#1-시스템-아키텍처-전체도)
2. [기술 스택](#2-기술-스택)
3. [디렉토리 구조](#3-디렉토리-구조)
4. [레이어별 설계 상세](#4-레이어별-설계-상세)
   - [4.1 하네스 관리 레이어](#41-하네스-관리-레이어)
   - [4.2 에이전트 실행 레이어](#42-에이전트-실행-레이어)
   - [4.3 드리프트 감지 레이어](#43-드리프트-감지-레이어)
   - [4.4 관찰 & 분석 레이어](#44-관찰--분석-레이어)
5. [데이터 모델](#5-데이터-모델)
6. [API 설계](#6-api-설계)
7. [통합 설계](#7-통합-설계)
8. [Phase별 개발 계획](#8-phase별-개발-계획)
9. [개발 환경 셋업](#9-개발-환경-셋업)
10. [코딩 컨벤션 & 하네스 자체 규칙](#10-코딩-컨벤션--하네스-자체-규칙)
11. [테스트 전략](#11-테스트-전략)
12. [배포 아키텍처](#12-배포-아키텍처)

---

## 1. 시스템 아키텍처 전체도

### 1.1 컴포넌트 다이어그램

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           HE-AMS Platform                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      API Gateway (FastAPI)                          │   │
│  │           REST API / WebSocket / MCP Server Endpoint               │   │
│  └──────────┬───────────────┬────────────────┬───────────────────────┘   │
│             │               │                │                            │
│  ┌──────────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐                   │
│  │  Harness        │ │  Agent      │ │  Drift      │                   │
│  │  Management     │ │  Execution  │ │  Detection  │                   │
│  │  Service        │ │  Service    │ │  Service    │                   │
│  │                 │ │             │ │             │                   │
│  │ • DocManager    │ │ • TaskQueue │ │ • Scanner   │                   │
│  │ • ConstraintReg │ │ • Orchestr. │ │ • Analyzer  │                   │
│  │ • KnowledgeRepo │ │ • Feedback  │ │ • GC Engine │                   │
│  └──────────┬──────┘ └──────┬──────┘ └──────┬──────┘                   │
│             │               │                │                            │
│  ┌──────────▼───────────────▼────────────────▼──────────────────────┐   │
│  │                    Observability Service                          │   │
│  │          Logger │ Metrics Collector │ Dashboard API              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Data Layer                                   │   │
│  │   PostgreSQL (메타데이터)  │  Redis (큐/캐시)  │  S3 (문서/로그) │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
         │                    │                    │                │
    ┌────▼────┐         ┌─────▼────┐         ┌────▼────┐    ┌─────▼────┐
    │ GitHub  │         │ Claude   │         │  CI/CD  │    │  Slack   │
    │ GitLab  │         │ Code/API │         │ Actions │    │  Webhook │
    └─────────┘         └──────────┘         └─────────┘    └──────────┘
```

### 1.2 데이터 흐름도

```
[개발자/에이전트]
      │
      │ 1. 작업 요청 (태스크 생성)
      ▼
[API Gateway]
      │
      │ 2. 하네스 컨텍스트 조회
      ▼
[Harness Management Service]
      │ ← AGENTS.md, 제약 규칙, 지식 저장소
      │
      │ 3. 컨텍스트 주입 후 작업 큐 등록
      ▼
[Agent Execution Service]
      │
      │ 4. 에이전트 실행 (Claude Code / API)
      ▼
[External Agent]
      │
      │ 5. 결과물 반환 (코드, PR)
      ▼
[Agent Execution Service]
      │
      │ 6. 피드백 루프 실행 (린터, 테스트)
      ▼
[Feedback Engine]
      │
      │ 7a. 통과 → PR 생성
      │ 7b. 실패 → 에이전트에게 피드백 반환 (3번으로)
      │
      │ 8. 행동 로그 기록
      ▼
[Observability Service]
      │
      │ 9. 드리프트 감지 (스케줄)
      ▼
[Drift Detection Service]
      │
      │ 10. 드리프트 발견 시 → 하네스 업데이트 제안
      ▼
[Harness Management Service]
```

### 1.3 하네스 계층 구조도

```
┌──────────────────────────────────────────────┐
│           GLOBAL HARNESS (조직 공통)          │
│  • 보안 규칙 (민감 파일 접근 금지)            │
│  • 커밋 메시지 형식                           │
│  • 라이센스 헤더 규칙                         │
└───────────────────┬──────────────────────────┘
                    │ 상속 (재정의 불가 항목)
        ┌───────────▼───────────┐
        │   TEAM HARNESS (팀별)  │
        │  • 언어별 코드 스타일  │
        │  • 브랜치 전략        │
        │  • 리뷰 프로세스      │
        └───────────┬───────────┘
                    │ 상속 (팀 규칙 내에서 확장 가능)
          ┌─────────▼─────────┐
          │  PROJECT HARNESS  │
          │  • 디렉토리 구조  │
          │  • 의존성 규칙    │
          │  • 테스트 기준    │
          └─────────┬─────────┘
                    │ 상속 (작업 범위로 제한)
            ┌───────▼───────┐
            │  TASK HARNESS │
            │  • 작업 범위  │
            │  • 임시 규칙  │
            └───────────────┘
```

### 1.4 피드백 루프 상세도

```
에이전트 출력
      │
      ▼
┌─────────────────────────────────────┐
│         GUIDE (방향 안내)            │
│                                     │
│  ┌──────────────┐ ┌──────────────┐  │
│  │ 예제 코드    │ │ 테스트 케이스 │  │
│  │ 주입         │ │ 안내         │  │
│  └──────────────┘ └──────────────┘  │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│         SENSOR (이상 감지)           │
│                                     │
│  pre-commit                         │
│  ├── 린터 (ruff, eslint)            │
│  ├── 타입 검사 (mypy, tsc)          │
│  └── 금지 패턴 탐지                  │
│                                     │
│  CI Pipeline                        │
│  ├── 단위 테스트                     │
│  ├── 통합 테스트                     │
│  └── 아키텍처 제약 검사              │
└────────────────┬────────────────────┘
                 │
          ┌──────▼──────┐
          │   통과?      │
          └──┬──────┬───┘
             │      │
           YES      NO
             │      │
             ▼      ▼
          PR 생성  피드백 패키징
                   → 에이전트 재실행
                   (최대 3회 재시도)
                   → 실패 시 에스컬레이션
```

---

## 2. 기술 스택

### 2.1 백엔드 서비스

| 컴포넌트 | 기술 | 선택 이유 |
|----------|------|-----------|
| API 서버 | **FastAPI** (Python 3.12) | 비동기 지원, 자동 OpenAPI 문서, Claude SDK와 동일 생태계 |
| 작업 큐 | **Celery** + **Redis** | 분산 태스크, 우선순위 큐, 재시도 로직 내장 |
| 데이터베이스 | **PostgreSQL 16** | 관계형 데이터, JSON 컬럼 지원, 트랜잭션 |
| 캐시 | **Redis 7** | 하네스 컨텍스트 캐싱, 세션 관리 |
| 파일 저장소 | **S3 호환** (MinIO 로컬 / AWS S3 프로덕션) | 지시문서, 로그, 지식 저장소 원본 |
| 스케줄러 | **Celery Beat** | 드리프트 감지 주기 실행 |

### 2.2 에이전트 통합

| 컴포넌트 | 기술 | 용도 |
|----------|------|------|
| Claude 연동 | **Anthropic Python SDK** (`anthropic>=0.40`) | 에이전트 API 호출 |
| MCP 서버 | **FastMCP** | Claude Code에서 HE-AMS 기능 직접 호출 |
| 컨텍스트 압축 | **tiktoken** | 하네스 컨텍스트 토큰 수 관리 |

### 2.3 드리프트 감지 도구

| 대상 | 도구 | 언어/환경 |
|------|------|-----------|
| 미사용 Python 코드 | **vulture** | Python |
| 미사용 JS/TS 코드 | **knip** | Node.js |
| 구조 드리프트 | 커스텀 스캐너 (Python) | 범용 |
| 문서 드리프트 | Claude API + diff 분석 | 범용 |
| 린트 통합 | **ruff** (Python), **eslint** (JS/TS) | 언어별 |

### 2.4 인프라 & DevOps

| 컴포넌트 | 기술 |
|----------|------|
| 컨테이너화 | Docker + Docker Compose |
| 오케스트레이션 | Kubernetes (프로덕션) |
| CI/CD | GitHub Actions |
| 모니터링 | Prometheus + Grafana |
| 로그 집계 | Loki + Grafana |
| 비밀 관리 | HashiCorp Vault / AWS Secrets Manager |

### 2.5 프론트엔드 (대시보드)

| 컴포넌트 | 기술 | 선택 이유 |
|----------|------|-----------|
| 프레임워크 | **Next.js 15** (App Router) | SSR/SSG, React 생태계 |
| UI 컴포넌트 | **shadcn/ui** + **Tailwind CSS** | 빠른 구성, 커스터마이징 |
| 차트 | **Recharts** | 드리프트 트렌드, 준수율 시각화 |
| 상태 관리 | **Zustand** + **TanStack Query** | 서버 상태와 클라이언트 상태 분리 |

---

## 3. 디렉토리 구조

```
he-ams/
│
├── apps/
│   ├── api/                        # FastAPI 백엔드
│   │   ├── he_ams/
│   │   │   ├── main.py             # 앱 진입점
│   │   │   ├── config.py           # 환경 설정
│   │   │   ├── harness/            # 하네스 관리 레이어
│   │   │   │   ├── documents/      # 지시문서 관리
│   │   │   │   ├── constraints/    # 아키텍처 제약 레지스트리
│   │   │   │   └── knowledge/      # 지식 저장소
│   │   │   ├── execution/          # 에이전트 실행 레이어
│   │   │   │   ├── orchestrator.py
│   │   │   │   ├── task_queue.py
│   │   │   │   └── feedback/       # 피드백 루프
│   │   │   ├── drift/              # 드리프트 감지 레이어
│   │   │   │   ├── scanner.py
│   │   │   │   ├── analyzers/      # 유형별 분석기
│   │   │   │   └── gc_engine.py
│   │   │   ├── observability/      # 관찰 & 분석 레이어
│   │   │   │   ├── logger.py
│   │   │   │   ├── metrics.py
│   │   │   │   └── dashboard_api.py
│   │   │   ├── integrations/       # 외부 시스템 통합
│   │   │   │   ├── github.py
│   │   │   │   ├── claude.py
│   │   │   │   └── slack.py
│   │   │   ├── models/             # SQLAlchemy 모델
│   │   │   ├── schemas/            # Pydantic 스키마
│   │   │   ├── api/                # 라우터
│   │   │   │   ├── v1/
│   │   │   │   └── mcp/            # MCP 엔드포인트
│   │   │   └── workers/            # Celery 태스크
│   │   ├── tests/
│   │   ├── alembic/                # DB 마이그레이션
│   │   ├── pyproject.toml
│   │   └── Dockerfile
│   │
│   └── dashboard/                  # Next.js 프론트엔드
│       ├── app/
│       │   ├── (auth)/
│       │   ├── harness/            # 하네스 관리 페이지
│       │   ├── agents/             # 에이전트 모니터링
│       │   ├── drift/              # 드리프트 현황
│       │   └── settings/
│       ├── components/
│       ├── lib/
│       └── package.json
│
├── packages/
│   ├── he-cli/                     # CLI 도구 (Phase 0)
│   │   ├── src/
│   │   │   ├── commands/
│   │   │   │   ├── init.py         # he init
│   │   │   │   ├── scan.py         # he scan
│   │   │   │   └── sync.py         # he sync
│   │   │   └── templates/          # AGENTS.md 등 템플릿
│   │   └── pyproject.toml
│   │
│   └── he-sdk/                     # Python SDK (외부 연동용)
│       ├── src/he_sdk/
│       └── pyproject.toml
│
├── infra/
│   ├── docker-compose.yml          # 로컬 개발 환경
│   ├── docker-compose.prod.yml
│   ├── k8s/                        # Kubernetes 매니페스트
│   └── terraform/                  # 인프라 코드
│
├── docs/
│   ├── HE_PRD.md
│   ├── HE_개발가이드.md
│   ├── adr/                        # Architecture Decision Records
│   └── api/                        # 자동 생성 API 문서
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── release.yml
│       └── drift-check.yml         # 하네스 드리프트 자동 감지
│
├── AGENTS.md                       # 이 프로젝트 자체의 하네스
└── pyproject.toml                  # 모노레포 루트 설정
```

---

## 4. 레이어별 설계 상세

### 4.1 하네스 관리 레이어

#### 지시문서 관리자 (Document Manager)

```
DocumentManager
├── create(template: DocTemplate, scope: HarnessScope) → Document
├── update(doc_id: str, changes: DocChangeset) → Document
├── sync(target_repos: list[Repo]) → SyncResult
├── validate(doc_id: str) → ValidationReport
└── get_context(scope: HarnessScope, token_budget: int) → HarnessContext
```

**지시문서 표준 스키마:**

```yaml
# AGENTS.md 메타데이터 헤더 (YAML frontmatter)
---
version: "1.2.0"
scope: project                    # global | team | project | task
team: backend
updated_at: 2026-04-29
reviewed_by: lesh915
next_review: 2026-10-29           # 6개월 후 유효성 검토
---

# AGENTS.md

## 1. 코드 스타일
## 2. 디렉토리 규칙
## 3. 절대 금지 항목
## 4. PR 작성 방식
## 5. Known Pitfalls         ← 에이전트 실수 자동 추가됨
## 6. 정리 규칙 (GC)
```

**계층별 컨텍스트 조립:**

```python
# harness/documents/context_builder.py

class HarnessContextBuilder:
    """
    에이전트에게 주입할 하네스 컨텍스트를 계층 순서대로 조립.
    토큰 예산 초과 시 하위 계층부터 압축.
    """
    PRIORITY_ORDER = ["global", "team", "project", "task"]
    MAX_TOKENS_PER_LAYER = 2000

    def build(self, scope: HarnessScope, budget: int) -> HarnessContext:
        layers = self._load_layers(scope)
        return self._assemble(layers, budget)

    def _assemble(self, layers: list[HarnessLayer], budget: int) -> HarnessContext:
        result = []
        remaining = budget
        for layer in layers:
            chunk = layer.to_text(max_tokens=min(remaining, self.MAX_TOKENS_PER_LAYER))
            result.append(chunk)
            remaining -= len(chunk.split())
            if remaining <= 0:
                break
        return HarnessContext(content="\n\n---\n\n".join(result))
```

#### 아키텍처 제약 레지스트리 (Constraint Registry)

```
ConstraintRegistry
├── register(rule: ConstraintRule) → Rule
├── validate(diff: CodeDiff) → ConstraintReport
├── generate_hooks(repo: Repo) → HookConfig
└── get_hint(violation: Violation) → str   # 수정 힌트 반환
```

**제약 규칙 정의 스키마:**

```yaml
# constraints/rules/no_temp_files.yaml
id: no-temp-files
name: 임시 파일 생성 금지
severity: error                   # error | warning | info
scope: global
patterns:
  forbidden_filenames:
    - "temp_*.py"
    - "*_new.py"
    - "*_backup.*"
    - "debug_*.py"
hint: |
  임시 파일 대신 feature 브랜치와 명확한 파일명을 사용하세요.
  예: utils_legacy.py (X) → utils.py (O)
auto_fix: false
references:
  - "AGENTS.md#정리-규칙"
```

#### 지식 저장소 (Knowledge Repository)

**ADR (Architecture Decision Record) 표준 형식:**

```markdown
---
id: ADR-0023
date: 2026-04-29
status: accepted                  # proposed | accepted | deprecated | superseded
deciders: [lesh915, team-backend]
---

# ADR-0023: Redis를 작업 큐 브로커로 선택

## 맥락
에이전트 작업 큐 브로커로 RabbitMQ와 Redis를 검토함.

## 결정
Redis(Celery)를 선택.

## 이유
- 이미 캐시 레이어로 Redis를 사용 중 → 인프라 단순화
- Celery의 우선순위 큐가 에이전트 작업 스케줄링에 충분

## 포기한 대안
- **RabbitMQ:** 메시지 보장성은 높으나 운영 복잡도 증가, 별도 인프라 필요
- **SQS:** 클라우드 종속성, 로컬 개발 환경 구성 번거로움

## 결과
에이전트가 이 결정을 알면: Redis 직접 조작 금지, Celery API를 통해서만 큐 접근.
```

---

### 4.2 에이전트 실행 레이어

#### 오케스트레이터 (Orchestrator)

```
AgentOrchestrator
├── submit(task: AgentTask) → TaskHandle
├── get_status(task_id: str) → TaskStatus
├── cancel(task_id: str) → bool
└── get_result(task_id: str) → AgentResult
```

**태스크 생명주기:**

```
PENDING → CONTEXT_LOADING → RUNNING → FEEDBACK_CHECK → COMPLETED
                                  ↓                ↓
                              FAILED          RETRY (최대 3회)
                                                   ↓
                                            ESCALATED (사람 개입 필요)
```

**태스크 정의 스키마:**

```python
# schemas/task.py
from pydantic import BaseModel
from enum import Enum

class TaskPriority(str, Enum):
    HIGH = "high"
    NORMAL = "normal"
    LOW = "low"

class AgentTask(BaseModel):
    id: str
    type: str                       # code_write | review | refactor | test_write
    description: str
    scope: HarnessScope             # 어느 하네스 계층 적용할지
    priority: TaskPriority = TaskPriority.NORMAL
    context_budget: int = 8000      # 최대 토큰 수
    max_retries: int = 3
    feedback_required: bool = True  # 피드백 루프 실행 여부
    metadata: dict = {}
```

#### 피드백 엔진 (Feedback Engine)

**피드백 패키지 구조:**

```python
# execution/feedback/package.py

class FeedbackPackage:
    """
    에이전트 재실행 시 주입하는 피드백 묶음.
    이전 시도의 모든 실패 정보를 포함.
    """
    attempt: int
    failures: list[FeedbackItem]     # 각 실패 항목
    hints: list[str]                 # 지식 저장소에서 추출한 힌트
    relevant_adr: list[str]          # 관련 의사결정 기록 ID

class FeedbackItem:
    source: str                      # "linter" | "test" | "constraint"
    severity: str                    # "error" | "warning"
    location: str                    # 파일:라인
    message: str                     # 구체적인 오류 메시지
    suggestion: str                  # 수정 방법 (제약 레지스트리에서 조회)
```

**피드백 루프 실행 순서:**

```python
# execution/feedback/runner.py

FEEDBACK_PIPELINE = [
    ("pre_commit",  PreCommitSensor,   severity="blocking"),
    ("lint",        LintSensor,        severity="blocking"),
    ("type_check",  TypeCheckSensor,   severity="blocking"),
    ("unit_test",   UnitTestSensor,    severity="blocking"),
    ("constraint",  ConstraintSensor,  severity="blocking"),
    ("style_guide", StyleGuideGuide,   severity="advisory"),
]
```

---

### 4.3 드리프트 감지 레이어

#### 드리프트 스캐너 아키텍처

```
DriftScanner
├── scan(repo: Repo, types: list[DriftType]) → DriftReport
├── schedule(repo: Repo, cron: str) → ScheduleHandle
└── explain(drift: DriftItem) → str    # AI로 드리프트 원인 설명 생성

DriftType (열거형)
├── CODE_DRIFT          # 미사용 코드
├── STRUCTURE_DRIFT     # 금지 파일명, 디렉토리 위반
├── DOC_DRIFT           # 지시문서 vs 실제 코드 불일치
└── HARNESS_DRIFT       # 하네스 규칙 자체의 모순/중복
```

**유형별 분석기 상세:**

```python
# drift/analyzers/structure_analyzer.py

class StructureDriftAnalyzer:
    FORBIDDEN_PATTERNS = [
        r"temp_.*\.(py|ts|js)",
        r".*_new\.(py|ts|js)",
        r".*_old\.(py|ts|js)",
        r".*_backup\.(py|ts|js)",
        r"debug_.*\.(py|ts|js)",
    ]

    def analyze(self, repo_path: Path) -> list[DriftItem]:
        items = []
        for file in repo_path.rglob("*"):
            for pattern in self.FORBIDDEN_PATTERNS:
                if re.match(pattern, file.name):
                    items.append(DriftItem(
                        type=DriftType.STRUCTURE_DRIFT,
                        severity="warning",
                        path=str(file),
                        message=f"금지된 파일명 패턴: {file.name}",
                        suggested_action="파일 이름을 의미있게 변경하거나 삭제하세요.",
                    ))
        return items
```

```python
# drift/analyzers/doc_analyzer.py

class DocDriftAnalyzer:
    """
    AGENTS.md에 명시된 규칙과 실제 코드 패턴을 비교.
    Claude API를 사용해 의미론적 불일치 감지.
    """
    def analyze(self, harness_doc: str, code_sample: str) -> list[DriftItem]:
        prompt = f"""
        다음 하네스 지시문서의 규칙과 실제 코드를 비교해서
        불일치 항목을 JSON 배열로 반환하세요.

        [지시문서]
        {harness_doc}

        [실제 코드 샘플]
        {code_sample}

        응답 형식: [{{"rule": "...", "violation": "...", "severity": "error|warning"}}]
        """
        response = self.claude_client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1000,
            messages=[{"role": "user", "content": prompt}],
        )
        return self._parse_response(response)
```

#### 가비지 컬렉션 엔진

**GC 실행 정책:**

```python
# drift/gc_engine.py

class GCPolicy:
    auto_fix_allowed = [
        "unused_import",            # 미사용 import 자동 제거
        "trailing_whitespace",      # 후행 공백 제거
    ]
    require_approval = [
        "unused_function",          # 미사용 함수 (사람 확인 필요)
        "forbidden_filename",       # 파일명 변경 (영향도 파악 필요)
        "doc_drift",                # 지시문서 업데이트 (의도 확인 필요)
    ]
    never_auto_fix = [
        "harness_drift",            # 하네스 규칙 자체 변경은 사람만
        "architecture_violation",   # 아키텍처 결정 번복은 사람만
    ]
```

---

### 4.4 관찰 & 분석 레이어

#### 메트릭 수집 구조

```python
# observability/metrics.py

# Prometheus 메트릭 정의
agent_tasks_total = Counter(
    "agent_tasks_total",
    "에이전트 태스크 총 수",
    ["status", "type", "scope"]
)
constraint_violations_total = Counter(
    "constraint_violations_total",
    "제약 위반 총 수",
    ["rule_id", "severity"]
)
drift_items_detected = Gauge(
    "drift_items_detected",
    "감지된 드리프트 항목 수",
    ["drift_type", "repo"]
)
harness_health_score = Gauge(
    "harness_health_score",
    "하네스 건강도 점수 (0~100)",
    ["repo", "scope"]
)
feedback_loop_duration_seconds = Histogram(
    "feedback_loop_duration_seconds",
    "피드백 루프 실행 시간",
    buckets=[1, 5, 10, 30, 60, 120]
)
```

#### 하네스 건강도 점수 산정

```
HarnessHealthScore (0 ~ 100점)

= 100
  - (코드 드리프트 건수 × 2)
  - (문서 드리프트 건수 × 5)
  - (구조 드리프트 건수 × 3)
  - (제약 위반 건수 × 4)
  - (유효기간 만료 지식 항목 × 1)
  + (최근 7일 에이전트 준수율 × 0.3)  ← 보너스

최솟값: 0 (음수 불가)
등급:
  90~100 → Excellent (녹색)
  70~89  → Good      (하늘색)
  50~69  → Warning   (노란색)
  0~49   → Critical  (빨간색)
```

---

## 5. 데이터 모델

### 5.1 핵심 엔티티 관계도

```
┌──────────────┐      ┌───────────────┐      ┌─────────────────┐
│ Organization │──1:N─│    Team       │──1:N─│    Project      │
└──────────────┘      └───────────────┘      └────────┬────────┘
                                                      │
                                    ┌─────────────────┼─────────────────┐
                                    │                 │                 │
                             ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
                             │  HarnessDoc │  │ConstraintRule│  │ KnowledgeItem│
                             └─────────────┘  └─────────────┘  └─────────────┘
                                    │
                             ┌──────▼──────┐
                             │  AgentTask  │
                             └──────┬──────┘
                                    │
                      ┌─────────────┼─────────────┐
                      │             │             │
               ┌──────▼──────┐ ┌───▼───────┐ ┌──▼──────────┐
               │ TaskAttempt │ │FeedbackLog│ │  DriftItem  │
               └─────────────┘ └───────────┘ └─────────────┘
```

### 5.2 주요 테이블 정의

```sql
-- 하네스 문서
CREATE TABLE harness_documents (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scope       VARCHAR(20) NOT NULL,          -- global|team|project|task
    scope_id    UUID NOT NULL,                 -- 해당 스코프의 ID
    content     TEXT NOT NULL,
    version     VARCHAR(20) NOT NULL,
    metadata    JSONB NOT NULL DEFAULT '{}',
    created_by  VARCHAR(100) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW(),
    next_review TIMESTAMPTZ,
    is_active   BOOLEAN DEFAULT TRUE
);

-- 제약 규칙
CREATE TABLE constraint_rules (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_key    VARCHAR(100) UNIQUE NOT NULL,  -- no-temp-files
    name        VARCHAR(200) NOT NULL,
    severity    VARCHAR(20) NOT NULL,           -- error|warning|info
    scope       VARCHAR(20) NOT NULL,
    config      JSONB NOT NULL,                -- 패턴, 조건 등
    hint        TEXT,
    auto_fix    BOOLEAN DEFAULT FALSE,
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 에이전트 태스크
CREATE TABLE agent_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    type            VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    priority        VARCHAR(20) DEFAULT 'normal',
    attempt_count   INTEGER DEFAULT 0,
    max_retries     INTEGER DEFAULT 3,
    result          JSONB,
    error           TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ
);

-- 드리프트 항목
CREATE TABLE drift_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    drift_type      VARCHAR(30) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    file_path       VARCHAR(500),
    message         TEXT NOT NULL,
    suggested_action TEXT,
    detected_at     TIMESTAMPTZ DEFAULT NOW(),
    resolved_at     TIMESTAMPTZ,
    resolved_by     VARCHAR(100),
    resolution_note TEXT
);

-- 하네스 건강도 이력
CREATE TABLE harness_health_snapshots (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id  UUID NOT NULL REFERENCES projects(id),
    score       INTEGER NOT NULL,
    breakdown   JSONB NOT NULL,   -- 항목별 감점 내역
    snapshot_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 6. API 설계

### 6.1 REST API 구조

**기본 URL:** `https://api.he-ams.internal/v1`

#### 하네스 관리

```
GET    /harness/{scope}/{scope_id}              # 하네스 컨텍스트 조회
POST   /harness/{scope}/{scope_id}/documents    # 지시문서 생성
PATCH  /harness/{scope}/{scope_id}/documents/{doc_id}  # 지시문서 수정
POST   /harness/{scope}/{scope_id}/sync         # 대상 저장소에 동기화

GET    /harness/constraints                     # 제약 규칙 목록
POST   /harness/constraints                     # 제약 규칙 등록
POST   /harness/constraints/validate            # 코드 diff 검증

GET    /harness/knowledge                       # 지식 항목 목록
POST   /harness/knowledge                       # 지식 항목 등록
GET    /harness/knowledge/search?q=...          # 지식 검색
```

#### 에이전트 실행

```
POST   /tasks                                   # 태스크 생성
GET    /tasks/{task_id}                         # 태스크 상태 조회
DELETE /tasks/{task_id}                         # 태스크 취소
GET    /tasks/{task_id}/result                  # 태스크 결과 조회
GET    /tasks/{task_id}/feedback                # 피드백 로그 조회
```

#### 드리프트

```
POST   /drift/scan                              # 즉시 드리프트 스캔
GET    /drift/items                             # 드리프트 항목 목록
PATCH  /drift/items/{item_id}/resolve           # 드리프트 해소 처리
GET    /drift/report                            # 드리프트 리포트
POST   /drift/schedule                          # 정기 스캔 설정
```

#### 관찰 & 분석

```
GET    /health/{project_id}                     # 하네스 건강도 점수
GET    /health/{project_id}/history             # 건강도 이력
GET    /metrics/compliance                      # 에이전트 준수율
GET    /metrics/feedback-loop                   # 피드백 루프 효율
```

### 6.2 MCP 서버 엔드포인트

Claude Code에서 직접 호출 가능한 MCP 툴:

```python
# api/mcp/tools.py  (FastMCP 기반)

@mcp.tool()
def get_harness_context(scope: str, scope_id: str, token_budget: int = 4000) -> str:
    """현재 작업 범위의 하네스 컨텍스트를 가져옵니다."""
    ...

@mcp.tool()
def report_agent_mistake(
    task_id: str,
    mistake_type: str,
    description: str,
    suggested_rule: str | None = None
) -> dict:
    """에이전트 실수를 보고하고 지시문서 업데이트를 제안합니다."""
    ...

@mcp.tool()
def validate_code(file_path: str, content: str) -> dict:
    """코드가 현재 아키텍처 제약을 준수하는지 검사합니다."""
    ...

@mcp.tool()
def search_knowledge(query: str, limit: int = 5) -> list[dict]:
    """지식 저장소에서 관련 의사결정 기록을 검색합니다."""
    ...
```

### 6.3 Webhook 수신 엔드포인트

```
POST   /webhooks/github                         # GitHub 이벤트 수신
POST   /webhooks/gitlab                         # GitLab 이벤트 수신

# 처리하는 GitHub 이벤트:
# - push: 코드 변경 시 드리프트 증분 스캔
# - pull_request.closed (merged): PR 패턴 분석
# - check_run.completed: CI 결과 → 피드백 루프 트리거
```

---

## 7. 통합 설계

### 7.1 GitHub 통합

```python
# integrations/github.py

class GitHubIntegration:
    """
    GitHub App으로 설치. OAuth 앱이 아닌 GitHub App 사용.
    이유: 저장소별 권한 제어, 봇 계정 불필요.
    """
    required_permissions = {
        "contents": "write",         # AGENTS.md 읽기/쓰기
        "pull_requests": "write",    # PR 생성
        "checks": "write",           # CI 상태 업데이트
        "statuses": "write",         # 커밋 상태
    }

    async def handle_push(self, payload: dict) -> None:
        """push 이벤트: 변경 파일 기반 증분 드리프트 스캔"""
        changed_files = self._extract_changed_files(payload)
        await self.drift_service.scan_incremental(changed_files)

    async def handle_pr_merged(self, payload: dict) -> None:
        """PR 머지: 리뷰 코멘트 패턴 분석 → 지시문서 개선 제안"""
        pr_data = await self._fetch_pr_details(payload["pull_request"]["number"])
        await self.harness_service.analyze_pr_patterns(pr_data)
```

### 7.2 Claude API 통합

```python
# integrations/claude.py

class ClaudeAgentRunner:
    def __init__(self):
        self.client = anthropic.Anthropic()

    async def run_task(self, task: AgentTask, context: HarnessContext) -> AgentResult:
        system_prompt = self._build_system_prompt(context)

        response = await self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=8096,
            system=system_prompt,           # 하네스 컨텍스트 주입
            messages=self._build_messages(task),
            tools=self._get_allowed_tools(task.scope),
        )
        return AgentResult.from_response(response)

    def _build_system_prompt(self, context: HarnessContext) -> str:
        return f"""
당신은 이 프로젝트의 하네스 규칙을 반드시 준수하며 작업하는 에이전트입니다.

=== 하네스 컨텍스트 ===
{context.content}

=== 규칙 준수 우선순위 ===
1. 글로벌 하네스 (절대 위반 불가)
2. 팀 하네스
3. 프로젝트 하네스
4. 작업 지시사항

규칙이 불명확하거나 충돌할 경우, 작업을 진행하지 말고 명시적으로 질문하세요.
"""
```

### 7.3 CI/CD 통합

**GitHub Actions 워크플로우 (자동 생성):**

```yaml
# .github/workflows/he-ams-feedback.yml
# HE-AMS가 저장소에 자동 생성하는 워크플로우

name: HE-AMS Feedback Loop
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  harness-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 하네스 컨텍스트 로드
        id: harness
        run: |
          curl -s "$HE_AMS_URL/v1/harness/project/$PROJECT_ID" \
            -H "Authorization: Bearer $HE_AMS_TOKEN" \
            > .harness_context.json
        env:
          HE_AMS_URL: ${{ vars.HE_AMS_URL }}
          HE_AMS_TOKEN: ${{ secrets.HE_AMS_TOKEN }}
          PROJECT_ID: ${{ vars.HE_AMS_PROJECT_ID }}

      - name: 아키텍처 제약 검사
        run: |
          curl -s -X POST "$HE_AMS_URL/v1/harness/constraints/validate" \
            -H "Authorization: Bearer $HE_AMS_TOKEN" \
            -d '{"diff_url": "${{ github.event.pull_request.diff_url }}"}' \
            | tee constraint_result.json
          jq -e '.violations | length == 0' constraint_result.json

      - name: 구조 드리프트 감지
        run: |
          curl -s -X POST "$HE_AMS_URL/v1/drift/scan" \
            -H "Authorization: Bearer $HE_AMS_TOKEN" \
            -d '{"repo": "${{ github.repository }}", "types": ["structure"]}'

      - name: 결과 PR 코멘트
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const result = require('./constraint_result.json');
            const body = result.violations
              .map(v => `- **${v.severity}** ${v.location}: ${v.message}\n  > ${v.suggestion}`)
              .join('\n');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## HE-AMS 하네스 검사 결과\n\n${body}`
            });
```

---

## 8. Phase별 개발 계획

### Phase 0: CLI 기반 기반 구축 (M1~M2)

**목표:** 단일 프로젝트에서 `he` CLI 명령 하나로 하네스를 초기화.

```bash
# Phase 0 완료 후 가능한 명령어
he init                             # 프로젝트 하네스 초기화
he scan --type structure            # 구조 드리프트 스캔
he scan --type code                 # 코드 드리프트 스캔
he validate <file>                  # 파일 제약 검사
he report                           # 드리프트 리포트 출력
```

**구현 순서:**

```
Week 1: 프로젝트 셋업
  ├── 모노레포 구조 생성
  ├── he-cli 패키지 초기화
  └── AGENTS.md 템플릿 작성

Week 2: he init 구현
  ├── 프로젝트 유형 감지 (Python/Node/Go)
  ├── AGENTS.md 자동 생성
  ├── pre-commit 훅 자동 구성
  └── docs/decisions/ 디렉토리 생성

Week 3: 드리프트 스캐너 구현
  ├── 구조 드리프트 분석기
  ├── 코드 드리프트 분석기 (vulture/knip 래퍼)
  └── 리포트 출력 포맷터

Week 4: 테스트 & 문서화
  ├── 단위 테스트 커버리지 80% 이상
  ├── README 작성
  └── 예제 프로젝트 구성
```

### Phase 1: 핵심 서비스 (M3~M4)

**목표:** FastAPI 서버와 대시보드 기본 기능.

**마일스톤:**

| 주차 | 목표 | 산출물 |
|------|------|--------|
| M3W1 | DB 스키마 & API 기반 | PostgreSQL 마이그레이션, FastAPI 기본 라우터 |
| M3W2 | 하네스 관리 API | 지시문서 CRUD, 제약 규칙 CRUD |
| M3W3 | GitHub 통합 | Webhook 수신, push 이벤트 처리 |
| M3W4 | CI 피드백 루프 | GitHub Actions 워크플로우 자동 생성 |
| M4W1 | 드리프트 자동화 | Celery Beat 스케줄, 주간 스캔 |
| M4W2 | Slack 알림 | 드리프트 리포트, 건강도 점수 |
| M4W3 | 대시보드 기본 | 드리프트 현황, 건강도 점수 시각화 |
| M4W4 | 통합 테스트 & 베타 | 실제 프로젝트 적용 테스트 |

### Phase 2: 자동화 심화 (M5~M6)

**목표:** 에이전트 실수 → 하네스 자동 학습.

**핵심 기능:**

```
에이전트 행동 로그 수집
  └── PR 패턴 분석 (머지된 PR에서 리뷰 코멘트 패턴 추출)
        └── 지시문서 업데이트 제안 (사람 승인 후 반영)
              └── 지식 저장소 자동 축적
```

**MCP 서버 출시:** Claude Code에서 `he_ams` 툴 직접 호출 가능.

### Phase 3: 조직 확장 (M7~M9)

**목표:** 멀티 팀·멀티 저장소.

**핵심 기능:**
- 하네스 계층 구조 UI (글로벌 → 팀 → 프로젝트)
- 팀별 규칙 충돌 감지 및 해소
- 엔지니어링 매니저 ROI 대시보드

---

## 9. 개발 환경 셋업

### 9.1 로컬 환경 요구사항

```
Python 3.12+
Node.js 22+
Docker Desktop 4.x+
Git 2.40+
```

### 9.2 초기 셋업

```bash
# 1. 저장소 클론
git clone https://github.com/lesh915/agent-dev.git
cd agent-dev

# 2. Python 가상환경 (uv 권장)
pip install uv
uv venv --python 3.12
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# 3. 의존성 설치
uv sync --all-extras

# 4. 로컬 인프라 실행 (PostgreSQL, Redis, MinIO)
docker compose -f infra/docker-compose.yml up -d

# 5. 환경 변수 설정
cp .env.example .env
# .env 파일에서 필수 항목 설정:
#   DATABASE_URL, REDIS_URL, ANTHROPIC_API_KEY, GITHUB_APP_*

# 6. DB 마이그레이션
cd apps/api
alembic upgrade head

# 7. 개발 서버 실행
uvicorn he_ams.main:app --reload --port 8000

# 8. 프론트엔드 (별도 터미널)
cd apps/dashboard
npm install
npm run dev                        # localhost:3000
```

### 9.3 환경 변수 목록

```bash
# .env.example

# 데이터베이스
DATABASE_URL=postgresql+asyncpg://heams:heams@localhost:5432/heams
REDIS_URL=redis://localhost:6379/0

# AI
ANTHROPIC_API_KEY=sk-ant-...

# GitHub App
GITHUB_APP_ID=123456
GITHUB_APP_PRIVATE_KEY_PATH=./github-app.pem
GITHUB_WEBHOOK_SECRET=...

# 스토리지
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET_HARNESS=he-ams-harness
S3_BUCKET_LOGS=he-ams-logs

# 알림
SLACK_WEBHOOK_URL=https://hooks.slack.com/...

# 보안
JWT_SECRET_KEY=...
CORS_ORIGINS=http://localhost:3000
```

### 9.4 하네스 적용 (이 프로젝트 자체)

이 프로젝트도 HE-AMS 원칙을 자기 자신에 적용합니다. 루트의 `AGENTS.md`를 참조하세요.

```bash
# he CLI로 이 프로젝트 하네스 상태 확인
python -m he_cli scan
python -m he_cli report
```

---

## 10. 코딩 컨벤션 & 하네스 자체 규칙

### 10.1 Python (백엔드)

```python
# 허용: async/await 일관성
async def get_harness(scope_id: str) -> HarnessDocument: ...

# 금지: 동기 함수에서 await 없이 비동기 호출
def get_harness(scope_id: str):
    return asyncio.run(...)            # X — 이벤트 루프 충돌 위험

# 허용: Pydantic v2 스타일
class TaskCreate(BaseModel):
    model_config = ConfigDict(strict=True)
    description: str
    priority: TaskPriority = TaskPriority.NORMAL

# 금지: 직접 dict 반환 (타입 안전성 없음)
async def create_task() -> dict: ...  # X
async def create_task() -> TaskResponse: ...  # O

# 린터: ruff (black + isort + flake8 통합)
# 타입 검사: mypy --strict
```

### 10.2 TypeScript (프론트엔드)

```typescript
// 허용: 서버 컴포넌트에서 직접 데이터 페칭
async function DriftDashboard() {
  const items = await fetchDriftItems();
  return <DriftList items={items} />;
}

// 금지: any 타입 사용
const handleClick = (e: any) => { ... }  // X
const handleClick = (e: React.MouseEvent) => { ... }  // O

// API 클라이언트: 자동 생성된 타입만 사용
// openapi-typescript로 FastAPI 스펙에서 생성
import type { components } from "@/lib/api-types";
type DriftItem = components["schemas"]["DriftItem"];
```

### 10.3 금지 패턴 요약

```
파일명:
  ❌ temp_*.py, *_new.ts, *_old.js, *_backup.*, debug_*
  ✅ 기능을 명확히 나타내는 이름

import:
  ❌ from he_ams import *
  ✅ from he_ams.harness.documents import DocumentManager

에러 처리:
  ❌ except Exception: pass
  ✅ except SpecificError as e: logger.error(...); raise

비밀 값:
  ❌ api_key = "sk-ant-..."  (하드코딩)
  ✅ api_key = settings.ANTHROPIC_API_KEY  (환경 변수)
```

---

## 11. 테스트 전략

### 11.1 테스트 피라미드

```
         ┌─────────────┐
         │   E2E 테스트  │  ← 핵심 사용자 플로우 5개만
         │    (5%)      │
        ┌┴─────────────┴┐
        │ 통합 테스트    │  ← 레이어 간 경계, 외부 통합
        │   (25%)       │
       ┌┴───────────────┴┐
       │   단위 테스트    │  ← 비즈니스 로직, 분석기
       │    (70%)        │
       └─────────────────┘
```

### 11.2 핵심 테스트 케이스

```python
# tests/unit/test_drift_analyzer.py

class TestStructureDriftAnalyzer:
    def test_detects_temp_file(self, tmp_path):
        (tmp_path / "temp_utils.py").touch()
        analyzer = StructureDriftAnalyzer()
        items = analyzer.analyze(tmp_path)
        assert len(items) == 1
        assert items[0].drift_type == DriftType.STRUCTURE_DRIFT

    def test_allows_normal_files(self, tmp_path):
        (tmp_path / "utils.py").touch()
        analyzer = StructureDriftAnalyzer()
        items = analyzer.analyze(tmp_path)
        assert len(items) == 0


# tests/integration/test_feedback_loop.py

class TestFeedbackLoop:
    async def test_constraint_violation_triggers_retry(self, client, fake_agent):
        """제약 위반 시 에이전트가 재시도되는지 검증"""
        fake_agent.set_response(violates_constraint=True)
        task = await client.post("/v1/tasks", json={...})
        await asyncio.sleep(0.5)
        status = await client.get(f"/v1/tasks/{task['id']}")
        assert status["attempt_count"] == 2
```

### 11.3 테스트 커버리지 기준

| 모듈 | 최소 커버리지 |
|------|--------------|
| drift/analyzers/ | 90% |
| harness/documents/ | 85% |
| harness/constraints/ | 90% |
| execution/feedback/ | 85% |
| integrations/ | 70% (외부 API 모킹) |
| api/ | 80% |

---

## 12. 배포 아키텍처

### 12.1 로컬 개발 (Docker Compose)

```yaml
# infra/docker-compose.yml
services:
  api:
    build: apps/api
    ports: ["8000:8000"]
    depends_on: [postgres, redis]

  worker:
    build: apps/api
    command: celery -A he_ams.workers worker -l info
    depends_on: [postgres, redis]

  scheduler:
    build: apps/api
    command: celery -A he_ams.workers beat -l info
    depends_on: [postgres, redis]

  dashboard:
    build: apps/dashboard
    ports: ["3000:3000"]

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: heams
      POSTGRES_USER: heams
      POSTGRES_PASSWORD: heams

  redis:
    image: redis:7-alpine

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]
```

### 12.2 프로덕션 (Kubernetes 개요)

```
Namespace: he-ams
│
├── Deployments
│   ├── api           (2 replicas, HPA: 2~10)
│   ├── worker        (3 replicas, HPA: 3~20)
│   ├── scheduler     (1 replica, no HPA)
│   └── dashboard     (2 replicas)
│
├── Services
│   ├── api-svc       (ClusterIP)
│   ├── dashboard-svc (ClusterIP)
│   └── Ingress       (NGINX, TLS)
│
├── StatefulSets
│   ├── postgres      (PVC: 100Gi)
│   └── redis         (PVC: 10Gi)
│
└── CronJobs
    └── weekly-drift-scan  (cron: "0 9 * * 1")
```

### 12.3 CI/CD 파이프라인

```
Push to main
    │
    ├── [lint] ruff check + mypy
    ├── [test] pytest --cov
    ├── [build] Docker 이미지 빌드
    ├── [scan] 이 프로젝트 자체 드리프트 스캔
    │
    └── [deploy] (main 브랜치만)
            ├── 스테이징 배포
            ├── 통합 테스트
            └── 프로덕션 배포 (수동 승인)
```

---

## 부록 A: 핵심 결정 사항 (ADR 요약)

| ADR | 결정 | 이유 |
|-----|------|------|
| ADR-001 | FastAPI 선택 | 비동기 지원, Python 생태계 통일 |
| ADR-002 | 모노레포 구조 | CLI·API·SDK·대시보드 간 코드 공유 |
| ADR-003 | Redis + Celery | 이미 캐시 레이어로 사용, 운영 단순화 |
| ADR-004 | MCP 서버 방식 | Claude Code와 네이티브 통합, 별도 플러그인 불필요 |
| ADR-005 | 하네스 계층 상속 시 보안 규칙 재정의 불가 | 글로벌 보안 규칙은 팀/프로젝트가 약화 불가 |

## 부록 B: 용어 & 약어

| 용어 | 정의 |
|------|------|
| HE-AMS | Harness Engineering Agent Management System |
| ADR | Architecture Decision Record |
| GC | Garbage Collection (하네스 드리프트 정리) |
| MCP | Model Context Protocol |
| HPA | Horizontal Pod Autoscaler |
| Scope | 하네스 적용 범위 (global / team / project / task) |

---

*이 개발가이드는 HE_PRD.md와 함께 읽어야 합니다. 아키텍처 변경 시 두 문서를 동시에 업데이트하세요.*
