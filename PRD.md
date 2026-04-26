# VOC 자동 분류 · 분석 · 대시보드 시스템 기획서

> **프로젝트명**: VOC Auto Classifier & Dashboard
> **작성일**: 2026-04-26
> **작성자**: Wooju Kang (dnwn4120@gmail.com)
> **상태**: MVP 구축 완료 / 자동화·리포트 기능 추가 예정

---

## 1. 프로젝트 개요

### 1.1 배경
온라인 교육 플랫폼(블루타이거)을 운영하면서 카카오톡, 이메일, 전화, 인스타DM 등 다양한 채널로 들어오는 **고객의 소리(VOC)** 를 수작업으로 분류하고 패턴을 찾는 데 많은 시간이 소요되었다. 분류 기준이 사람마다 달라지는 일관성 문제, 누적된 데이터를 기간별로 추이 분석할 수 없는 가시성 부족 문제도 함께 존재했다.

기존 방식: VOC 한 건 → ChatGPT에 붙여넣기 → 카테고리 추정 → 시트에 수동 입력 → 시간 소모 ↑, 패턴 발견 ↓

### 1.2 목적
**Google Sheets에 VOC가 쌓이면 자동으로 분류 → 패턴 분석 → 개선안 초안 생성 → 대시보드 시각화 + 주간 리포트 발송**까지 흐르는 파이프라인 구축.

### 1.3 기대 효과
- 분류 작업 시간 **수 시간 → 수 분** (170건 기준)
- 동일한 기준으로 분류된 일관된 데이터셋 확보
- 일/주/월 단위 추이를 시각적으로 모니터링
- 핵심 키워드·개선안을 데이터로 근거 있게 도출

---

## 2. 서비스 환경 분석

### 2.1 비즈니스 구조 (블루타이거)
- **운영 방식**: 1년 6번 정기 개강 (1·3·5·7·9·11월), 각 월 첫째 주 토요일
- **수업 형태**: 줌 화면 공유 영상 시청 → 노션 페이지에 과제 제출 → 줌 실시간 멘토링
- **수업 기간**: 8주 정규 + 9주차 토요일 종강세션 (우수 수강생 발표회)
- **클래스**: 한 기수당 약 20개 클래스 (클래스_A ~ 클래스_T)
- **반 구성**: 한 클래스당 2개 반 (예: A-1, A-2)
- **멘토**: 반당 멘토 1명 매칭
- **수강 옵션**: 한 명의 수강생이 동시 또는 순차로 여러 클래스 수강 가능

### 2.2 VOC 채널
카카오톡, 이메일, 전화, **채널톡**, 웹문의, **주차별 만족도 설문**

> 📌 채널이 추가/변경되어도 `01_Raw`의 통합 형식만 맞추면 분류·집계·대시보드는 그대로 동작 (확장성 ↑)

### 2.3 핵심 분석 포인트
- 각 클래스 **개강일+2주 구간**에 들어오는 VOC가 가장 중요 (초기 이탈/만족도 결정 시기)
- 분석 결과 기준 비중: **서비스 이용방법 63% / LMS·강의시청 오류 18% / 기타 19%**

---

## 3. 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│             [Google Sheets + Apps Script]               │
│                                                         │
│   01_Raw  ──onChange──▶  Claude API  ──▶  02_Classified │
│                              ↓                          │
│                          (분류·요약·키워드·개선안)        │
│                              ↓                          │
│                          04_Aggregates                  │
│                              ↓                          │
│                          doGet (JSON API)               │
└──────────────────────────────┬──────────────────────────┘
                               ↓ fetch
┌──────────────────────────────────────────────────────────┐
│                  [Vercel - index.html]                   │
│   - KPI 카드 / 일·주·월 추이 차트                          │
│   - Top 10 키워드 / 개선안 초안                           │
│                                                          │
│   URL: https://voc-dashboard-iota.vercel.app/            │
└──────────────────────────────────────────────────────────┘

                ─── 추가 흐름 (5/1 이후 활성) ───

[주간 트리거] ──▶ Claude API ──▶ Google Docs ──▶ Slack/이메일 발송
```

---

## 4. 기능 명세

### 4.1 VOC 입력 및 저장 (`01_Raw`)
- 컬럼: timestamp, channel, 수강생ID, 이름, 이메일, 클래스, 반, 멘토, 기수, 수강상태, content
- 수강생 정보를 함께 저장하여 분류 시 컨텍스트 제공 + 분석 시 클래스/반/멘토별 분석 가능

### 4.2 자동 분류 (Claude API)
- **모델**: `claude-sonnet-4-6` (가성비·품질 균형)
- **방식**: Tool use (`record_classifications`)로 JSON 스키마 강제
- **배치**: 한 번에 10건씩 묶어 호출 (비용·속도 최적화)
- **결과 컬럼** (02_Classified):
  - `category`, `subcategory` (28개 카테고리 마스터에서 매칭)
  - `sentiment`: positive / neutral / negative
  - `severity`: 1~5
  - `summary`: 한 문장 요약
  - `keywords`: 핵심 키워드 1~3개
  - `improvement_hint`: 개선안 한 문장
  - `processed_at`, `model_version`: 메타 정보

### 4.3 카테고리 체계 (`03_Categories`)

총 28개 (대분류 9개 / 소분류 28개):

| 대분류 | 소분류 |
|---|---|
| 수강 신청/결제 | 결제 오류 / 환불 요청 / 다중 클래스 결제 |
| 클래스 운영 | 개강·종강 일정 / 커리큘럼·진도 / 자료 미제공 / 줌 수업 안내 |
| 멘토/강사 | 멘토 응대 / 피드백 품질 / 답변 속도 |
| 반/기수 관련 | 반 변경 요청 / 기수 변경 / 동기·분위기 |
| 플랫폼/기술 | 줌 영상·음성 오류 / 로그인·계정 / **노션 페이지 접근** / LMS 접속 |
| 학습 관리 | 출결 / 과제 제출 / 수료증·수료 조건 / 진도율 |
| 고객 응대(CS) | 응대 만족도 / 응대 속도 / 안내 부족 |
| 콘텐츠 품질 | 강의 난이도 / 자료 오류 / 업데이트 요청 |
| 기타 | - |

### 4.4 일/주/월 집계 (`04_Aggregates`)
- 자동 함수 `rebuildAggregates()`가 02_Classified를 읽어 갱신
- 컬럼: period_type (day/week/month), period_key, category, count, avg_severity, top_keywords

### 4.5 대시보드 (Vercel 호스팅)
- **기술**: HTML + Tailwind CSS (CDN) + Chart.js (CDN) + Pretendard 폰트
- **데이터 연결**: Apps Script `doGet`이 JSON 반환 → 프론트에서 `fetch`
- **구성**:
  - 헤더: 총 분류 건수
  - 추이 차트: 일별/주별/월별 탭 전환 (Chart.js)
  - Top 10 키워드 표
  - 개선안 초안 카드 (카테고리 배지 + 개선 제안)
- **URL**: https://voc-dashboard-iota.vercel.app/
- **자동 배포**: GitHub repo에 push → Vercel이 감지 → 자동 재배포

### 4.6 자동 트리거 (계획)
- **`onChange` 트리거**: `01_Raw`에 새 VOC 행 추가 시 → 자동 분류 → `02_Classified` 저장
- 디바운스/Lock으로 중복 호출 방지
- 일일 호출 카운터로 비용 가드레일

### 4.7 주간 리포트 (계획)
- 매주 월요일 09:00 시간 기반 트리거
- 지난 주 `04_Aggregates` 데이터 + 대표 VOC를 Claude에 전달 → 마크다운 리포트 생성
- 출력 채널:
  - Google Docs 자동 생성 (도메인 사용자 보기 권한)
  - 이메일 (`MailApp`)
  - Slack Incoming Webhook
- `05_Reports`에 발송 메타 기록

---

## 5. 데이터 모델

### 5.1 시트 구성 (6개 탭)

| 탭 | 역할 | 행 수 (현재) |
|---|---|---|
| `00_Students` | 수강생 마스터 (3기수 × 20클래스 × 2반 × 3명) | 360 |
| `01_Raw` | VOC 원본 입력 | 170 (더미) |
| `02_Classified` | 분류 결과 | 50 (한도 도달, 5/1 후 120건 추가) |
| `03_Categories` | 카테고리 마스터 | 28 |
| `04_Aggregates` | 일/주/월 × 카테고리 집계 | 자동 갱신 |
| `05_Reports` | 발송된 리포트 메타 | 0 (5/1 후) |

### 5.2 더미 데이터 분포
- 기간: **2023-07-01 ~ 2023-11-18** (7월·9월·11월 개강 + 9월 종강+2주)
- 분석 구간 비중: **서비스 63 / LMS 18 / 기타 19**
- 11월 분석구간: 서비스만 30% 감소 → 22 / 9 / 9 (총 40건)

---

## 6. 기술 스택

| 영역 | 기술 |
|---|---|
| 데이터 저장 | Google Sheets |
| 백엔드/자동화 | Google Apps Script (V8 런타임) |
| AI 분류 엔진 | Anthropic Claude API (`claude-sonnet-4-6`) |
| 인증/시크릿 | Apps Script Properties Service |
| API 게이트웨이 | Apps Script `doGet` (JSON) |
| 프론트엔드 | HTML + Tailwind CSS + Chart.js |
| 폰트 | Pretendard (CDN) |
| 호스팅 | Vercel (정적) |
| 코드 저장소 | GitHub |
| 알림 채널 | Gmail (MailApp) + Slack Webhook (예정) |

---

## 7. 진행 현황

### ✅ 완료
- [x] Google Sheet 6개 탭 세팅 (`Code.gs / setupSheets`)
- [x] Claude API 연결 + 인증 (`Config.gs` + `ClaudeClient.gs`)
- [x] 카테고리 28개 입력 (`SeedCategories.gs / seedCategories`)
- [x] 더미 학생 360명 생성 (`SeedStudents.gs / seedStudents`)
- [x] 더미 VOC 170건 생성 (`SeedVOC.gs / seedDummyVOC`)
- [x] 분류기 구현 (`Classifier.gs / classifyOne, classifyAll`)
- [x] 50건 분류 완료 (Claude Sonnet 4.6, Tool use 기반)
- [x] 분류 비중 검증 (`RatioCheck.gs / checkRatio`)
- [x] 일/주/월 집계 (`Aggregator.gs / rebuildAggregates`)
- [x] HTML 대시보드 (`index.html`)
- [x] GitHub repo 생성 (`voc-dashboard`)
- [x] Vercel 배포 (https://voc-dashboard-iota.vercel.app/)
- [x] 5/1 자동 알림 트리거 (`Reminder.gs / setupReminder`)

### ⏳ 진행 예정 (한도 상향 후 또는 5/1 이후)
- [ ] 남은 120건 분류 완료 (`classifyAll` 재실행)
- [ ] `onChange` 트리거 — 새 VOC 자동 분류
- [ ] 주간 리포트 (`Reporter.gs`) — Slack/이메일/Docs
- [ ] 카테고리 자동 추출 (Bootstrap, 신규 패턴 감지)
- [ ] 분류 기준 명세 강화 (severity·sentiment 회사 기준)

---

## 8. 운영 가이드

### 8.1 새 VOC 추가
1. `01_Raw` 탭에 행 추가 (수강생 정보 + content)
2. 수동: `classifyAll` 실행 → 51번째부터 자동 분류
3. 자동 트리거 활성 후: 행 추가만 해도 자동 분류

### 8.2 대시보드 데이터 갱신
1. `Aggregator.gs / rebuildAggregates` 실행 → 04_Aggregates 갱신
2. 대시보드는 `doGet` API를 fetch → **새로고침만 해도 최신 데이터** 표시

### 8.3 카테고리 변경 시
1. `03_Categories`에서 카테고리 추가/수정/`active=FALSE`
2. 다음 분류부터 자동 반영

### 8.4 비용 모니터링
- Anthropic Console → Usage
- 분류 1건당 약 $0.005~0.01 (Sonnet 4.6 기준)
- 일일 한도: Apps Script Properties로 카운터 관리 (예정)

### 8.5 코드 수정 후 대시보드 반영
1. 로컬에서 `index.html` 수정 → `Cmd+S`
2. VSCode Source Control → commit → push
3. Vercel이 1~2분 내 자동 재배포

### 8.6 대시보드 디자인 수정 (claude.ai 활용)

복잡한 코드 수정 없이 **claude.ai 웹 채팅**으로 디자인을 빠르게 바꿀 수 있다.

**흐름**:
1. claude.ai 접속 → 새 대화
2. VSCode의 `index.html` **코드 통째로 복사** → 붙여넣기
3. 변경 요청 작성 (예: "다크모드로 변경", "차트를 도넛형으로", "더 미니멀하게")
4. Claude가 응답한 새 코드 → VSCode `index.html`에 **통째로 덮어쓰기**
5. `Cmd + S` 저장 → Source Control commit → push
6. Vercel 자동 재배포 (1~2분 후 반영)

**주의사항**:
- ⚠️ 변경 요청 끝에 **"data fetch 로직 (API_URL, fetch 등)은 그대로 유지"** 반드시 명시 → 데이터 연결이 안 깨짐
- 큰 변경 후엔 로컬(`python3 -m http.server`)에서 먼저 확인 → 정상 동작 시 push
- Claude 응답 코드가 비정상적으로 짧으면 일부 생략됐을 가능성 → "전체 코드를 빠짐없이 다시 출력해줘" 요청
- 한 번에 여러 변경보다 **단일 변경 단위로** 진행해야 디버깅이 쉬움
- 매우 큰 레이아웃 변경 시엔 claude.ai의 **Artifacts** 기능으로 실시간 미리보기 활용

**효과적인 프롬프트 템플릿**:
```
이 대시보드 HTML 코드를 다음과 같이 바꿔줘:

1. (변경사항 1)
2. (변경사항 2)
3. (변경사항 3)

전체 코드를 빠짐없이 다시 출력해줘.
data fetch 로직 (API_URL, fetch 등)은 그대로 유지.
```

**자주 쓰는 변경 예시**:
| 목적 | 요청 키워드 |
|---|---|
| 다크모드 | "배경 #0f172a, 카드 #1e293b로 다크모드 변환" |
| 색상 변경 | "메인 컬러를 #2563eb에서 #7c3aed로" |
| 차트 스타일 | "라인 차트를 도넛형으로 / 막대를 가로형으로" |
| 모바일 최적화 | "모바일 우선 반응형으로 정렬, 터치 친화" |
| 호버 효과 | "카드에 부드러운 그림자 + 살짝 떠오르는 호버" |
| 다국어 | "헤더와 라벨을 영어로 (데이터는 그대로)" |

---

## 9. 향후 계획

### 9.1 단기 (1주)
- 자동 트리거 + 주간 리포트 완성
- 분류 기준(severity·sentiment) 회사 표준화

### 9.1.1 실제 채널 연동 (PoC)
- **채널톡 Webhook** → Apps Script `doPost` → `01_Raw` 자동 입력 (가장 우선)
- **Gmail** `GmailApp` → 시간 트리거로 새 메일 → 자동 입력
- **구글폼 (주차별 만족도 설문)** → 응답 시트 → 변환 후 `01_Raw` 추가
- **카카오톡 채널** → Kakao i 오픈빌더 + Webhook (또는 Zapier)

### 9.2 중기 (1개월)
- 멘토별 / 클래스별 / 반별 드릴다운 대시보드
- 카테고리 자동 추출(Bootstrap)으로 신규 패턴 감지
- VOC 응대 SLA(응답시간) 측정

### 9.3 장기
- Slack 봇 연동: VOC가 들어오면 자동 분류된 정보와 함께 채널 알림
- 환불 위험도 예측 (감정·심각도·과거 패턴 학습)
- 클래스/기수별 만족도 점수화

---

## 10. 부록

### 10.1 디렉토리 구조 (Apps Script)

```
VOC 자동화 (Google Sheets)
└── Apps Script
    ├── Code.gs              ── setupSheets()
    ├── Config.gs            ── 상수 + getSecret()
    ├── ClaudeClient.gs      ── callClaude(), testClaudeConnection()
    ├── SeedCategories.gs    ── seedCategories()
    ├── SeedStudents.gs      ── seedStudents()
    ├── SeedVOC.gs           ── seedDummyVOC()
    ├── Classifier.gs        ── classifyOne(), classifyAll()
    ├── RatioCheck.gs        ── checkRatio()
    ├── Aggregator.gs        ── rebuildAggregates()
    ├── WebDashboard.gs      ── doGet(), getDashboardData(), exportDashboardHtml()
    └── Reminder.gs          ── setupReminder(), notifyMay1()
```

### 10.2 디렉토리 구조 (프론트엔드)

```
voc-dashboard/
├── index.html              ── 대시보드 페이지
└── PRD.md                  ── 본 문서
```

### 10.3 핵심 URL
- **대시보드**: https://voc-dashboard-iota.vercel.app/
- **GitHub**: https://github.com/dnwn4120-max/voc-dashboard
- **Anthropic 콘솔**: https://console.anthropic.com/

---

*문서 끝*
