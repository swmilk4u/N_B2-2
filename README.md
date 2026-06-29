# N_B2-2 뉴스 요약 자동화 워크플로우 (Make & n8n)

> **Project B** | Codyssey AI Native Course  
> **자동화 툴**: Make (make.com) & n8n  
> **AI 모델**: Google Gemini 2.5 Flash 및 OpenAI GPT-4o-mini  
> **저장소**: Notion Database


---

## 1. 프로젝트 개요
본 프로젝트에서는 노코드(No-Code) 툴인 **Make**와 로코드(Low-Code) 툴인 **n8n** 두 가지 방식을 모두 활용하여 파이프라인을 다각도로 설계하고 성공적으로 가동시켰습니다.
- **수집**: RSS 피드를 통한 최신 기술 뉴스 모니터링
- **정제/필터**: HTML 노이즈 제거 및 사전에 정의한 기술 키워드 기반 필터링
- **요약**: 생성형 AI(Gemini 2.5 Flash / GPT-4o-mini)를 활용한 핵심 3줄 요약
- **저장**: Notion 데이터베이스에 저장하여 지속 관리

### 사용 도구 스택
| 역할 | Make 워크플로우 | n8n 워크플로우 |
| :--- | :--- | :--- |
| **자동화 플랫폼** | **Make** (SaaS형 노코드) | **n8n** (셀프 호스팅/로코드) |
| **뉴스 수집 (RSS)** | RSS (Retrieve RSS feed items) | RSS Feed Read Node |
| **HTML 정제** | Text parser (HTML to text) | Code Node (Regex & DOM 파싱) |
| **주제 필터링** | Text parser (Match pattern) | Code Node (키워드 매칭 및 최신 1건 선택) |
| **AI 요약 모델** | **Google gemini-2.5-flash** | **OpenAI gpt-4o-mini** |
| **저장 플랫폼** | **Notion Database** | **Notion Database** |

---

## 2. Make 워크플로우 구조 및 설명

### 2.1 워크플로우 구조도
```
[Schedule Trigger] (매일 09:00, Asia/Seoul)
     │
     ▼
[모듈 2] RSS 피드 수집 (Retrieve RSS feed items)
     │ Feed URL: https://www.aitimes.com/rss/allArticle.xml
     ▼
[모듈 3] HTML 정제 (Text parser · HTML to text)
     │
     ▼
[모듈 5] 키워드 필터 (Text parser · Match pattern)
     │ AI | 인공지능 | LLM | GPT | 머신러닝 등
     ▼
[모듈 8] Google Gemini AI (gemini-2.5-flash)
     │ System Prompt 기반 3줄 요약 생성
     ▼
[모듈 14] Notion DB 저장 (Create a Database Item Legacy)
```

### 2.2 Make워크플로우 (성공)
![Make 워크플로우](./02_img/01%20make/01%20메이크%20워크플로우_1.png)

### 2.3 Notion 저장 결과
![Make 노션 저장 결과](./02_img/01%20make/02%20메이크_노션%20결과_1.png)

### 2.4 단계별 모듈 설명
- **모듈 2 - RSS Feed 수집**: `https://www.aitimes.com/rss/allArticle.xml` (AI타임스) 피드로부터 뉴스 데이터를 수집합니다.
- **모듈 3 - HTML 정제**: RSS 본문에 포함된 지저분한 HTML 태그를 제거해 순수 텍스트로 변환하여 AI 토큰 사용량을 최적화합니다.
- **모듈 5 - 키워드 필터**: AI 관련 키워드 정규식 매칭을 거쳐 필터링합니다. 단, 파이프라인의 중단을 막기 위해 `Continue even if no matches` 옵션을 `Yes`로 적용했습니다.
- **모듈 8 - Google Gemini AI 요약**: `gemini-2.5-flash` 모델을 사용하여 기사 제목 및 정제된 본문을 바탕으로 핵심 3줄 요약을 생성합니다.
- **모듈 14 - Notion DB 저장**: 신규 모듈의 연동 불안정을 방지하고자 안정적인 `Legacy` 모듈을 선택하고 Database ID를 수동 입력하여 저장했습니다.

---

## 3. n8n 워크플로우 구조 및 설명

### 3.1 워크플로우 구조도
```
[스케줄 트리거] (매일 09:00, Asia/Seoul)
        │
        ▼
[RSS 수집(TechCrunch)]
        │ Feed URL: https://techcrunch.com/feed/
        ▼
[주제 필터 (Code 노드)] ── 뉴스 없음(false) ──▶ [알림: 뉴스 없음] (정상 종료)
        │ 뉴스 있음(true)
        ▼
[Notion 중복 조회] (원문 링크 비교)
        │
        ▼
[신규 기사?] ── 중복 기사(false) ──▶ [중복: 스킵] (비용 0, 생략)
        │ 신규 기사(true)
        ▼
[OpenAI 요약 (gpt-4o-mini)] (요약 + 감성 분석 수행)
        │
        ▼
[요약 파싱 (Code 노드)]
        │
        ▼
[Notion 저장] (HTTP Request 활용 API 저장)
```

### 3.2 n8n 워크플로우
#### 성공 케이스 (Schedule Trigger)
![n8n 워크플로우 성공](./02_img/02%20n8n/01_n8n_워크플로_Schedule_Trigger.png)

#### 에러 케이스 (Error Trigger)
![n8n 워크플로우 에러](./02_img/02%20n8n/01_n8n_워크플로_Error_Trigger.png)

### 3.3 Notion 저장 결과
![n8n 노션 저장 결과](./02_img/02%20n8n/02_노션_DB_페이지_260629.png)

### 3.4 단계별 노드 설명
- **Schedule Trigger**: 매일 아침 9시 정각에 뉴스 정리 작업이 스스로 시작하도록 실행 시간(크론식 `0 9 * * *`)을 예약해 둔 단계입니다.
- **RSS Feed Read**: TechCrunch RSS를 통해 최신 발행 기사 데이터를 수집합니다.
- **주제 필터 (Code Node)**: AI 분야 키워드를 검색하여 매칭시키며, 과제 요구사항에 맞게 조건 충족 기사 중 **가장 최신 1건**을 최종 추출합니다. 기사가 존재하지 않는 경우 파이프라인의 에러가 아닌 정상 분기로 빠지게 처리합니다.
- **Notion 중복 조회 (HTTP Request)**: Notion DB를 URL(원문 링크) 기준으로 필터링 쿼리하여 이미 등록된 기사가 있는지 선제 조회합니다.
- **OpenAI 요약 (HTTP Request)**: 신규 기사인 경우에만 `gpt-4o-mini` API를 호출해 핵심 3줄 요약과 함께 기사 감성(긍정/부정/중립)을 한 번에 분석합니다.
- **요약 파싱 (Code Node)**: AI 응답에서 요약문과 감성 데이터를 발라내어 정제합니다.
- **Notion 저장 (HTTP Request)**: 최종 정제된 텍스트 및 속성 값을 Notion DB에 저장합니다.

---

## 4. 두 워크플로우 비교 분석 (Make vs n8n)

자동화 목적을 달성하기 위해 사용한 두 툴의 차이점 및 특징을 다음과 같이 비교 분석했습니다.

| 비교 항목 | Make 워크플로우 | n8n 워크플로우 |
| :--- | :--- | :--- |
| **플랫폼 특징** | 클라우드 SaaS (웹 인터페이스 중심) | 오픈소스 / 셀프 호스팅 가능 (Docker 등) |
| **요금 및 제한** | 무료 플랜: 월 1,000 operations 제한 | 셀프 호스팅 시 무료 (API 호출 비용만 발생) |
| **적용 AI 모델** | Google gemini-2.5-flash | OpenAI gpt-4o-mini |
| **중복 방지** | Notion Legacy의 필터링이 어려워 RSS 설정에 의존 | 저장 전 Notion API 쿼리로 **중복 사전 탐지 및 건너뛰기** 완벽 구현 |
| **비용 최적화** | 필터 여부와 상관없이 모듈 실행 횟수 차감 | 중복 기사는 요약 단계를 거치지 않고 종료되어 **AI 토큰 비용 0원** 유지 |
| **에러 핸들링** | 모듈별 에러 디렉티브(Ignore, Resume) 연결 | `Error Trigger` 노드를 통해 중앙 집중식 에러 핸들링 가능 |
| **장점** | UI/UX가 직관적이고 노코드 설정이 극도로 간편함 | 코드 노드(Javascript)를 쓸 수 있어 복잡한 가공과 분기 처리가 강력함 |
| **단점** | 복잡한 조건 처리 시 오퍼레이션 소모량이 급격히 증가함 | 설치 및 세팅 장벽이 있으며 자격증명 설정이 조금 더 복잡함 |

---

## 5. 주제 필터링 기준

### 5.1 필터 키워드
* **Make**: `(AI|인공지능|LLM|GPT|ChatGPT|Gemini|머신러닝|딥러닝|생성형|Claude|자동화)` (정규식 OR 매칭)
* **n8n**: `AI`, `artificial intelligence`, `machine learning`, `deep learning`, `LLM`, `GPT`, `OpenAI`, `Anthropic`, `Gemini`, `neural`, `generative`, `인공지능`, `생성형`, `머신러닝`, `딥러닝`

### 5.2 선택 이유
- **트렌드 집중**: 업무 생산성 향상과 밀접하게 연관된 AI/ML 기술 중심의 뉴스만을 골라내기 위함입니다.
- **영한 혼용 지원**: 영문 IT 뉴스(TechCrunch 등)와 국내 IT 뉴스(AI타임스 등)에 모두 유연하게 대응하기 위해 키워드를 다국어로 구성했습니다.

---

## 6. 에러 처리 및 예외 복구 정책

안정적인 무인 가동을 실현하기 위해 설계한 예외 처리 체계는 다음과 같습니다.

### 6.1 Make의 에러 정책
- **Gemini Rate Limit (429 에러) 예방**: 무료 티어 제한(분당 5건)을 초과하지 않도록 RSS 수집 최대 개수를 4건 내외로 적절히 제어합니다.
- **Notion 모듈 연동 불안정 대응**: 신형 모듈 대신 안정성이 확보된 Notion Legacy 모듈을 사용하여 필터링 오류를 예방합니다.

### 6.2 n8n의 에러 정책
- **일시 장애 자동 복구**: RSS, OpenAI, Notion 등 외부 API 통신 노드들에 대해 **최대 2회 자동 재시도(5초 간격)** 옵션을 지정했습니다.
- **에러 격리 (Graceful Fail)**: 특정 뉴스 가공에 실패하더라도 전체 파이프라인이 멈추지 않고, 에러 로그 기록 후 다음 프로세스를 정상적으로 수행합니다.
- **오류 감지 및 알림**: 최종 실패 건은 `Error Trigger` 노드가 이를 캐치하여 관리자 통지(Slack/Email NoOp 노드 연결) 경로를 실행합니다.

---

## 7. 비용 및 중복 방지 정책

### 7.1 중복 저장 방지
- **Make**: RSS 모듈 자체의 고유 데이터 수집 캐싱과 날짜 조건 설정을 이용해 매 실행 시 신규 건만 들여옵니다.
- **n8n**: Notion 데이터베이스 조회 API를 사전에 호출하여 `원문 링크 == link` 조건을 만족하는 기존 데이터가 발견되면 **저장 및 요약 단계를 스킵(Skip)**합니다.

### 7.2 AI 요약 비용 최소화
- **n8n 중복 사전 차단**: 이미 저장 완료된 뉴스 링크에 대해서는 OpenAI 요약 노드를 건너뛰도록 분기를 설계하여, 불필요한 AI 요약 비용을 0원으로 억제했습니다.
- **1건 요약 원칙**: RSS 수집 리스트 중 주제가 매칭된 최상위 1건만 최종 요약함으로써 일별 AI 토큰 비용을 최소화합니다.

---

## 8. 노션 DB 스키마 (보너스 과제 적용)

뉴스 요약 결과를 구조화하여 저장하기 위해 구축한 데이터베이스 스키마 명세입니다.

| 속성명 | 데이터 타입 | 설명 | Make 값 매핑 | n8n 값 매핑 |
| :--- | :--- | :--- | :--- | :--- |
| **이름** | Title | 뉴스 기사 제목 | `{{2.title}}` | `title` |
| **Summary** | Rich text | AI가 작성한 3줄 요약 | `{{8.result}}` | `summary` |
| **URL** | URL | 기사 원본 링크 (중복 방지 키) | `{{2.url}}` | `link` |
| **Published Date** | Date | 기사 발행 일시 | `{{2.RSS fields: pubdate}}` | `isoDate` |
| **Source** | Text / Rich text | 뉴스의 출처 매체명 | `AI타임스` (고정) | `TechCrunch` (고정) |
| **주제 태그** | Multi-select | 매칭된 AI 기술 분야 키워드 | - | `matchedKeywords` |
| **감성** | Select | 기사 톤앤매너 감성 분류 (긍정/부정/중립) | - | `sentiment` (보너스 과제 2) |
| **상태** | Select | 저장 처리 및 운영 모니터링 상태 | `완료` | `저장됨` |

---

## 9. 산출물 목록

| 번호 | 산출물 구분 | 파일/디렉토리 위치 |
| :--- | :--- | :--- |
| **1** | Make 워크플로우 상세 설정 가이드 | [03_make_workflow_guide.md](./03_make/03_make_workflow_guide.md) |
| **2** | n8n 설계 결정서 | [design-decisions.md](./04_n8n/docs/design-decisions.md) |
| **3** | n8n 워크플로우 설명서 | [workflow-design.md](./04_n8n/docs/workflow-design.md) |
| **4** | n8n 워크플로우 내보내기 JSON | [news-summary.n8n.json](./04_n8n/workflow/news-summary.n8n.json) |

---

## 10. 팀 역할 / 개인 작업 요약
| 이름 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 역할 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 담당 작업 |
| :--- | :--- | :--- |
| <span style="white-space: nowrap;"><b>유상우</b></span> | <span style="white-space: nowrap;">팀장</span> | **프로젝트 관리 및 Make 워크플로우 설계** (전체 일정 조율 및 산출물 통합 관리, Make 시나리오 구성, RSS 수집 및 HTML 정제 필터 구축, `gemini-2.5-flash` API 연동 및 프롬프트 최적화, Notion Legacy DB 연동 및 매핑) |
| <span style="white-space: nowrap;"><b>정정일</b></span> | <span style="white-space: nowrap;">팀원</span> | **n8n 워크플로우 설계** (n8n 노드 구성, 스케줄·필터·분기 로직 설계, Notion DB 중복 조회 쿼리 구현, `gpt-4o-mini` API 연동 및 에러 처리 설계) |
- **공통 작업**:
  - **문서화 및 비교**: 두 도구의 워크플로우와 특징을 깊게 이해하고 비교 정리하여 통합 보고서 작성.