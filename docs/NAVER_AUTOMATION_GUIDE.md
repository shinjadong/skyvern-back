# 네이버 자동화 완벽 가이드

> **문서 작성일**: 2025-12-02
> **테스트 완료**: 2025-12-02
> **테스트 결과 URL**: https://blog.naver.com/PostView.naver?blogId=shinws8908&logNo=224094913446

---

## 목차

1. [개요](#1-개요)
2. [사전 준비](#2-사전-준비)
3. [CDP 브라우저 설정](#3-cdp-브라우저-설정)
4. [네이버 로그인 세션 유지](#4-네이버-로그인-세션-유지)
5. [네이버 블로그 자동화](#5-네이버-블로그-자동화)
6. [SmartEditor 상호작용](#6-smarteditor-상호작용)
7. [이미지 업로드](#7-이미지-업로드)
8. [이미지 링크 추가](#8-이미지-링크-추가)
9. [포스트 발행](#9-포스트-발행)
10. [워크플로우 YAML](#10-워크플로우-yaml)
11. [Playwright 코드 예시](#11-playwright-코드-예시)
12. [문제 해결](#12-문제-해결)
13. [보안 고려사항](#13-보안-고려사항)

---

## 1. 개요

### 1.1 네이버 자동화가 필요한 이유

네이버는 한국에서 가장 큰 포털 서비스로, 다음과 같은 자동화 수요가 있습니다:

- **블로그 포스팅**: 마케팅, 콘텐츠 발행
- **카페 관리**: 게시글 작성, 댓글 관리
- **쇼핑/스마트스토어**: 상품 등록, 주문 처리
- **메일/캘린더**: 업무 자동화

### 1.2 네이버 자동화의 어려움

| 문제 | 원인 | 해결책 |
|------|------|--------|
| 로그인 차단 | 자동 로그인 감지 | CDP 모드로 수동 로그인 세션 유지 |
| CAPTCHA | 봇 감지 | 기존 로그인 세션 재사용 |
| 동적 UI | JavaScript 기반 렌더링 | Playwright 사용 |
| SmartEditor | contenteditable 사용 | `pressSequentially()` 사용 |
| 2차 인증 | OTP/인증서 | 수동 인증 후 세션 유지 |

### 1.3 테스트 완료된 기능

| 기능 | 상태 | 비고 |
|------|------|------|
| 로그인 세션 유지 | ✅ | CDP 모드 |
| 블로그 에디터 접근 | ✅ | |
| 제목 입력 | ✅ | pressSequentially |
| 본문 입력 | ✅ | pressSequentially |
| 이미지 업로드 | ✅ | 다중 이미지 지원 |
| 이미지 레이아웃 선택 | ✅ | 개별사진 |
| 이미지 링크 추가 | ✅ | |
| 포스트 발행 | ✅ | |

---

## 2. 사전 준비

### 2.1 필요한 계정 정보

```yaml
네이버 계정:
  아이디: shinws8908
  비밀번호: Tidtyd8908@@
  블로그 URL: https://blog.naver.com/shinws8908
```

### 2.2 시스템 요구사항

- **OS**: Windows 10/11 (WSL2), macOS, Linux
- **Python**: 3.11+
- **Chrome**: 최신 버전
- **Skyvern**: 설치 및 설정 완료

### 2.3 환경 변수 설정

```bash
# .env 파일

# 브라우저 설정 (CDP 모드 필수)
BROWSER_TYPE="cdp-connect"
BROWSER_REMOTE_DEBUGGING_URL="http://127.0.0.1:9222"

# LLM 설정 (한국어 지원 모델 권장)
ENABLE_ANTHROPIC=true
ANTHROPIC_API_KEY="your_key"
LLM_KEY="ANTHROPIC_CLAUDE4.5_HAIKU"

# 기타 설정
BROWSER_LOCALE="ko-KR"
BROWSER_TIMEZONE="Asia/Seoul"
```

---

## 3. CDP 브라우저 설정

### 3.1 CDP (Chrome DevTools Protocol) 이란?

CDP는 Chrome 브라우저와 프로그래밍 방식으로 통신하는 프로토콜입니다.
**기존 Chrome 프로필(로그인 정보, 쿠키, 확장프로그램)을 유지**하면서 자동화할 수 있습니다.

### 3.2 Chrome CDP 모드 시작 스크립트

**Windows (WSL 환경) - `start-chrome-cdp.sh`:**

```bash
#!/bin/bash
# Chrome CDP 모드 시작 스크립트

CHROME_PATH="/mnt/c/Program Files/Google/Chrome/Application/chrome.exe"
USER_DATA_DIR="C:\\chrome-cdp-profile"
DEBUG_PORT=9222

# 기존 Chrome 프로세스 확인
if pgrep -f "remote-debugging-port=$DEBUG_PORT" > /dev/null; then
    echo "Chrome CDP가 이미 실행 중입니다. (포트: $DEBUG_PORT)"
    exit 0
fi

# Chrome 시작
"$CHROME_PATH" \
    --remote-debugging-port=$DEBUG_PORT \
    --user-data-dir="$USER_DATA_DIR" \
    --no-first-run \
    --no-default-browser-check \
    --disable-background-timer-throttling \
    --disable-backgrounding-occluded-windows \
    --disable-renderer-backgrounding \
    &

echo "Chrome CDP 모드가 시작되었습니다."
echo "디버그 포트: $DEBUG_PORT"
echo "프로필 경로: $USER_DATA_DIR"
```

**macOS:**

```bash
#!/bin/bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
    --remote-debugging-port=9222 \
    --user-data-dir="$HOME/chrome-cdp-profile" \
    --no-first-run \
    --no-default-browser-check \
    &
```

**Linux:**

```bash
#!/bin/bash
google-chrome \
    --remote-debugging-port=9222 \
    --user-data-dir="$HOME/chrome-cdp-profile" \
    --no-first-run \
    --no-default-browser-check \
    &
```

### 3.3 CDP 연결 확인

```bash
# CDP 엔드포인트 확인
curl http://127.0.0.1:9222/json

# 예상 응답
[
  {
    "id": "...",
    "type": "page",
    "title": "새 탭",
    "url": "chrome://newtab/",
    "webSocketDebuggerUrl": "ws://127.0.0.1:9222/devtools/page/..."
  }
]
```

---

## 4. 네이버 로그인 세션 유지

### 4.1 최초 로그인 (수동)

1. CDP 모드로 Chrome 시작
2. `https://naver.com` 접속
3. 수동으로 로그인 (ID: shinws8908, PW: Tidtyd8908@@)
4. 2차 인증 완료 (필요시)
5. "로그인 상태 유지" 체크

### 4.2 세션 유지 원리

```
┌─────────────────┐
│  Chrome CDP     │
│  (9222 포트)    │
├─────────────────┤
│  쿠키 저장      │  ← NID_AUT, NID_SES 등
│  프로필 유지    │  ← chrome-cdp-profile/
│  확장프로그램   │
└─────────────────┘
         │
         ▼
┌─────────────────┐
│   Skyvern      │
│   (CDP 연결)    │
├─────────────────┤
│  기존 세션 사용 │
│  재로그인 불필요│
└─────────────────┘
```

### 4.3 세션 만료 시 대응

네이버 세션은 보통 **2주** 정도 유지됩니다. 만료 시:

1. Chrome CDP 모드 실행
2. 네이버 접속하여 재로그인
3. 로그인 상태 유지 체크
4. 이후 자동화 작업 재개

---

## 5. 네이버 블로그 자동화

### 5.1 블로그 에디터 URL 구조

```
https://blog.naver.com/{블로그ID}/postwrite

예시:
https://blog.naver.com/shinws8908/postwrite
```

### 5.2 에디터 페이지 구조

```
┌─────────────────────────────────────────────────────────────┐
│  [발행] 버튼                                    [임시저장]  │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │  제목 입력란 (contenteditable)                      │   │
│  │  placeholder: "제목"                                 │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  [사진] [동영상] [스티커] [지도] [링크] [파일] [일정] ...   │  ← 툴바
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  본문 입력란 (contenteditable)                      │   │
│  │  placeholder: "이곳을 눌러 글을 작성해 주세요"       │   │
│  │                                                      │   │
│  │  [이미지]  [이미지]                                  │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 자동화 흐름

```
1. 블로그 에디터 페이지 로드
   └── https://blog.naver.com/shinws8908/postwrite

2. 제목 입력
   └── 제목 영역 클릭 → pressSequentially("제목 텍스트")

3. 본문 입력
   └── 본문 영역 클릭 → pressSequentially("본문 텍스트")

4. 이미지 추가
   └── [사진] 버튼 클릭 → 파일 선택 → 레이아웃 선택

5. 이미지 링크 추가 (선택)
   └── 이미지 클릭 → [링크] 버튼 → URL 입력

6. 발행
   └── [발행] 버튼 클릭 → 설정 확인 → 최종 발행
```

---

## 6. SmartEditor 상호작용

### 6.1 SmartEditor 특성

네이버 블로그는 **SmartEditor**라는 자체 에디터를 사용합니다.

| 특성 | 설명 |
|------|------|
| 기술 | contenteditable 기반 |
| 입력 방식 | 일반 input이 아님 |
| 문제점 | `fill()` 메서드 작동 안 함 |
| 해결책 | `pressSequentially()` 사용 |

### 6.2 contenteditable vs input

**일반 input:**
```html
<input type="text" value="텍스트">
```
→ `element.fill("텍스트")` 작동

**contenteditable (SmartEditor):**
```html
<div contenteditable="true">
  <p><br></p>
</div>
```
→ `element.fill()` 작동 안 함
→ `element.press_sequentially("텍스트")` 사용 필요

### 6.3 제목 입력 영역

**HTML 구조:**
```html
<div class="se-title-area">
  <div class="se-title-text">
    <div class="se-content" contenteditable="true">
      <p class="se-text-paragraph">
        <span class="se-text-content">제목 텍스트</span>
      </p>
    </div>
  </div>
</div>
```

**선택자:**
```javascript
// 제목 영역 선택
'.se-title-text .se-content'
// 또는
'[placeholder="제목"]'
```

### 6.4 본문 입력 영역

**HTML 구조:**
```html
<div class="se-main-editor">
  <div class="se-content" contenteditable="true">
    <p class="se-text-paragraph">
      <span class="se-text-content">본문 텍스트</span>
    </p>
  </div>
</div>
```

**선택자:**
```javascript
// 본문 영역 선택
'.se-main-editor .se-content'
// 또는
'[data-placeholder="이곳을 눌러 글을 작성해 주세요"]'
```

---

## 7. 이미지 업로드

### 7.1 이미지 업로드 흐름

```
1. [사진] 버튼 클릭
   └── 툴바에서 "사진 추가" 또는 카메라 아이콘

2. 파일 선택 대화상자
   └── file input이 자동으로 열림

3. 파일 선택
   └── browser_file_upload 사용

4. 레이아웃 선택
   └── "개별사진", "나란히 2장", "격자형" 등

5. 업로드 완료
   └── 이미지가 본문에 삽입됨
```

### 7.2 Windows/WSL 경로 변환

**Windows 경로:**
```
C:\Users\tlswk\OneDrive\Desktop\템플릿_이미지\image.jpg
```

**WSL 경로 (Skyvern에서 사용):**
```
/mnt/c/Users/tlswk/OneDrive/Desktop/템플릿_이미지/image.jpg
```

**변환 규칙:**
```
C:\  →  /mnt/c/
D:\  →  /mnt/d/
백슬래시(\)  →  슬래시(/)
```

### 7.3 테스트된 이미지 경로

```yaml
이미지 1:
  Windows: C:\Users\tlswk\OneDrive\Desktop\템플릿_이미지\템플릿_이미지\3번 템플릿\템플릿_01_위협.JPG
  WSL: /mnt/c/Users/tlswk/OneDrive/Desktop/템플릿_이미지/템플릿_이미지/3번 템플릿/템플릿_01_위협.JPG

이미지 2:
  Windows: C:\Users\tlswk\OneDrive\Desktop\템플릿_이미지\템플릿_이미지\3번 템플릿\cctv-3.JPG
  WSL: /mnt/c/Users/tlswk/OneDrive/Desktop/템플릿_이미지/템플릿_이미지/3번 템플릿/cctv-3.JPG
```

### 7.4 레이아웃 옵션

| 옵션 | 설명 |
|------|------|
| 개별사진 | 각 이미지를 개별 블록으로 |
| 나란히 2장 | 2장을 좌우 배치 |
| 나란히 3장 | 3장을 좌우 배치 |
| 격자형 | 그리드 레이아웃 |

---

## 8. 이미지 링크 추가

### 8.1 이미지 링크 추가 흐름

```
1. 이미지 클릭 (선택)
   └── 이미지가 선택되면 테두리 표시

2. 툴바 표시
   └── 이미지 위에 편집 툴바 나타남

3. [링크] 버튼 클릭
   └── "링크 입력 열기" 또는 체인 아이콘

4. URL 입력
   └── 텍스트 필드에 URL 입력

5. 확인
   └── "링크 입력" 버튼 클릭

6. 완료
   └── 이미지에 "링크" 표시 나타남
```

### 8.2 이미지 선택 시 나타나는 툴바

```
┌─────────────────────────────────────────┐
│  [이미지]                               │
├─────────────────────────────────────────┤
│  [정렬] [크기] [여백] [링크] [삭제]     │  ← 툴바
└─────────────────────────────────────────┘
```

### 8.3 링크 입력 UI

```
┌─────────────────────────────────────────┐
│  링크 입력                              │
├─────────────────────────────────────────┤
│  URL: [https://kt-cctv.ai.kr/        ] │
│                                         │
│  [취소]                    [링크 입력]  │
└─────────────────────────────────────────┘
```

### 8.4 테스트된 링크

```yaml
URL: https://kt-cctv.ai.kr/
상태: 정상 작동
용도: KT CCTV AI 서비스 랜딩 페이지
```

**주의**: 초기 테스트에서 `https://kt-ccvt.ai.kr/` (오타)로 입력하여 DNS 오류 발생.
올바른 URL은 `https://kt-cctv.ai.kr/` (cctv)입니다.

---

## 9. 포스트 발행

### 9.1 발행 흐름

```
1. [발행] 버튼 클릭 (에디터 상단)
   └── 발행 설정 팝업 표시

2. 발행 설정 확인
   ├── 공개 설정: 전체 공개 / 이웃 공개 / 비공개
   ├── 카테고리: 선택
   ├── 태그: 입력 (선택)
   └── 발행 시간: 즉시 / 예약

3. 최종 [발행] 버튼 클릭
   └── 팝업 내 발행 버튼

4. 발행 완료
   └── 포스트 보기 페이지로 이동
```

### 9.2 발행 설정 팝업

```
┌─────────────────────────────────────────────────┐
│  발행                                     [X]   │
├─────────────────────────────────────────────────┤
│                                                 │
│  공개 설정                                      │
│  ● 전체 공개  ○ 이웃 공개  ○ 비공개            │
│                                                 │
│  카테고리                                       │
│  [▼ 카테고리 선택                           ]   │
│                                                 │
│  태그                                           │
│  [태그를 입력하세요                          ]   │
│                                                 │
├─────────────────────────────────────────────────┤
│                              [취소]   [발행]    │
└─────────────────────────────────────────────────┘
```

### 9.3 발행 완료 후 URL

```
https://blog.naver.com/PostView.naver?blogId=shinws8908&logNo=224094913446

구조:
- blogId: 블로그 ID (shinws8908)
- logNo: 포스트 번호 (224094913446)
```

---

## 10. 워크플로우 YAML

### 10.1 전체 워크플로우

```yaml
# docs/workflows/naver_blog_posting.yaml

title: "Naver Blog Post with Images and Links"
description: |
  네이버 블로그에 제목, 본문, 이미지, 이미지 링크를 포함한 포스트를 발행합니다.
  SmartEditor의 contenteditable 요소를 사용하여 텍스트를 입력합니다.

proxy_location: null
persist_browser_session: true

workflow_definition:
  version: 1

  parameters:
    # 입력 파라미터
    - parameter_type: workflow
      key: naver_id
      description: "네이버 블로그 ID"
      workflow_parameter_type: string
      default_value: "shinws8908"

    - parameter_type: workflow
      key: blog_title
      description: "블로그 포스트 제목"
      workflow_parameter_type: string
      default_value: "Skyvern 테스트 포스팅"

    - parameter_type: workflow
      key: blog_body
      description: "블로그 포스트 본문"
      workflow_parameter_type: string
      default_value: "이 글은 Skyvern AI 자동화 테스트로 작성되었습니다."

    - parameter_type: workflow
      key: image_paths
      description: "업로드할 이미지 경로 목록 (WSL 경로)"
      workflow_parameter_type: json
      default_value:
        - "/mnt/c/Users/tlswk/OneDrive/Desktop/템플릿_이미지/템플릿_이미지/3번 템플릿/템플릿_01_위협.JPG"
        - "/mnt/c/Users/tlswk/OneDrive/Desktop/템플릿_이미지/템플릿_이미지/3번 템플릿/cctv-3.JPG"

    - parameter_type: workflow
      key: image_link_url
      description: "이미지에 연결할 URL"
      workflow_parameter_type: string
      default_value: "https://kt-cctv.ai.kr/"

    # 출력 파라미터
    - parameter_type: output
      key: published_url
      description: "발행된 블로그 포스트 URL"

  blocks:
    # Step 1: 블로그 에디터로 이동
    - block_type: navigation
      label: navigate_to_blog_editor
      url: "https://blog.naver.com/{{naver_id}}/postwrite"
      navigation_goal: |
        네이버 블로그 에디터 페이지로 이동합니다.
        제목 입력란과 본문 입력란이 보이면 성공입니다.
      max_steps_per_run: 5
      parameter_keys:
        - naver_id

    # Step 2: 제목 입력
    - block_type: action
      label: input_title
      navigation_goal: |
        블로그 제목 입력란을 클릭하고 제목을 입력합니다.
        SmartEditor는 contenteditable을 사용하므로
        클릭 후 텍스트를 순차적으로 입력합니다.

        제목: {{blog_title}}
      parameter_keys:
        - blog_title

    # Step 3: 본문 입력
    - block_type: action
      label: input_body
      navigation_goal: |
        블로그 본문 입력란에 내용을 입력합니다.

        본문: {{blog_body}}
      parameter_keys:
        - blog_body

    # Step 4: 이미지 업로드
    - block_type: action
      label: click_image_upload
      navigation_goal: |
        에디터 툴바에서 "사진" 또는 "사진 추가" 버튼을 클릭합니다.

    # Step 5: 파일 선택 및 업로드
    - block_type: file_upload
      label: upload_images
      navigation_goal: |
        이미지 파일을 업로드합니다.
        이미지 경로: {{image_paths}}
        업로드 후 "개별사진" 레이아웃을 선택합니다.
      parameter_keys:
        - image_paths

    # Step 6: 첫 번째 이미지에 링크 추가
    - block_type: action
      label: add_link_to_first_image
      navigation_goal: |
        첫 번째 이미지를 클릭하여 선택합니다.
        툴바에서 "링크 입력 열기" 버튼을 클릭합니다.
        URL 입력란에 다음 URL을 입력합니다: {{image_link_url}}
        "링크 입력" 버튼을 클릭합니다.
      parameter_keys:
        - image_link_url

    # Step 7: 두 번째 이미지에 링크 추가
    - block_type: action
      label: add_link_to_second_image
      navigation_goal: |
        두 번째 이미지를 클릭하여 선택합니다.
        툴바에서 "링크 입력 열기" 버튼을 클릭합니다.
        URL 입력란에 다음 URL을 입력합니다: {{image_link_url}}
        "링크 입력" 버튼을 클릭합니다.
      parameter_keys:
        - image_link_url

    # Step 8: 발행 버튼 클릭
    - block_type: action
      label: click_publish_button
      navigation_goal: |
        에디터 상단의 "발행" 버튼을 클릭합니다.
        발행 설정 팝업이 나타납니다.

    # Step 9: 최종 발행
    - block_type: navigation
      label: confirm_publish
      navigation_goal: |
        발행 설정 팝업에서 최종 "발행" 버튼을 클릭합니다.
        발행이 완료되면 블로그 포스트 URL을 확인합니다.
      complete_criterion: "URL이 blog.naver.com/PostView.naver 형식이면 발행 완료"
      max_steps_per_run: 5
```

### 10.2 파라미터 상세

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `naver_id` | string | shinws8908 | 네이버 블로그 ID |
| `blog_title` | string | Skyvern 테스트 포스팅 | 포스트 제목 |
| `blog_body` | string | 이 글은... | 포스트 본문 |
| `image_paths` | json (array) | [...] | 이미지 경로 목록 |
| `image_link_url` | string | https://kt-cctv.ai.kr/ | 이미지 클릭 시 이동 URL |
| `published_url` | output | - | 발행된 포스트 URL |

---

## 11. Playwright 코드 예시

### 11.1 CDP 연결

```python
from playwright.async_api import async_playwright

async def connect_to_chrome_cdp():
    async with async_playwright() as p:
        # CDP로 기존 Chrome에 연결
        browser = await p.chromium.connect_over_cdp(
            "http://127.0.0.1:9222"
        )

        # 기존 컨텍스트 사용
        context = browser.contexts[0]
        page = context.pages[0]

        return browser, context, page
```

### 11.2 네이버 블로그 에디터 접속

```python
async def navigate_to_blog_editor(page, naver_id="shinws8908"):
    url = f"https://blog.naver.com/{naver_id}/postwrite"
    await page.goto(url)

    # 에디터 로드 대기
    await page.wait_for_selector('.se-title-text', timeout=10000)
    print("블로그 에디터 로드 완료")
```

### 11.3 제목 입력 (pressSequentially)

```python
async def input_title(page, title):
    # 제목 영역 클릭
    title_element = page.locator('.se-title-text .se-content')
    await title_element.click()

    # 기존 텍스트 선택 후 삭제
    await page.keyboard.press('Control+A')
    await page.keyboard.press('Backspace')

    # 순차적으로 문자 입력 (contenteditable용)
    await title_element.press_sequentially(title, delay=50)
    print(f"제목 입력 완료: {title}")
```

### 11.4 본문 입력

```python
async def input_body(page, body):
    # 본문 영역 클릭
    body_element = page.locator('.se-main-editor .se-content')
    await body_element.click()

    # 순차적으로 문자 입력
    await body_element.press_sequentially(body, delay=30)
    print(f"본문 입력 완료: {body}")
```

### 11.5 이미지 업로드

```python
async def upload_images(page, image_paths):
    # 사진 추가 버튼 클릭
    photo_button = page.locator('button:has-text("사진")')
    await photo_button.click()

    # 파일 선택 대화상자 처리
    async with page.expect_file_chooser() as fc_info:
        # "내 PC" 또는 업로드 영역 클릭
        upload_area = page.locator('.se-image-uploader')
        await upload_area.click()

    file_chooser = await fc_info.value
    await file_chooser.set_files(image_paths)

    # 레이아웃 선택 (개별사진)
    await page.locator('button:has-text("개별사진")').click()

    print(f"이미지 업로드 완료: {len(image_paths)}개")
```

### 11.6 이미지 링크 추가

```python
async def add_link_to_image(page, image_index, url):
    # 이미지 선택
    images = page.locator('.se-image-resource')
    await images.nth(image_index).click()

    # 링크 버튼 클릭
    link_button = page.locator('button:has-text("링크 입력 열기")')
    await link_button.click()

    # URL 입력
    url_input = page.locator('input[placeholder*="URL"]')
    await url_input.fill(url)

    # 링크 입력 확인
    confirm_button = page.locator('button:has-text("링크 입력")')
    await confirm_button.click()

    print(f"이미지 {image_index}에 링크 추가: {url}")
```

### 11.7 포스트 발행

```python
async def publish_post(page):
    # 발행 버튼 클릭
    publish_button = page.locator('button:has-text("발행")')
    await publish_button.click()

    # 발행 팝업에서 최종 발행
    await page.wait_for_selector('.publish-popup', timeout=5000)
    final_publish = page.locator('.publish-popup button:has-text("발행")')
    await final_publish.click()

    # URL 변경 대기
    await page.wait_for_url('**/PostView.naver**', timeout=10000)

    published_url = page.url
    print(f"발행 완료: {published_url}")
    return published_url
```

### 11.8 전체 자동화 코드

```python
import asyncio
from playwright.async_api import async_playwright

async def naver_blog_post_automation():
    async with async_playwright() as p:
        # CDP 연결
        browser = await p.chromium.connect_over_cdp("http://127.0.0.1:9222")
        context = browser.contexts[0]
        page = await context.new_page()

        try:
            # 1. 에디터 접속
            await page.goto("https://blog.naver.com/shinws8908/postwrite")
            await page.wait_for_selector('.se-title-text', timeout=10000)

            # 2. 제목 입력
            title_el = page.locator('.se-title-text .se-content')
            await title_el.click()
            await title_el.press_sequentially("Skyvern 테스트 포스팅", delay=50)

            # 3. 본문 입력
            body_el = page.locator('.se-main-editor .se-content')
            await body_el.click()
            await body_el.press_sequentially(
                "이 글은 Skyvern AI 자동화 테스트로 작성되었습니다.",
                delay=30
            )

            # 4. 이미지 업로드
            await page.locator('button:has-text("사진")').click()

            async with page.expect_file_chooser() as fc:
                await page.locator('.se-image-uploader').click()

            file_chooser = await fc.value
            await file_chooser.set_files([
                "/mnt/c/Users/tlswk/OneDrive/Desktop/템플릿_이미지/템플릿_이미지/3번 템플릿/템플릿_01_위협.JPG",
                "/mnt/c/Users/tlswk/OneDrive/Desktop/템플릿_이미지/템플릿_이미지/3번 템플릿/cctv-3.JPG"
            ])

            await page.locator('button:has-text("개별사진")').click()
            await page.wait_for_timeout(2000)

            # 5. 이미지 링크 추가
            images = page.locator('.se-image-resource')
            for i in range(2):
                await images.nth(i).click()
                await page.locator('button:has-text("링크 입력 열기")').click()
                await page.locator('input[type="text"]').fill("https://kt-cctv.ai.kr/")
                await page.locator('button:has-text("링크 입력")').click()
                await page.wait_for_timeout(500)

            # 6. 발행
            await page.locator('button:has-text("발행")').first.click()
            await page.wait_for_timeout(1000)
            await page.locator('.publish-popup button:has-text("발행")').click()

            # 7. 결과 확인
            await page.wait_for_url('**/PostView.naver**', timeout=10000)
            print(f"발행 완료: {page.url}")

        finally:
            await page.close()

# 실행
asyncio.run(naver_blog_post_automation())
```

---

## 12. 문제 해결

### 12.1 로그인 세션 만료

**증상**: 블로그 에디터 접속 시 로그인 페이지로 리다이렉트

**해결**:
1. Chrome CDP 모드 종료
2. 새로 Chrome CDP 시작
3. 네이버 수동 로그인
4. "로그인 상태 유지" 체크
5. 자동화 재시작

### 12.2 제목/본문 입력 안됨

**증상**: `fill()` 사용 시 텍스트 입력 안됨

**원인**: SmartEditor는 contenteditable 사용

**해결**:
```python
# 잘못된 방법
await element.fill("텍스트")

# 올바른 방법
await element.press_sequentially("텍스트", delay=50)
```

### 12.3 이미지 업로드 실패

**증상**: 파일을 찾을 수 없음

**원인**: Windows 경로를 WSL에서 인식 못함

**해결**:
```python
# 잘못된 경로
"C:\\Users\\tlswk\\image.jpg"

# 올바른 경로 (WSL)
"/mnt/c/Users/tlswk/image.jpg"
```

### 12.4 이미지 링크 버튼 없음

**증상**: 이미지 클릭해도 링크 버튼이 안 보임

**원인**: 이미지가 선택되지 않음

**해결**:
1. 이미지 정확히 클릭 (중앙)
2. 선택 테두리 확인
3. 툴바가 나타날 때까지 대기
4. 스크롤하여 툴바 확인

### 12.5 발행 실패

**증상**: 발행 버튼 클릭해도 반응 없음

**가능한 원인**:
- 제목이 비어있음
- 본문이 비어있음
- 이미지 업로드 중
- 네트워크 오류

**해결**:
1. 필수 필드 입력 확인
2. 업로드 완료 대기
3. 재시도

### 12.6 CDP 연결 실패

**증상**: `connect_over_cdp` 오류

**확인사항**:
```bash
# Chrome이 실행 중인지 확인
curl http://127.0.0.1:9222/json

# 포트 사용 확인
netstat -an | grep 9222
```

**해결**:
1. Chrome 완전 종료
2. `--remote-debugging-port=9222` 옵션으로 재시작
3. 방화벽 확인

---

## 13. 보안 고려사항

### 13.1 계정 정보 보호

```yaml
# .env 파일에 저장 (절대 커밋하지 않음)
NAVER_ID=shinws8908
NAVER_PASSWORD=********

# .gitignore에 추가
.env
*.env
```

### 13.2 세션 보안

- CDP 포트(9222)는 localhost만 접근 허용
- 외부에서 접근 불가하도록 방화벽 설정
- 프로필 디렉토리 권한 제한

### 13.3 자동화 주의사항

- 네이버 이용약관 준수
- 과도한 요청 자제
- 적절한 딜레이 설정
- 캡차 발생 시 수동 처리

### 13.4 API 키 관리

```yaml
# 절대 커밋하지 않을 것
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```

---

## 14. 테스트 결과 증거

### 14.1 발행된 포스트

- **URL**: https://blog.naver.com/PostView.naver?blogId=shinws8908&logNo=224094913446
- **제목**: Skyvern 테스트 포스팅
- **본문**: 이 글은 Skyvern AI 자동화 테스트로 작성되었습니다.
- **이미지**: 2장 (템플릿_01_위협.JPG, cctv-3.JPG)
- **이미지 링크**: https://kt-cctv.ai.kr/

### 14.2 실행 로그 요약

```
1. Chrome CDP 연결 성공 (포트 9222)
2. 블로그 에디터 로드 완료
3. 제목 입력 완료: "Skyvern 테스트 포스팅"
4. 본문 입력 완료
5. 이미지 2장 업로드 완료
6. 레이아웃 선택: 개별사진
7. 첫 번째 이미지 링크 추가 완료
8. 두 번째 이미지 링크 추가 완료
9. 발행 버튼 클릭
10. 발행 설정 확인
11. 최종 발행 완료
12. 포스트 URL 확인: blog.naver.com/PostView.naver?blogId=shinws8908&logNo=224094913446
```

---

**문서 작성자**: Claude Code
**마지막 업데이트**: 2025-12-02
**버전**: 1.0.0
