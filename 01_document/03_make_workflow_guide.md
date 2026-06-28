# Make 워크플로우 상세 설정 가이드

> **도구**: Make (make.com)  
> **업데이트**: 2026-06-28  
> **워크플로우명**: N_B2-2_News rss  
> **시나리오 URL**: https://us2.make.com/2493980/scenarios/5514616/edit

---

## 0. 사전 준비

| 항목 | 내용 |
|------|------|
| Make 계정 | make.com 무료 플랜 (월 1,000 작업) |
| **Google AI Studio API 키** | [aistudio.google.com](https://aistudio.google.com) → API Keys → Create API Key |
| Notion Integration Token | [notion.com/my-integrations](https://www.notion.com/my-integrations) → New Integration |
| Notion Database ID | DB 페이지 URL에서 추출 (`/p/` 뒤 32자리) |

> ⚠️ **보안 주의**: API 키는 Make의 `Connections`에 등록. 스크린샷 공유 시 반드시 마스킹 처리.

> ⚠️ **모델 주의**: `gemini-2.0-flash`는 2026년 6월 1일 단종. 반드시 **`Gemini Flash Latest`** (Gemini 2.5 Flash) 이상 사용.

---

## 1. 시나리오 생성

1. Make 대시보드 → **Create a new scenario**
2. 시나리오명: `N_B2-2_News rss`
3. 하단 스케줄: **Daily at 09:00 (Asia/Seoul)** 설정

---

## 2. 모듈 구성 (실제 구현 순서)

실제 구현된 파이프라인은 5개 모듈로 구성:

```
RSS[2] → Text parser[3] → Text parser[5] → Google Gemini AI[8] → Notion[14]
```

---

### 모듈 2 – RSS 피드 수집 (RSS · Retrieve RSS feed items)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **RSS > Retrieve RSS feed items** |
| Feed URL | `https://www.aitimes.com/rss/allArticle.xml` |
| Date from | *(비워둠 - 날짜 제한 없음)* |
| Date to | *(비워둠)* |
| Maximum number of returned items | `10` |

> ⚠️ **필터 주의**: RSS와 다음 모듈 사이의 Filter(오늘 발행된 글만)는 삭제 권장.  
> `Date created > now` 조건은 항상 0건 반환. 대신 RSS의 `Date from` 필드 사용.

**추천 RSS 피드 (AI 관련)**

```
# 국내
https://www.aitimes.com/rss/allArticle.xml    # AI타임스 (실사용)
https://zdnet.co.kr/rss/                       # ZDNet Korea

# 국제
https://techcrunch.com/feed/                   # TechCrunch
https://feeds.feedburner.com/oreilly/radar/atom  # O'Reilly Radar
https://rss.arxiv.org/rss/cs.AI               # arXiv AI
```

---

### 모듈 3 – HTML 정제 (Text parser · HTML to text)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Text parser > HTML to text** |
| HTML | `{{2.description}}` (RSS 기사 본문 HTML) |

> RSS 피드의 본문에 포함된 HTML 태그를 제거하여 순수 텍스트로 변환.

---

### 모듈 5 – 키워드 필터 (Text parser · Match pattern)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Text parser > Match pattern** |
| Pattern | `(AI\|인공지능\|LLM\|GPT\|ChatGPT\|Gemini\|머신러닝\|딥러닝\|생성형\|Claude\|자동화)` |
| Text | `{{2.title}} {{3.text}}` (제목 + 본문) |
| Case sensitive | **No** |
| **Continue even if no matches** | **Yes** (파이프라인 중단 방지) |

**필터 키워드 (OR 조건)**

```
AI, 인공지능, LLM, GPT, ChatGPT, Gemini, Claude,
머신러닝, 딥러닝, 생성형, 자동화
```

> **선택 이유**: AI/ML 분야 핵심 용어로 관련 기사를 선별. `Continue even if no match: Yes`로 설정하여 비매칭 기사도 파이프라인을 계속 통과시켜 안정성 확보.

---

### 모듈 8 – AI 요약 (Google Gemini AI · Generate a response)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Google Gemini AI > Generate a response** |
| Connection | Google AI Studio API Key 연결 |
| **Model** | **`Gemini Flash Latest`** (Gemini 2.5 Flash) |
| Temperature | `0.3` |

> ⚠️ `gemini-2.0-flash`는 2026년 6월 단종. `Gemini Flash Latest` 사용.  
> 무료 티어: **분당 5건** (5 RPM) 한도 적용.

**System Prompt**
```
당신은 전문 기술 뉴스 요약 편집자입니다.
입력된 뉴스 기사를 읽고 핵심 내용을 정확하고 간결하게 한국어로 요약합니다.
반드시 3줄 이내 불릿 포인트(•)로 작성하세요.
```

**User Prompt**
```
다음 뉴스 기사를 3줄 이내로 요약해주세요:
제목: {{2.title}}
내용: {{3.text}}
```

---

### 모듈 14 – Notion DB 저장 (Notion · Create a Database Item Legacy)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Notion > Create a Database Item (Legacy)** |
| Connection | Notion Internal Integration Token 연결 |
| Database ID | `38cce80d-e197-80d9-ae01-e400904c153d` (또는 실제 DB ID) |

> ✅ **Legacy 모듈 사용 이유**: Make의 신버전 Notion 모듈(`Create a Data Source Item`)은 Internal Integration Token과 호환 불안정. Legacy 모듈은 수동 DB ID 입력으로 안정적으로 작동.

**속성 매핑**

| Notion 속성 | 타입 | Make 값 | 비고 |
|-------------|------|---------|------|
| `이름` | Title | `{{2.title}}` | RSS 기사 제목 |
| `Summary` | Rich Text | `{{8.result}}` | Gemini 요약 결과 |
| `URL` | URL | `{{2.url}}` | 원문 링크 |
| `Published Date > Start Time` | Date | `{{2.RSS fields: pubdate}}` | 발행일 |
| `Source` | Text (Map ON) | `AI타임스` | 직접 입력 |
| `Status` | Select (Map ON) | `완료` | 직접 입력 |

> ⚠️ **Map 토글**: Source, Status가 Select 유형이면 오른쪽 `Map` 토글을 **ON**으로 설정 후 직접 텍스트 입력.

---

## 3. Notion Integration 설정 방법

### 3.1 Integration 생성
1. [notion.com/my-integrations](https://www.notion.com/my-integrations) 접속
2. **`+ New integration`** 클릭
3. Integration 이름: `Make_NewsBot`
4. Workspace 선택 후 **Submit**
5. **Internal Integration Secret** 복사 (`secret_xxx...`)

### 3.2 DB에 Integration 연결 (필수!)
1. Notion DB 페이지 열기
2. 오른쪽 상단 **`···`** 클릭
3. **`연결`** → **`연결 추가`** → `Make_NewsBot` 선택

> ⚠️ **DB 연결을 하지 않으면** Make에서 `No matches found` 오류 발생.

### 3.3 Make에서 Connection 추가
1. Notion 모듈 설정 → **`Connection`** → **`Add`**
2. Connection type: **`Notion Internal`**
3. Internal Integration Token: 복사한 `secret_xxx...` 붙여넣기
4. **Save**

---

## 4. 에러 처리 설정

### 4.1 RSS 날짜 필터 오류
- **증상**: `RSS 수집 결과 0건`
- **원인**: 모듈 간 Filter 조건이 `Date created > now` (미래 날짜 조건)
- **해결**: 해당 Filter 삭제 or RSS `Date from`에 `2026-06-01` 형태 직접 입력

### 4.2 Gemini API 429 오류 (Rate Limit)
- **증상**: `Request limit reached, limit: 0 (gemini-2.0-flash)`
- **원인**: 모델 단종 (2026.6.1 이후 limit=0)
- **해결**: 모델을 `Gemini Flash Latest`로 변경

### 4.3 Gemini API 503 오류 (일시적 과부하)
- **증상**: `This model is currently experiencing high demand`
- **원인**: 서버 일시 과부하 (5~10분 내 자동 해소)
- **해결**: 잠시 후 `Run once` 재시도

### 4.4 RSS 처리 건수 초과 (429 Rate Limit)
- **증상**: Operation 5~8번에서 `limit: 5` 초과
- **원인**: Gemini 2.5 Flash 무료 티어 분당 5건 제한
- **해결**: RSS `Maximum items`를 `4` 이하로 설정

---

## 5. 테스트 & 검증

```
1. RSS [2] 모듈 우클릭 → "Choose where to start" → "All" 선택
   (처음부터 모든 기사 재수집 - 반복 테스트용)

2. 시나리오 편집 화면에서 [Run once] 클릭

3. 각 모듈의 출력 버블(숫자)을 클릭하여 데이터 흐름 확인:
   - RSS[2]: 수집된 기사 수 확인
   - Text parser[3]: HTML 제거된 텍스트 확인
   - Text parser[5]: 키워드 매칭 결과 확인
   - Gemini[8]: 3줄 요약 생성 확인
   - Notion[14]: DB 저장 성공 확인

4. Notion DB에서 레코드 생성 여부 확인

5. 스케줄 활성화: 하단 토글 ON (Daily at 09:00)
```

---

## 6. 비용 예상 (2026년 기준)

| 항목 | 무료 플랜 기준 |
|------|---------------|
| Make 작업 수 | 월 1,000 작업 (하루 약 33건 여유) |
| **Google Gemini 2.5 Flash** | 무료 티어: 분당 5건, 일 500건 |
| Notion API | 무료 |
| **합계** | **전체 무료** (하루 10건 이하 처리 기준) |

> ⚠️ Gemini 2.5 Flash 무료 티어는 분당 5 RPM 제한. RSS 최대 수집을 4건으로 설정 시 안전.
