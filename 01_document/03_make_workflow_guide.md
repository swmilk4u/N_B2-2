# Make 워크플로우 상세 설정 가이드

> **도구**: Make (make.com)  
> **업데이트**: 2026-06-28  
> **워크플로우명**: News_RSS_Summarizer

---

## 0. 사전 준비

| 항목 | 내용 |
|------|------|
| Make 계정 | make.com 무료 플랜 (월 1,000 작업) |
| **Google AI Studio API 키** | [aistudio.google.com](https://aistudio.google.com) → API Keys → Create API Key |
| Notion Integration Token | notion.so → Settings → Integrations |
| Notion Database ID | DB 페이지 URL에서 추출 |

> ⚠️ **보안 주의**: API 키는 Make의 `Connections` 또는 환경 변수로 등록. 스크린샷 공유 시 반드시 마스킹 처리.

---

## 1. 시나리오 생성

1. Make 대시보드 → **Create a new scenario**
2. 시나리오명: `News_RSS_AI_Summarizer`
3. **Add first module** 클릭

---

## 2. 모듈 구성 (순서대로)

### 모듈 1 – Schedule Trigger (스케줄 트리거)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Tools > Schedule** |
| 실행 주기 | Every Day |
| 실행 시각 | `09:00` |
| 타임존 | `Asia/Seoul` |

---

### 모듈 2 – RSS 피드 수집

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **RSS > Get RSS Feed Items** |
| Feed URL | `https://feeds.feedburner.com/oreilly/radar/atom` (예시) |
| Maximum number of results | `20` |

**추천 RSS 피드 목록 (AI 관련)**

```
# 국제
https://feeds.feedburner.com/oreilly/radar/atom          # O'Reilly Radar
https://rss.arxiv.org/rss/cs.AI                          # arXiv AI
https://techcrunch.com/feed/                              # TechCrunch
https://feeds.arstechnica.com/arstechnica/index          # Ars Technica
https://www.technologyreview.com/feed/                    # MIT Technology Review

# 국내
https://www.aitimes.com/rss/allArticle.xml               # AI타임스
https://zdnet.co.kr/rss/                                  # ZDNet Korea
```

---

### 모듈 3 – Iterator (반복 처리)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Flow Control > Iterator** |
| Array | `{{2.items}}` (RSS 피드 결과 배열) |

---

### 모듈 4 – 키워드 필터 (Router + Filter)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Flow Control > Filter** |
| 필터명 | AI_Keyword_Filter |
| 조건 | `{{3.title}}` + `{{3.summary}}` 포함 여부 |

**필터 키워드 (OR 조건)**

```
AI | 인공지능 | LLM | GPT | 머신러닝 | machine learning
| deep learning | 딥러닝 | ChatGPT | Gemini | Claude
| 생성형 AI | generative AI | 자율주행 | autonomous
| 로봇 | robotics
```

> **선택 이유**: 기술 트렌드 중 빠르게 성장하는 AI/ML 분야에 집중하여 팀 업무 관련 인사이트를 최대화하기 위함.

---

### 모듈 5 – Data Store 중복 체크

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Data Store > Search Records** |
| Data Store | `NewsLinks` (직접 생성) |
| Filter Field | `url_hash` |
| Filter Value | `{{md5(3.link)}}` |

**Data Store 구조 (`NewsLinks`)**

| 필드명 | 타입 | 설명 |
|--------|------|------|
| `url_hash` | Text (Key) | MD5(원문 링크) |
| `processed_at` | Date | 처리 일시 |

**중복 필터 설정**
- Search Records 결과가 비어있을 때만 다음 단계 진행
- Filter: `{{length(5.records)}} = 0`

---

### 모듈 6 – Limit (1건만 처리)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Flow Control > Limiter** |
| 최대 처리 수 | `1` |

> 하루 1건만 선택하여 AI 요약 비용 최소화 (요구사항 4.1 준수)

---

### 모듈 7 – Google Gemini (AI 요약)

| 설정 항목 | 값 |
|----------|----|  
| 모듈 | **Google Gemini > Generate Content** |
| Connection | Google AI Studio API Key 연결 |
| Model | `gemini-2.0-flash` |
| Temperature | `0.3` |

> ✅ **Gemini 무료 티어**: 일 1,500원격 / 분 15회 호출 무료. 이 오토메이션의 일 1건 처리에 없는 요금.

**Prompt (System + User 통합 입력)**
```
[시스템]
당신은 전문 기술 뉴스 요약 편집자입니다.
입력된 뉴스 기사를 읽고 핵심 내용을 정확하고 간결하게 한국어로 요약합니다.
반드시 3줄 이내 불릿 포인트(•)로 작성하세요.

[유저]
다음 뉴스 기사를 3줄 이내로 요약해주세요:
제목: {{3.title}}
내용: {{3.summary}}
```

---

### 모듈 8 – Data Store 저장 (중복 방지 등록)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Data Store > Add/Replace a Record** |
| Data Store | `NewsLinks` |
| `url_hash` | `{{md5(3.link)}}` |
| `processed_at` | `{{now}}` |

---

### 모듈 9 – Notion DB 저장

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Notion > Create a Database Item** |
| Connection | Notion Integration Token 연결 |
| Database ID | 노션 DB URL에서 복사 |

**속성 매핑**

| Notion 속성 | 타입 | Make 값 |
|-------------|------|---------|
| `Title` | Title | `{{3.title}}` |
| `Summary` | Rich Text | `{{7.candidates[].content.parts[].text}}` |
| `URL` | URL | `{{3.link}}` |
| `Published Date` | Date | `{{3.pubDate}}` |
| `Source` | Select | `AI타임스` (또는 피드 이름) |
| `Status` | Select | `완료` |

---

## 3. 에러 처리 설정

### 3.1 Error Handler 추가

모든 모듈에서 우클릭 → **Add error handler** → **Resume** 선택

```
오류 발생
    → 재시도 1회 (5분 후)
    → 재시도 2회 (10분 후)
    → 최대 재시도 초과 시:
        → 이메일 또는 Slack 알림 발송
        → 해당 기사 스킵 (Ignore)
```

### 3.2 불완전 실행 처리

Make 시나리오 설정 → **Allow storing Incomplete Executions** 활성화

| 에러 유형 | 처리 방법 |
|-----------|-----------|
| RSS 피드 수집 실패 | 재시도 2회 → 알림 후 스킵 |
| OpenAI API 오류 | 재시도 2회 → 알림 후 스킵 |
| Notion 저장 실패 | 재시도 2회 → 불완전 실행 저장 |
| 키워드 미매칭 | 해당 일자 스킵 (정상 처리) |

### 3.3 이메일 알림 모듈 (선택)

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **Email > Send an Email** |
| To | 팀 이메일 주소 |
| Subject | `[뉴스봇] 오류 발생 - {{now}}` |
| Body | `오류 내용: {{error.message}}` |

---

## 4. 보너스 기능 (선택)

### 보너스 1 – 감성 분석 태그

모듈 7 (Google Gemini) Prompt에 추가:
```
추가로 기사의 감성을 분석하여 아래 중 하나로 분류하세요:
[긍정적] [부정적] [중립적]
```

노션 속성에 `Sentiment` (Select) 추가하여 저장.

### 보너스 2 – 썸네일 이미지 생성

| 설정 항목 | 값 |
|----------|----|
| 모듈 | **OpenAI > Create an Image** |
| Model | `dall-e-3` |
| Prompt | `기사 제목 기반 썸네일: {{3.title}}` |
| Size | `1024x1024` |

생성된 이미지 URL을 노션 `Cover` 또는 `Thumbnail URL` 속성에 저장.

---

## 5. 테스트 & 검증

```
1. 시나리오 편집 화면에서 [Run once] 클릭
2. 각 모듈의 출력 버블(숫자)을 클릭하여 데이터 흐름 확인
3. 노션 DB에 레코드 생성 여부 확인
4. 동일 URL로 재실행 → Data Store 필터로 스킵되는지 확인
5. 스케줄 활성화: 오른쪽 하단 토글 ON
```

---

## 6. 비용 예상

| 항목 | 무료 플랜 기준 |
|------|---------------|
| Make 작업 수 | 월 1,000 작업 (하루 약 33건 여유) |
| **Google Gemini** | **완전 무료** (일 1,500원격 / 분 15회 한도 내) |
| Notion API | 무료 |
| **합계** | **전체 무료** (Make + Gemini + Notion 모두 무료 플랜 적용 가능) |
