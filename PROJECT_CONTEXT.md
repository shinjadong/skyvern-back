# Skyvern Backend - 완전한 프로젝트 컨텍스트 문서

> **문서 작성일**: 2025-12-02
> **저장소**: https://github.com/shinjadong/skyvern-back.git
> **원본 프로젝트**: [Skyvern-AI/skyvern](https://github.com/Skyvern-AI/skyvern)

---

## 1. 프로젝트 개요

### 1.1 Skyvern이란?

Skyvern은 **LLM(대규모 언어 모델)과 컴퓨터 비전을 활용한 브라우저 자동화 플랫폼**입니다. 기존의 CSS 셀렉터나 XPath 기반 자동화와 달리, Skyvern은 웹사이트의 시각적 레이아웃과 텍스트를 AI가 이해하여 동적으로 상호작용합니다.

**핵심 차별점:**
- 사이트 구조 변경에 강건함 (셀렉터 의존 X)
- 자연어 지시로 복잡한 작업 수행
- 멀티스텝 워크플로우 자동화
- 컴퓨터 비전 기반 요소 인식

### 1.2 이 저장소의 목적

이 저장소는 **Skyvern의 백엔드 전용 코드**를 분리한 것입니다:
- 프론트엔드 (`skyvern-frontend/`) 제외
- Python 백엔드, API, 워크플로우 엔진, 브라우저 자동화 핵심 코드 포함
- 독립적인 백엔드 서비스 배포 및 개발 가능

### 1.3 주요 사용 사례

1. **웹 폼 자동 작성**: 보험 신청, 회원가입, 데이터 입력
2. **데이터 스크래핑**: 웹사이트에서 구조화된 데이터 추출
3. **블로그/SNS 자동 포스팅**: 콘텐츠 자동 발행 (네이버 블로그 등)
4. **E-Commerce 자동화**: 가격 모니터링, 자동 주문
5. **RPA(Robotic Process Automation)**: 반복적인 웹 작업 자동화

---

## 2. 시스템 아키텍처

### 2.1 전체 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SKYVERN BACKEND                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │   CLI       │    │  REST API   │    │  WebSocket  │    │     MCP     │  │
│  │ (skyvern)   │    │  (FastAPI)  │    │  Streaming  │    │   Server    │  │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘  │
│         │                  │                  │                  │          │
│         └──────────────────┴──────────────────┴──────────────────┘          │
│                                    │                                        │
│                                    ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        FORGE (Core Engine)                           │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐     │  │
│  │  │   Agent    │  │  Workflow  │  │   LLM      │  │  Artifact  │     │  │
│  │  │  System    │  │   Engine   │  │  Handler   │  │  Manager   │     │  │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘     │  │
│  └────────┼───────────────┼───────────────┼───────────────┼────────────┘  │
│           │               │               │               │               │
│           ▼               ▼               ▼               ▼               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        WEBEYE (Browser Layer)                        │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐     │  │
│  │  │  Browser   │  │   DOM      │  │   Action   │  │ Screenshot │     │  │
│  │  │  Factory   │  │  Scraper   │  │  Handler   │  │  Capture   │     │  │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘     │  │
│  └────────┼───────────────┼───────────────┼───────────────┼────────────┘  │
│           │               │               │               │               │
│           └───────────────┴───────────────┴───────────────┘               │
│                                    │                                        │
│                                    ▼                                        │
│                    ┌──────────────────────────────┐                        │
│                    │   Playwright (Browser)        │                        │
│                    │   - Chromium (Headful/less)   │                        │
│                    │   - CDP Connect Mode          │                        │
│                    └──────────────────────────────┘                        │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                              EXTERNAL SERVICES                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │ PostgreSQL │  │    LLM     │  │  Bitwarden │  │   AWS S3   │           │
│  │  Database  │  │ Providers  │  │ (Secrets)  │  │  Storage   │           │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 핵심 컴포넌트 상세

#### 2.2.1 Forge (핵심 엔진) - `skyvern/forge/`

Skyvern의 핵심 비즈니스 로직을 담당합니다.

```
skyvern/forge/
├── sdk/
│   ├── api/              # API 유틸리티 (LLM, 파일, 암호화)
│   │   ├── llm/          # LLM 설정 및 호출 관리
│   │   │   ├── config_registry.py   # LLM 설정 레지스트리
│   │   │   ├── api_handler_factory.py
│   │   │   └── models.py
│   │   ├── aws.py        # AWS S3 연동
│   │   ├── azure.py      # Azure Blob Storage
│   │   └── crypto.py     # 암호화
│   ├── artifact/         # 아티팩트(스크린샷, 비디오 등) 관리
│   │   ├── manager.py
│   │   └── storage/      # 로컬/S3 스토리지
│   ├── db/               # 데이터베이스 레이어
│   │   ├── client.py     # DB 클라이언트
│   │   └── models.py     # SQLAlchemy 모델
│   ├── routes/           # FastAPI 라우트
│   │   ├── agent_protocol.py
│   │   ├── browser_sessions.py
│   │   ├── credentials.py
│   │   ├── debug_sessions.py
│   │   ├── scripts.py
│   │   └── streaming/    # WebSocket 스트리밍
│   ├── workflow/         # 워크플로우 엔진
│   │   ├── models/       # 워크플로우 모델
│   │   └── blocks/       # 워크플로우 블록
│   └── core/             # 공통 유틸리티
│       ├── security.py
│       └── skyvern_context.py
└── prompts/              # LLM 프롬프트 템플릿
```

#### 2.2.2 Webeye (브라우저 레이어) - `skyvern/webeye/`

Playwright 기반 브라우저 자동화를 담당합니다.

```
skyvern/webeye/
├── browser_factory.py       # 브라우저 인스턴스 생성
├── browser_manager.py       # 브라우저 세션 관리
├── persistent_sessions_manager.py  # 영구 세션 관리
├── actions/                 # 브라우저 액션
│   ├── actions.py           # 액션 실행기
│   └── handler.py           # 액션 핸들러
├── scraper/                 # DOM 스크래핑
│   ├── scraper.py           # 페이지 스크래핑
│   └── domUtils.js          # DOM 유틸리티 (JS)
└── utils/                   # 유틸리티
```

**브라우저 타입:**
| 타입 | 설명 | 사용 시나리오 |
|------|------|--------------|
| `chromium-headful` | GUI 있는 브라우저 | 개발/디버깅 |
| `chromium-headless` | GUI 없는 브라우저 | 프로덕션 서버 |
| `cdp-connect` | 기존 Chrome 연결 | 로그인 세션 유지 |

#### 2.2.3 Services - `skyvern/services/`

비즈니스 서비스 레이어입니다.

```
skyvern/services/
├── bitwarden.py            # 패스워드 매니저 연동
├── onepassword.py          # 1Password 연동
└── ... (기타 서비스)
```

#### 2.2.4 CLI - `skyvern/cli/`

명령줄 인터페이스입니다.

```
skyvern/cli/
├── commands.py             # 메인 CLI 진입점
├── run_commands.py         # skyvern run server/ui/all
├── stop_commands.py        # skyvern stop
├── status.py               # skyvern status
├── init_command.py         # skyvern init
├── llm_setup.py            # skyvern init llm
├── quickstart.py           # skyvern quickstart
└── mcp.py                  # MCP 서버 실행
```

**주요 CLI 명령어:**
```bash
# 첫 설치 시
skyvern quickstart

# 서비스 실행
skyvern run server        # 백엔드만
skyvern run all           # 백엔드 + UI

# 상태 확인
skyvern status

# 서비스 중지
skyvern stop all

# LLM 설정
skyvern init llm
```

---

## 3. 데이터 모델 및 스키마

### 3.1 핵심 엔티티

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│  Organization   │──────▶│     Workflow    │──────▶│  WorkflowRun    │
│                 │       │                 │       │                 │
│ - org_id        │       │ - workflow_id   │       │ - run_id        │
│ - name          │       │ - title         │       │ - status        │
│ - api_key       │       │ - definition    │       │ - started_at    │
└─────────────────┘       │ - parameters    │       │ - completed_at  │
                          │ - blocks        │       │ - output        │
                          └─────────────────┘       └────────┬────────┘
                                                             │
                          ┌─────────────────┐       ┌────────▼────────┐
                          │      Task       │──────▶│      Step       │
                          │                 │       │                 │
                          │ - task_id       │       │ - step_id       │
                          │ - url           │       │ - action        │
                          │ - navigation_   │       │ - status        │
                          │   goal          │       │ - screenshot    │
                          └─────────────────┘       └─────────────────┘
```

### 3.2 워크플로우 블록 타입

```python
class BlockType(StrEnum):
    TASK = "task"                    # 단일 태스크 실행
    TaskV2 = "task_v2"               # 개선된 태스크 (프롬프트 기반)
    FOR_LOOP = "for_loop"            # 반복 블록
    CONDITIONAL = "conditional"       # 조건 분기
    CODE = "code"                    # Python 코드 실행
    TEXT_PROMPT = "text_prompt"      # LLM 텍스트 프롬프트
    DOWNLOAD_TO_S3 = "download_to_s3"
    UPLOAD_TO_S3 = "upload_to_s3"
    FILE_UPLOAD = "file_upload"      # 파일 업로드
    SEND_EMAIL = "send_email"        # 이메일 발송
    FILE_URL_PARSER = "file_url_parser"
    VALIDATION = "validation"        # 검증 블록
    ACTION = "action"                # 단일 액션
    NAVIGATION = "navigation"        # 네비게이션
    EXTRACTION = "extraction"        # 데이터 추출
    LOGIN = "login"                  # 로그인 처리
    WAIT = "wait"                    # 대기
    FILE_DOWNLOAD = "file_download"  # 파일 다운로드
    GOTO_URL = "goto_url"            # URL 이동
    PDF_PARSER = "pdf_parser"        # PDF 파싱
    HTTP_REQUEST = "http_request"    # HTTP 요청
    HUMAN_INTERACTION = "human_interaction"  # 인간 개입 요청
```

### 3.3 파라미터 타입

```python
class ParameterType(StrEnum):
    WORKFLOW = "workflow"           # 워크플로우 입력 파라미터
    CONTEXT = "context"             # 컨텍스트 파라미터
    OUTPUT = "output"               # 출력 파라미터
    AWS_SECRET = "aws_secret"       # AWS 시크릿
    BITWARDEN_LOGIN_CREDENTIAL = "bitwarden_login_credential"
    BITWARDEN_SENSITIVE_INFORMATION = "bitwarden_sensitive_information"
    BITWARDEN_CREDIT_CARD_DATA = "bitwarden_credit_card_data"
    ONEPASSWORD = "onepassword"
    AZURE_VAULT_CREDENTIAL = "azure_vault_credential"
    CREDENTIAL = "credential"       # 자격증명
```

---

## 4. API 구조

### 4.1 REST API 엔드포인트

**Base URL**: `http://localhost:8000/api/v1`

| Method | Endpoint | 설명 |
|--------|----------|------|
| **Tasks** | | |
| POST | `/tasks` | 태스크 생성 및 실행 |
| GET | `/tasks/{task_id}` | 태스크 상태 조회 |
| GET | `/tasks/{task_id}/steps` | 태스크 스텝 목록 |
| **Workflows** | | |
| POST | `/workflows` | 워크플로우 생성 |
| GET | `/workflows` | 워크플로우 목록 |
| GET | `/workflows/{workflow_id}` | 워크플로우 상세 |
| POST | `/workflows/{workflow_id}/run` | 워크플로우 실행 |
| GET | `/workflows/runs/{run_id}` | 실행 상태 조회 |
| **Browser Sessions** | | |
| POST | `/browser_sessions` | 브라우저 세션 생성 |
| GET | `/browser_sessions/{session_id}` | 세션 상태 조회 |
| DELETE | `/browser_sessions/{session_id}` | 세션 종료 |
| **Credentials** | | |
| POST | `/credentials` | 자격증명 저장 |
| GET | `/credentials` | 자격증명 목록 |

### 4.2 WebSocket 스트리밍

```
ws://localhost:8000/api/v1/stream/{task_id}
```

- 실시간 태스크 진행 상황
- 스크린샷 스트리밍
- 액션 로그

### 4.3 인증

```bash
# API 키 헤더
X-API-KEY: your_api_key_here

# 또는 Bearer 토큰
Authorization: Bearer your_jwt_token
```

---

## 5. LLM 설정

### 5.1 지원 LLM 제공자

| 제공자 | 환경변수 접두사 | 지원 모델 |
|--------|----------------|-----------|
| OpenAI | `OPENAI_` | GPT-4o, GPT-4o-mini, GPT-5 |
| Anthropic | `ANTHROPIC_` | Claude 3.5/4/4.5 Sonnet, Opus, Haiku |
| Azure OpenAI | `AZURE_` | GPT-4, GPT-4o, GPT-5 |
| Google Gemini | `GEMINI_` | Gemini 2.5 Pro/Flash |
| AWS Bedrock | `BEDROCK_` | Claude (via Bedrock) |
| Ollama | `OLLAMA_` | 로컬 모델 (llama, qwen 등) |
| OpenRouter | `OPENROUTER_` | 다양한 모델 |
| Groq | `GROQ_` | llama, mixtral |

### 5.2 LLM 설정 예시

```bash
# .env 파일

# 주 LLM (복잡한 추론용)
ENABLE_ANTHROPIC=true
ANTHROPIC_API_KEY="sk-ant-api03-..."
LLM_KEY="ANTHROPIC_CLAUDE4.5_HAIKU"

# 보조 LLM (간단한 작업용)
ENABLE_OPENAI=true
OPENAI_API_KEY="sk-..."
SECONDARY_LLM_KEY="OPENAI_GPT4_1_MINI"
```

### 5.3 LLM Key 목록

```python
# OpenAI
"OPENAI_GPT4O"
"OPENAI_GPT4O_MINI"
"OPENAI_GPT4_1"
"OPENAI_GPT4_1_MINI"
"OPENAI_GPT5"

# Anthropic
"ANTHROPIC_CLAUDE3.5_SONNET"
"ANTHROPIC_CLAUDE3.7_SONNET"
"ANTHROPIC_CLAUDE4_SONNET"
"ANTHROPIC_CLAUDE4.5_SONNET"
"ANTHROPIC_CLAUDE4.5_HAIKU"
"ANTHROPIC_CLAUDE4_OPUS"

# Azure
"AZURE_OPENAI"
"AZURE_OPENAI_GPT4O_MINI"
"AZURE_OPENAI_GPT5"

# Gemini
"GEMINI_2.5_PRO"
"GEMINI_2.5_FLASH"

# Ollama
"OLLAMA"
```

---

## 6. 브라우저 설정

### 6.1 브라우저 모드

#### Chromium Headful (기본값)

```bash
BROWSER_TYPE="chromium-headful"
```
- GUI가 보임
- 개발 및 디버깅에 적합

#### Chromium Headless

```bash
BROWSER_TYPE="chromium-headless"
```
- GUI 없음
- 서버 환경에 적합

#### CDP Connect (중요!)

```bash
BROWSER_TYPE="cdp-connect"
BROWSER_REMOTE_DEBUGGING_URL="http://127.0.0.1:9222"
```
- 기존 Chrome 브라우저에 연결
- **로그인 세션 유지 가능** (네이버, 구글 등)
- 쿠키, 확장프로그램 유지

### 6.2 CDP 모드 사용법

**1. Chrome 시작 스크립트 (`start-chrome-cdp.sh`):**

```bash
#!/bin/bash
# Windows (WSL)
"/mnt/c/Program Files/Google/Chrome/Application/chrome.exe" \
    --remote-debugging-port=9222 \
    --user-data-dir="C:\chrome-cdp-profile" \
    --no-first-run \
    --no-default-browser-check &

# macOS
# /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
#     --remote-debugging-port=9222 \
#     --user-data-dir="/Users/$(whoami)/chrome-cdp-profile" &
```

**2. 수동 로그인:**
```
1. Chrome 실행 후 네이버/구글 등에 수동 로그인
2. 로그인 상태가 유지됨
3. Skyvern이 해당 세션을 사용
```

### 6.3 브라우저 설정 옵션

```bash
# 타임아웃 설정 (밀리초)
BROWSER_ACTION_TIMEOUT_MS=5000
BROWSER_SCREENSHOT_TIMEOUT_MS=20000
BROWSER_LOADING_TIMEOUT_MS=60000

# 브라우저 크기
BROWSER_WIDTH=1920
BROWSER_HEIGHT=1080

# 로케일 및 타임존
BROWSER_LOCALE="ko-KR"
BROWSER_TIMEZONE="Asia/Seoul"

# 비디오 녹화 경로
VIDEO_PATH="./videos"
```

---

## 7. 데이터베이스

### 7.1 PostgreSQL 설정

```bash
# 기본 연결 문자열
DATABASE_STRING="postgresql+psycopg://skyvern:skyvern@localhost:5432/skyvern"

# Docker Compose 환경
DATABASE_STRING="postgresql+psycopg://skyvern:skyvern@postgres:5432/skyvern"

# Windows 환경 (asyncpg 사용)
DATABASE_STRING="postgresql+asyncpg://skyvern:skyvern@localhost:5432/skyvern"
```

### 7.2 마이그레이션

```bash
# 마이그레이션 실행
alembic upgrade head

# 새 마이그레이션 생성
alembic revision --autogenerate -m "description"

# 마이그레이션 히스토리 확인
alembic history
```

### 7.3 주요 테이블

| 테이블 | 설명 |
|--------|------|
| `organizations` | 조직/테넌트 |
| `workflows` | 워크플로우 정의 |
| `workflow_runs` | 워크플로우 실행 기록 |
| `workflow_run_blocks` | 블록 실행 기록 |
| `tasks` | 태스크 |
| `steps` | 태스크 스텝 |
| `artifacts` | 아티팩트 (스크린샷 등) |
| `actions` | 실행된 액션 |
| `browser_sessions` | 브라우저 세션 |
| `credentials` | 저장된 자격증명 |

---

## 8. 워크플로우 시스템

### 8.1 워크플로우 YAML 구조

```yaml
title: "워크플로우 제목"
description: "설명"

workflow_definition:
  version: 1

  parameters:
    - parameter_type: workflow
      key: param_name
      workflow_parameter_type: string
      default_value: "기본값"

    - parameter_type: output
      key: result
      description: "출력 결과"

  blocks:
    - block_type: navigation
      label: step_1
      url: "https://example.com"
      navigation_goal: |
        페이지에서 수행할 작업을 자연어로 설명합니다.
        예: 로그인 버튼을 클릭하고, 아이디와 비밀번호를 입력합니다.
      parameter_keys:
        - param_name

    - block_type: extraction
      label: step_2
      data_extraction_goal: |
        추출할 데이터를 설명합니다.
      data_schema:
        type: object
        properties:
          title:
            type: string
          price:
            type: number
```

### 8.2 블록 타입 상세

#### navigation 블록

```yaml
- block_type: navigation
  label: navigate_to_page
  url: "https://example.com"
  navigation_goal: "로그인 페이지로 이동하여 로그인합니다."
  max_steps_per_run: 10
  parameter_keys:
    - username
    - password
```

#### extraction 블록

```yaml
- block_type: extraction
  label: extract_data
  data_extraction_goal: "상품 목록에서 이름과 가격을 추출합니다."
  data_schema:
    type: array
    items:
      type: object
      properties:
        name:
          type: string
        price:
          type: number
```

#### for_loop 블록

```yaml
- block_type: for_loop
  label: process_items
  loop_over_parameter_key: items
  loop_variable_reference: current_item
  loop_blocks:
    - block_type: navigation
      label: process_single_item
      navigation_goal: "현재 아이템을 처리합니다: {{current_item}}"
```

#### file_upload 블록

```yaml
- block_type: file_upload
  label: upload_image
  storage_type: local
  path: "/path/to/file.jpg"
```

### 8.3 실제 워크플로우 예시: 네이버 블로그 포스팅

```yaml
# docs/workflows/naver_blog_posting.yaml 참조

title: "Naver Blog Post with Images and Links"
workflow_definition:
  version: 1

  parameters:
    - parameter_type: workflow
      key: blog_title
      workflow_parameter_type: string
      default_value: "Skyvern 테스트 포스팅"

    - parameter_type: workflow
      key: image_link_url
      workflow_parameter_type: string
      default_value: "https://kt-cctv.ai.kr/"

  blocks:
    - block_type: navigation
      label: navigate_to_blog_editor
      url: "https://blog.naver.com/{{naver_id}}/postwrite"
      navigation_goal: "네이버 블로그 에디터로 이동합니다."

    - block_type: action
      label: input_title
      navigation_goal: "제목 입력란에 {{blog_title}}을 입력합니다."

    # ... 추가 블록
```

---

## 9. 현재 환경 설정

### 9.1 활성화된 설정 (.env)

```bash
# 환경
ENV=local

# LLM 설정
ENABLE_OPENAI=true
OPENAI_API_KEY="sk-svcacct-..."

ENABLE_ANTHROPIC=true
ANTHROPIC_API_KEY="sk-ant-api03-..."

LLM_KEY="ANTHROPIC_CLAUDE4.5_HAIKU"
SECONDARY_LLM_KEY="OPENAI_GPT4_1_MINI"

# 브라우저 설정 (CDP 모드)
BROWSER_TYPE="cdp-connect"
BROWSER_REMOTE_DEBUGGING_URL="http://127.0.0.1:9222"

# 데이터베이스
DATABASE_STRING="postgresql+psycopg://skyvern:skyvern@localhost:15432/skyvern"

# 서버
PORT=8000
```

### 9.2 테스트된 기능

| 기능 | 상태 | 비고 |
|------|------|------|
| CDP 브라우저 연결 | ✅ 작동 | Chrome 9222 포트 |
| 네이버 로그인 유지 | ✅ 작동 | CDP 세션 사용 |
| 네이버 블로그 포스팅 | ✅ 작동 | 텍스트, 이미지, 링크 |
| 이미지 업로드 | ✅ 작동 | WSL 경로 변환 필요 |
| 이미지 링크 추가 | ✅ 작동 | SmartEditor 툴바 사용 |
| Anthropic Claude | ✅ 작동 | Claude 4.5 Haiku |
| OpenAI GPT | ✅ 작동 | GPT-4.1-mini |

---

## 10. 개발 환경 설정

### 10.1 필수 요구사항

- **Python**: 3.11 ~ 3.13
- **Node.js**: 18+ (프론트엔드 빌드 시)
- **PostgreSQL**: 14+
- **uv**: Python 패키지 관리자

### 10.2 설치 방법

```bash
# 1. 저장소 클론
git clone https://github.com/shinjadong/skyvern-back.git
cd skyvern-back

# 2. Python 의존성 설치
uv sync

# 3. 환경 변수 설정
cp .env.example .env
# .env 파일 편집하여 API 키 등 설정

# 4. 데이터베이스 마이그레이션
alembic upgrade head

# 5. Playwright 브라우저 설치
playwright install chromium

# 6. 서버 실행
skyvern run server
```

### 10.3 Docker 실행

```bash
# Docker Compose로 전체 스택 실행
docker-compose up -d

# 로그 확인
docker-compose logs -f skyvern
```

### 10.4 개발 명령어

```bash
# 코드 품질
ruff check                    # 린트
ruff format                   # 포맷
mypy skyvern                  # 타입 체크

# 테스트
pytest tests/                 # 전체 테스트
pytest tests/unit_tests/      # 단위 테스트

# Pre-commit 훅
pre-commit run --all-files
```

---

## 11. 외부 통합

### 11.1 LangChain 통합

```python
# integrations/langchain/
from skyvern_langchain import SkyvernAgent

agent = SkyvernAgent(api_key="your_key")
result = await agent.run("https://example.com", "폼을 작성합니다.")
```

### 11.2 LlamaIndex 통합

```python
# integrations/llama_index/
from skyvern_llamaindex import SkyvernTool

tool = SkyvernTool(api_key="your_key")
```

### 11.3 MCP (Model Context Protocol) 서버

```bash
# MCP 서버 실행
skyvern mcp
```

MCP를 통해 Claude Desktop, Continue 등에서 Skyvern 사용 가능.

### 11.4 Make.com / n8n 통합

- `integrations/make/`: Make.com 블루프린트
- `integrations/n8n/`: n8n 워크플로우

---

## 12. 알려진 이슈 및 해결책

### 12.1 네이버 SmartEditor 입력 문제

**문제**: `fill()` 메서드가 contenteditable 요소에서 작동하지 않음

**원인**: 네이버 SmartEditor는 일반 `<input>`이 아닌 `contenteditable` 사용

**해결책**: `pressSequentially()` 메서드 사용
```python
# 요소 클릭 후 순차적으로 문자 입력
await element.press_sequentially("텍스트 내용")
```

### 12.2 WSL 경로 변환

**문제**: Windows 경로를 WSL에서 인식하지 못함

**해결책**: 경로 변환
```
Windows: C:\Users\username\file.jpg
WSL:     /mnt/c/Users/username/file.jpg
```

### 12.3 CDP 연결 실패

**문제**: Chrome에 연결할 수 없음

**확인사항**:
1. Chrome이 `--remote-debugging-port=9222`로 실행되었는지 확인
2. `http://127.0.0.1:9222/json` 접속하여 응답 확인
3. 방화벽/포트 차단 확인

### 12.4 데이터베이스 연결 오류

**Windows 환경**:
```bash
# psycopg 대신 asyncpg 사용
DATABASE_STRING="postgresql+asyncpg://skyvern:skyvern@localhost:5432/skyvern"
```

---

## 13. 디렉토리 구조 요약

```
skyvern-back/
├── skyvern/                    # 핵심 Python 패키지
│   ├── __init__.py
│   ├── __main__.py             # python -m skyvern
│   ├── config.py               # 설정 (Settings 클래스)
│   ├── constants.py            # 상수
│   ├── analytics.py            # 분석/텔레메트리
│   │
│   ├── cli/                    # CLI 명령어
│   ├── client/                 # Python SDK 클라이언트
│   ├── core/                   # 핵심 유틸리티
│   ├── errors/                 # 예외 정의
│   ├── forge/                  # 핵심 엔진
│   │   ├── prompts/            # LLM 프롬프트 템플릿
│   │   └── sdk/                # SDK 구현
│   ├── library/                # 라이브러리 유틸리티
│   ├── schemas/                # Pydantic 스키마
│   ├── services/               # 비즈니스 서비스
│   ├── utils/                  # 유틸리티
│   └── webeye/                 # 브라우저 자동화
│
├── alembic/                    # DB 마이그레이션
│   └── versions/               # 마이그레이션 파일
│
├── tests/                      # 테스트
│   └── unit_tests/
│
├── integrations/               # 외부 통합
│   ├── langchain/
│   ├── llama_index/
│   ├── mcp/
│   └── make/
│
├── scripts/                    # 유틸리티 스크립트
├── kubernetes-deployment/      # K8s 배포 매니페스트
├── bitwarden-cli-server/       # Bitwarden CLI 서버
│
├── pyproject.toml              # Python 프로젝트 설정
├── uv.lock                     # 의존성 락 파일
├── alembic.ini                 # Alembic 설정
├── docker-compose.yml          # Docker Compose
├── Dockerfile                  # Docker 이미지
├── .env.example                # 환경변수 템플릿
│
├── CLAUDE.md                   # Claude Code 가이드
├── PROJECT_CONTEXT.md          # 이 문서
└── README.md                   # 기본 README
```

---

## 14. 다음 단계 및 로드맵

### 14.1 즉시 사용 가능한 작업

1. **워크플로우 실행**: 기존 YAML 워크플로우 실행
2. **API 호출**: REST API를 통한 태스크 생성
3. **브라우저 자동화**: CDP 모드로 로그인 세션 유지

### 14.2 확장 가능한 영역

1. **새 워크플로우 개발**: 다른 사이트용 자동화 워크플로우
2. **커스텀 블록**: 새로운 블록 타입 개발
3. **LLM 최적화**: 프롬프트 튜닝, 모델 선택
4. **프로덕션 배포**: Kubernetes, 클라우드 배포

### 14.3 주의사항

- API 키는 절대 커밋하지 않음
- 프로덕션에서는 적절한 인증/인가 설정 필요
- 웹사이트 이용약관 준수 필요

---

## 15. 참고 자료

### 공식 문서
- [Skyvern GitHub](https://github.com/Skyvern-AI/skyvern)
- [Skyvern Docs](https://docs.skyvern.com)

### 관련 기술
- [Playwright Python](https://playwright.dev/python/)
- [FastAPI](https://fastapi.tiangolo.com/)
- [LiteLLM](https://docs.litellm.ai/)
- [Anthropic Claude](https://docs.anthropic.com/)
- [OpenAI API](https://platform.openai.com/docs/)

---

**문서 작성자**: Claude Code
**마지막 업데이트**: 2025-12-02
**버전**: 1.0.0
