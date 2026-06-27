# N_B2-2 뉴스 요약 자동화 워크플로우 만들기

> **Project B** | Codyssey AI Native Course  
> **제출일**: 2026-06-28  
> **자동화 툴**: Make (make.com)  
> **AI 모델**: Google Gemini 2.0 Flash  
> **저장소**: Notion Database

---

## 📋 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [워크플로우 구조](#2-워크플로우-구조)
3. [단계별 모듈 설명](#3-단계별-모듈-설명)
4. [주제 필터링 기준](#4-주제-필터링-기준)
5. [에러 처리 정책](#5-에러-처리-정책)
6. [비용 및 중복 방지 정책](#6-비용-및-중복-방지-정책)
7. [노션 DB 스키마](#7-노션-db-스키마)
8. [산출물 목록](#8-산출물-목록)
9. [팀 역할 및 개인 작업 요약](#9-팀-역할-및-개인-작업-요약)

---

## 1. 프로젝트 개요

새로운 프로젝트 착수 시 시장 조사에 소요되는 반복적 수작업을 자동화한다.  
**RSS 피드 수집 → AI 요약 → 노션 DB 자동 저장** 파이프라인을 Make 노코드 플랫폼으로 구현하여 매일 최신 AI 관련 뉴스를 자동으로 요약 정리한다.

### 사용 도구 스택

| 역할 | 도구 |
|------|------|
| 자동화 플랫폼 | **Make** (make.com) |
| RSS 수집 | Make 내장 RSS 모듈 |
| AI 요약 | **Google Gemini 2.0 Flash** (Google AI Studio API) |
| 저장소 | Notion Database |
| 중복 방지 | Make Data Store (URL MD5 해시) |
| 에러 알림 | Make Error Handler + Email |

---

## 2. 워크플로우 구조

```
┌─────────────────────────────────────────────────────────┐
│              Make 시나리오: News_RSS_AI_Summarizer       │
└─────────────────────────────────────────────────────────┘

[1] 스케줄 트리거
    매일 09:00 (Asia/Seoul)
         │
         ▼
[2] RSS 피드 수집
    Get RSS Feed Items
    최대 20건 수집
         │
         ▼
[3] Iterator + 키워드 필터
    AI / LLM / GPT / 인공지능 / 머신러닝 등
         │
         ▼
[4] Data Store 중복 체크
    MD5(URL) 해시로 기존 처리 여부 확인
    → 중복이면 SKIP
         │ (신규 기사)
         ▼
[5] Limiter
    하루 1건만 선택
         │
         ▼
[6] Google Gemini
     gemini-2.0-flash로 3줄 요약 생성
         │
         ▼
[7] Data Store 저장
    URL 해시 등록 (중복 방지)
         │
         ▼
[8] Notion DB 저장
    Title / Summary / URL / Date 매핑
         │
         ▼
[Error Handler]
    실패 시 재시도 (최대 2회)
    → 초과 시 이메일 알림 + 스킵
```

---

## 3. 단계별 모듈 설명

### 모듈 1 – Schedule Trigger
- **Make 모듈**: Tools > Schedule
- **주기**: 매일 1회
- **실행 시각**: 09:00 (Asia/Seoul)
- **역할**: 전체 워크플로우의 자동 시작점. 사람 개입 없이 매일 정해진 시간에 파이프라인을 실행한다.

### 모듈 2 – RSS Feed 수집
- **Make 모듈**: RSS > Get RSS Feed Items
- **수집 피드**: AI 관련 뉴스 RSS (아래 목록 참고)
- **수집 건수**: 최대 20건

**사용 RSS 피드 목록**

| 피드명 | URL |
|--------|-----|
| AI타임스 | `https://www.aitimes.com/rss/allArticle.xml` |
| TechCrunch | `https://techcrunch.com/feed/` |
| O'Reilly Radar | `https://feeds.feedburner.com/oreilly/radar/atom` |
| MIT Technology Review | `https://www.technologyreview.com/feed/` |
| arXiv CS.AI | `https://rss.arxiv.org/rss/cs.AI` |

### 모듈 3 – Iterator + Filter (키워드 필터링)
- **Make 모듈**: Flow Control > Iterator → Filter
- **조건**: 제목 또는 내용에 아래 키워드 포함 시 통과

### 모듈 4 – Data Store 중복 체크
- **Make 모듈**: Data Store > Search Records
- **키**: `MD5(기사 URL)`
- **처리**: 이미 처리된 URL이면 해당 기사 건너뜀

### 모듈 5 – Limiter
- **Make 모듈**: Flow Control > Limiter
- **최대값**: 1 (하루 1건만 처리)

### 모듈 6 – Google Gemini (AI 요약)
- **Make 모듈**: Google Gemini > Generate Content
- **키 발급**: [Google AI Studio](https://aistudio.google.com) → API Keys
- **모델**: `gemini-2.0-flash`
- **Temperature**: 0.3

> ✅ **장점**: Gemini 무료 티어는 일 1,500원격/분 15회 호출 무료. API 키만 있으면 별도 요금 없이 사용 가능.

**프롬프트**
```
[시스템]
당신은 전문 기술 뉴스 요약 편집자입니다.
입력된 뉴스 기사를 읽고 핵심 내용을 정확하고 간결하게 한국어로 요약합니다.
반드시 3줄 이내 불릿 포인트(•)로 작성하세요.

[유저]
다음 뉴스 기사를 3줄 이내로 요약해주세요:
제목: {{제목}}
내용: {{본문}}
```

### 모듈 7 – Data Store 저장
- **Make 모듈**: Data Store > Add/Replace a Record
- MD5(URL) 해시와 처리 일시 저장 → 이후 중복 방지에 활용

### 모듈 8 – Notion DB 저장
- **Make 모듈**: Notion > Create a Database Item
- 아래 [노션 DB 스키마](#7-노션-db-스키마) 기준으로 속성 매핑

---

## 4. 주제 필터링 기준

### 선택 키워드 목록 (OR 조건)

```
AI, 인공지능, LLM, GPT, ChatGPT, Gemini, Claude,
머신러닝, machine learning, 딥러닝, deep learning,
생성형 AI, generative AI, 자율주행, autonomous,
로봇, robotics, 반도체, GPU, NVIDIA
```

### 선택 이유

| 이유 | 설명 |
|------|------|
| **업무 관련성** | AI/ML 기술 트렌드가 팀 프로젝트 방향에 직결됨 |
| **빠른 변화** | AI 분야는 주 단위로 중요 뉴스가 발생하여 매일 모니터링 필요 |
| **다양성 확보** | 영/한 키워드를 혼합하여 국내외 뉴스 모두 수집 |
| **노이즈 감소** | 광고·이벤트성 기사를 걸러내고 실질적 기술 기사만 선별 |

---

## 5. 에러 처리 정책

### 에러 처리 흐름

```
오류 발생
   │
   ├─ 재시도 1회 (5분 후 자동 재실행)
   │
   ├─ 재시도 2회 (10분 후 자동 재실행)
   │
   └─ 최대 재시도(2회) 초과
         │
         ├─ 이메일 알림 발송 (오류 내용 포함)
         └─ 해당 기사 스킵 (Ignore) → 다음 실행까지 대기
```

### 에러 유형별 처리 방법

| 에러 유형 | 처리 방법 | 이유 |
|-----------|-----------|------|
| RSS 피드 수집 실패 | 재시도 2회 → 알림 후 스킵 | 일시적 서버 오류 대응 |
| 키워드 미매칭 | 해당 일자 스킵 (정상 처리) | 뉴스가 없는 경우는 오류 아님 |
| OpenAI API 오류 | 재시도 2회 → 알림 후 스킵 | Rate limit / 네트워크 불안정 대응 |
| Notion 저장 실패 | 재시도 2회 → 불완전 실행 저장 | 데이터 손실 방지 |
| 중복 기사 감지 | 즉시 스킵 (정상 처리) | 불필요한 API 호출 방지 |

### 선택 이유

- **재시도 2회 상한**: 과제 요구사항(4.5) 준수 및 무한 루프·비용 폭증 방지
- **알림 발송**: 자동화 장애를 팀이 즉시 인지하여 수동 대응 가능
- **Ignore 처리**: 하나의 기사 실패가 전체 파이프라인을 중단하지 않도록 격리
- **불완전 실행 저장**: Make의 미완료 실행 기록을 보관해 수동 재처리 가능

---

## 6. 비용 및 중복 방지 정책

### 중복 방지

| 방법 | 구현 |
|------|------|
| URL MD5 해시 키 | `md5(기사 URL)` 를 Make Data Store에 저장 |
| 처리 전 조회 | 수집 후 즉시 Data Store 조회 → 존재 시 스킵 |
| 처리 후 등록 | 노션 저장 성공 후 Data Store에 등록 |

### 비용 최적화

| 항목 | 정책 |
|------|------|
| AI 요약 호출 | 기사 1건당 1회 (재시도 제외) |
| 하루 처리 건수 | Limiter로 1건 제한 |
| Make 작업 수 | 하루 약 30작업 → 무료 플랜(월 1,000) 내 처리 |
| **Gemini 비용** | **무료** (일 1,500원격, 분 15회 한도 내 무료 사용 가능) |

---

## 7. 노션 DB 스키마

### 데이터베이스명: `뉴스 요약 자동화`

| 속성명 | 타입 | 내용 | Make 매핑값 |
|--------|------|------|-------------|
| **Title** | Title | 기사 제목 | `{{rss.title}}` |
| **Summary** | Rich Text | AI 3줄 요약 | `{{openai.choices[].message.content}}` |
| **URL** | URL | 원문 링크 | `{{rss.link}}` |
| **Published Date** | Date | 기사 발행일 | `{{rss.pubDate}}` |
| **Source** | Select | 출처 피드명 | `AI타임스` 등 |
| **Status** | Select | 처리 상태 | `완료` / `오류` |
| **URL Hash** | Rich Text | 중복 방지 키 | `{{md5(rss.link)}}` |

### 노션 저장 결과 예시

```
┌────────────────────────────────────────────────────┐
│ 📰 AI 에이전트의 미래와 전망                        │
├────────────────────────────────────────────────────┤
│ 📝 요약                                            │
│  • 최신 AI 에이전트는 자율적 의사결정 능력이 강화   │
│  • 기업 생산성 도구와의 결합 사례가 급증하는 추세   │
│  • 보안 및 윤리적 가이드라인의 필요성이 대두됨      │
├────────────────────────────────────────────────────┤
│ 🔗 URL: https://example.com/news/ai-agent-2026     │
│ 📅 발행일: 2026-06-28                              │
│ 📡 출처: AI타임스                                  │
│ ✅ 상태: 완료                                      │
└────────────────────────────────────────────────────┘
```

---

## 8. 산출물 목록

| 번호 | 산출물 | 파일/위치 |
|------|--------|-----------|
| ① | Make 워크플로우 상세 설정 가이드 | [03_make_workflow_guide.md](./01_document/03_make_workflow_guide.md) |
| ② | 과제 미션 원문 | [PJ_B 과제미션.txt](./01_document/PJ_B%20과제미션.txt) |
| ③ | README (과제 수행 보고서) | README.md (현재 파일) |
| ④ | 워크플로우 스크린샷 | `02_img/` 폴더 (Make 실행 후 추가) |

> **보너스 기능**: 감성 분석 태그(긍정/부정/중립) 및 썸네일 이미지 자동 생성 기능을 워크플로우 가이드에 설계 포함 (Make에서 선택 활성화 가능)

---

## 9. 팀 역할 및 개인 작업 요약

| 역할 | 담당 | 주요 작업 |
|------|------|-----------|
| 워크플로우 설계 | 전체 | Make 시나리오 구조 기획 및 모듈 연결 |
| RSS 피드 선정 | 전체 | AI 관련 국내외 RSS 피드 목록 조사 및 선정 |
| AI 프롬프트 설계 | 전체 | GPT-4o-mini 요약 프롬프트 최적화 |
| 노션 DB 설계 | 전체 | 속성 스키마 정의 및 Make 매핑 설정 |
| 에러 처리 설계 | 전체 | Error Handler 정책 수립 |
| 문서화 | 전체 | README 및 워크플로우 가이드 작성 |

> ℹ️ 이번 과제는 **개인 수행** 과제로 전 단계를 단독으로 진행하였습니다.

---

## 📝 참고 문서

- [Make 공식 문서](https://www.make.com/en/help)
- [OpenAI API 문서](https://platform.openai.com/docs)
- [Notion API 문서](https://developers.notion.com)
- [Make 워크플로우 상세 설정 가이드](./01_document/03_make_workflow_guide.md)

---

*이 README는 과제 수행 보고서로 작성되었습니다. Make 계정 설정 후 [워크플로우 가이드](./01_document/03_make_workflow_guide.md)를 참고하여 실제 시나리오를 구성하세요.*
