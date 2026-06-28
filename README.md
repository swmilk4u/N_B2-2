# N_B2-2 뉴스 요약 자동화 워크플로우 만들기

> **Project B** | Codyssey AI Native Course  
> **제출일**: 2026-06-28  
> **자동화 툴**: Make (make.com)  
> **AI 모델**: Google Gemini 2.5 Flash (Gemini Flash Latest)  
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
**RSS 피드 수집 → HTML 정제 → AI 키워드 필터 → AI 요약 → 노션 DB 자동 저장** 파이프라인을 Make 노코드 플랫폼으로 구현하여 매일 최신 AI 관련 뉴스를 자동으로 요약 정리한다.

### 사용 도구 스택

| 역할 | 도구 |
|------|------|
| 자동화 플랫폼 | **Make** (make.com) |
| RSS 수집 | Make 내장 RSS 모듈 |
| HTML 정제 | Make Text parser (HTML to text) |
| 키워드 필터 | Make Text parser (Match pattern) |
| AI 요약 | **Google Gemini 2.5 Flash** (Gemini Flash Latest) |
| 저장소 | **Notion Database** (Internal Integration 연결) |

### 실행 결과

| 항목 | 내용 |
|------|------|
| Make 시나리오 | https://us2.make.com/2493980/scenarios/5514616/edit |
| Notion DB | https://app.notion.com/p/38cce80de19780d9ae01e400904c153d |
| 테스트 실행 결과 | 8건 처리 (7건 성공, 1건 Rate Limit - 무료 티어 한도) |

---

## 2. 워크플로우 구조

```
┌─────────────────────────────────────────────────────────┐
│          Make 시나리오: N_B2-2_News rss                  │
│          매일 09:00 자동 실행 (Asia/Seoul)                │
└─────────────────────────────────────────────────────────┘

[Schedule Trigger]
    매일 09:00 (Asia/Seoul)
         │
         ▼
[모듈 2] RSS 피드 수집
    https://www.aitimes.com/rss/allArticle.xml
    최대 10건 수집
         │
         ▼
[모듈 3] HTML 정제 (Text parser · HTML to text)
    RSS 본문 HTML 태그 제거 → 순수 텍스트 추출
         │
         ▼
[모듈 5] 키워드 필터 (Text parser · Match pattern)
    AI | 인공지능 | LLM | GPT | Gemini | 머신러닝 등
    Continue even if no match: Yes
         │
         ▼
[모듈 8] Google Gemini AI (Gemini Flash Latest)
    3줄 이내 한국어 불릿 요약 생성
         │
         ▼
[모듈 14] Notion DB 저장 (Create a Database Item Legacy)
    Title / Summary / URL / Date / Source / Status 매핑
```

---

## 3. 단계별 모듈 설명

### 모듈 2 – RSS Feed 수집
- **Make 모듈**: RSS > Retrieve RSS feed items
- **수집 피드**: `https://www.aitimes.com/rss/allArticle.xml` (AI타임스)
- **수집 건수**: 최대 10건 (Rate Limit 고려 시 4건 권장)
- **날짜 필터**: 없음 (스케줄 실행 시 마지막 처리 이후 신규 기사 자동 감지)

### 모듈 3 – HTML 정제
- **Make 모듈**: Text parser > HTML to text
- **입력**: `{{2.description}}` (RSS 본문 HTML)
- **역할**: HTML 태그 제거로 Gemini에 순수 텍스트 전달, 토큰 낭비 방지

### 모듈 5 – 키워드 필터
- **Make 모듈**: Text parser > Match pattern
- **Pattern**: `(AI|인공지능|LLM|GPT|ChatGPT|Gemini|머신러닝|딥러닝|생성형|Claude|자동화)`
- **Case sensitive**: No (대소문자 무시)
- **Continue even if no match**: Yes (비매칭 시에도 파이프라인 계속 실행)

### 모듈 8 – Google Gemini AI 요약
- **Make 모듈**: Google Gemini AI > Generate a response
- **모델**: `Gemini Flash Latest` (Gemini 2.5 Flash)
- **역할**: 수집된 기사를 3줄 한국어 불릿으로 요약
- **System Prompt**: 전문 기술 뉴스 요약 편집자 역할 지정

**프롬프트**
```
[System]
당신은 전문 기술 뉴스 요약 편집자입니다.
입력된 뉴스 기사를 읽고 핵심 내용을 정확하고 간결하게 한국어로 요약합니다.
반드시 3줄 이내 불릿 포인트(•)로 작성하세요.

[User]
다음 뉴스 기사를 3줄 이내로 요약해주세요:
제목: {{2.title}}
내용: {{3.text}}
```

### 모듈 14 – Notion DB 저장
- **Make 모듈**: Notion > Create a Database Item (Legacy)
- **연결**: Notion Internal Integration (`Make_NewsBot` 토큰)
- **역할**: Gemini 요약 결과를 포함한 기사 정보를 Notion DB에 자동 저장

> **Legacy 모듈 선택 이유**: Make의 신버전 Notion 모듈은 Internal Integration Token과 OAuth 방식이 달라 DB 목록 로드 불안정. Legacy 모듈은 Database ID 직접 입력으로 안정적 연동 가능.

---

## 4. 주제 필터링 기준

### 선택 키워드 목록 (OR 조건)

```
AI, 인공지능, LLM, GPT, ChatGPT, Gemini, Claude,
머신러닝, 딥러닝, 생성형, 자동화
```

### 선택 이유

| 이유 | 설명 |
|------|------|
| **업무 관련성** | AI/ML 기술 트렌드가 프로젝트 방향에 직결됨 |
| **빠른 변화** | AI 분야는 주 단위로 중요 뉴스가 발생하여 매일 모니터링 필요 |
| **노이즈 감소** | 광고·이벤트성 기사를 걸러내고 실질적 기술 기사만 선별 |
| **안정성** | `Continue even if no match: Yes`로 필터 미통과 시에도 파이프라인 유지 |

---

## 5. 에러 처리 정책

### 에러 처리 흐름

```
오류 발생
   │
   ├─ [Gemini 429 Rate Limit] → 잠시 대기 후 재시도
   │       원인: 무료 티어 분당 5건(5 RPM) 초과
   │       해결: RSS 최대 수집 건수를 4건으로 제한
   │
   ├─ [Gemini 503 High Demand] → 5~10분 후 자동 재시도
   │       원인: Google 서버 일시 과부하
   │       해결: Make Run once 재시도 (일시적 해소)
   │
   ├─ [RSS 0건 수집] → RSS 필터/날짜 설정 확인
   │       원인: Date 필터 조건 오류 (Date > now 형태)
   │       해결: 필터 삭제 또는 RSS Date from 날짜 직접 입력
   │
   └─ [Notion 저장 실패] → Integration 연결 확인
           원인: DB에 Integration 미연결
           해결: Notion DB 페이지 ··· → 연결 → Make_NewsBot 추가
```

### 에러 유형별 처리 방법

| 에러 유형 | 처리 방법 | 이유 |
|-----------|-----------|------|
| RSS 피드 수집 실패 | 재시도 후 스킵 | 일시적 서버 오류 대응 |
| 키워드 미매칭 | 계속 진행 (Continue: Yes) | 필터 미통과를 오류로 처리 안 함 |
| Gemini API 429 | 기사 수 줄이기 (4건 이하) | 무료 5 RPM 한도 준수 |
| Gemini API 503 | 잠시 후 재시도 | 일시적 과부하, 곧 해소 |
| Notion 저장 실패 | Integration 연결 재확인 | DB 페이지 수준 연결 필수 |

---

## 6. 비용 및 중복 방지 정책

### 비용 최적화

| 항목 | 정책 |
|------|------|
| AI 요약 호출 | 기사 1건당 1회 |
| 하루 처리 건수 | RSS 최대 10건 (Rate Limit 고려 시 4건) |
| Make 작업 수 | 하루 약 50작업 → 무료 플랜(월 1,000) 내 처리 |
| **Gemini 비용** | **무료** (Gemini Flash Latest 무료 티어: 분당 5건) |

### 비용 예상

| 항목 | 무료 플랜 기준 |
|------|---------------|
| Make | 월 1,000 작업 무료 |
| Google Gemini 2.5 Flash | 분당 5건, 일 500건 무료 |
| Notion API | 무료 |
| **합계** | **전체 무료 운영 가능** |

---

## 7. 노션 DB 스키마

### 데이터베이스명: `뉴스 요약 자동화`

| 속성명 | 타입 | 내용 | Make 매핑값 |
|--------|------|------|-------------|
| **이름** | Title | 기사 제목 | `{{2.title}}` |
| **Summary** | Rich Text | AI 3줄 요약 | `{{8.result}}` |
| **URL** | URL | 원문 링크 | `{{2.url}}` |
| **Published Date** | Date | 기사 발행일 | `{{2.RSS fields: pubdate}}` |
| **Source** | Text | 출처 피드명 | `AI타임스` (직접 입력, Map ON) |
| **Status** | Select | 처리 상태 | `완료` (직접 입력, Map ON) |

### 노션 저장 결과 예시

```
┌────────────────────────────────────────────────────┐
│ 📰 오픈AI, 애플의 핵심 엔지니어 영입                │
├────────────────────────────────────────────────────┤
│ 📝 요약 (Gemini 2.5 Flash 생성)                    │
│  • 애플의 혼합현실 헤드셋 핵심 엔지니어가 오픈AI 합류 │
│  • 차세대 AI 하드웨어 제품군 개발 담당 예정          │
│  • 메타와의 AI 하드웨어 경쟁 심화 전망              │
├────────────────────────────────────────────────────┤
│ 🔗 URL: https://www.aitimes.com/news/...           │
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
| ④ | Make 워크플로우 실행 성공 스크린샷 | [메이크 결과_1.png](./02_img/메이크%20결과_1.png) |
| ⑤ | Notion DB 저장 결과 스크린샷 | [노션_1.png](./02_img/노션_1.png) |

---

## 9. 팀 역할 및 개인 작업 요약

| 역할 | 담당 | 주요 작업 |
|------|------|-----------|
| 워크플로우 설계 | 전체 | Make 시나리오 구조 기획 및 모듈 연결 |
| RSS 피드 선정 | 전체 | AI 관련 국내 RSS 피드(AI타임스) 조사 및 선정 |
| AI 프롬프트 설계 | 전체 | Google Gemini Flash 요약 프롬프트 최적화 |
| 노션 DB 설계 | 전체 | 속성 스키마 정의 및 Make Legacy 모듈 매핑 설정 |
| 에러 처리 설계 | 전체 | RSS 날짜 필터, Gemini Rate Limit, Notion 연결 문제 해결 |
| 문서화 | 전체 | README 및 워크플로우 가이드 작성 |

> ℹ️ 이번 과제는 **개인 수행** 과제로 전 단계를 단독으로 진행하였습니다.

---

## 🔧 구현 과정에서 해결한 주요 이슈

| 이슈 | 원인 | 해결 |
|------|------|------|
| Notion DB ID 인식 불가 | 신버전 모듈과 Internal Token 비호환 | **Legacy 모듈**로 교체 + 수동 ID 입력 |
| RSS 0건 수집 | `Date created > now` 필터 오류 | RSS와 Text parser 사이 필터 삭제 |
| Gemini `limit: 0` | `gemini-2.0-flash` 2026.6 단종 | **Gemini Flash Latest** (2.5)로 변경 |
| Gemini 503 오류 | 서버 일시 과부하 | 재시도로 해결 (일시적) |
| Gemini 429 오류 | 무료 티어 5 RPM 한도 | RSS 수집 건수 4건으로 제한 권장 |

---

## 📝 참고 문서

- [Make 공식 문서](https://www.make.com/en/help)
- [Google AI Studio](https://aistudio.google.com)
- [Notion API 문서](https://developers.notion.com)
- [Make 워크플로우 상세 설정 가이드](./01_document/03_make_workflow_guide.md)

---

*이 README는 과제 수행 보고서로 작성되었습니다. Make 계정 설정 후 [워크플로우 가이드](./01_document/03_make_workflow_guide.md)를 참고하여 실제 시나리오를 구성하세요.*
