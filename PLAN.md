# 면접 대비 뉴스 수집 자동화 – 기획안

> 목적: 지원한 회사별로 "회사 뉴스 + 산업/경제/시사 뉴스"를 매일 자동 수집·정리해서
> 면접 전에 한 번에 읽을 수 있는 리포트를 만든다.
> 회사 목록은 설정 파일 하나만 고치면 추가/삭제/수정이 되도록 한다.

---

## 1. 목표와 범위

### 꼭 되어야 하는 것 (MVP)
- 회사 목록을 **설정 파일(`companies.yaml`)** 로 관리 → 회사 추가/삭제/키워드 수정이 쉬움
- 회사별 뉴스 수집 (회사명 + 별칭 + 핵심 키워드로 검색)
- 공통 뉴스 수집 (경제, 산업, 시사/상식 등 카테고리별)
- 중복 제거 + 이미 본 기사 제외
- **일일 리포트(마크다운)** 생성 → `reports/YYYY-MM-DD/` 아래에 회사별 파일 + 공통 파일
- 매일 정해진 시간에 자동 실행 (GitHub Actions cron)

### 있으면 좋은 것 (확장)
- LLM 요약: 기사 3줄 요약, 회사별 "이번 주 핵심 이슈", **예상 면접 질문** 자동 생성
- 알림 전송: 이메일 / Slack / Telegram / Notion 중 택1
- 관련도 점수로 정렬 (제목에 회사명 포함 > 본문에만 포함 등)
- 면접일 기준 "D-7 집중 브리핑" 모드

### 하지 않는 것
- 유료 뉴스 본문 크롤링(저작권/약관 이슈) → 제목·요약·링크만 저장
- 웹 UI (처음엔 마크다운 리포트로 충분, 필요해지면 나중에)

---

## 2. 전체 흐름

```
companies.yaml / sources.yaml
          │
          ▼
┌─────────────────┐   ┌─────────────────┐
│ 회사별 검색 수집  │   │ 카테고리 RSS 수집 │
│ (Google News RSS │   │ (경제/산업/시사)  │
│  + Naver 검색 API)│   │                 │
└────────┬────────┘   └────────┬────────┘
         └──────────┬──────────┘
                    ▼
          정규화 → 중복 제거 → 이미 본 기사 제외
                    ▼
          관련도 점수 계산 / 정렬 / 상위 N개
                    ▼
          (선택) LLM 요약 + 예상 질문 생성
                    ▼
          마크다운 리포트 생성 (reports/날짜/회사.md)
                    ▼
          git commit & push  /  (선택) 알림 전송
```

---

## 3. 회사 설정 방식 (핵심 요구사항)

`companies.yaml` 하나로 관리. 회사가 늘어나면 블록만 추가.

```yaml
companies:
  - id: samsung-sds
    name: 삼성SDS
    aliases: ["삼성에스디에스", "Samsung SDS"]   # 검색 시 OR 조건
    industry: ["IT서비스", "클라우드", "생성형 AI"] # 산업 뉴스 키워드
    competitors: ["LG CNS", "SK C&C"]            # 경쟁사 동향도 같이 수집
    extra_keywords: ["브리티웍스", "삼성클라우드플랫폼"]
    interview_date: 2026-10-20                    # 선택. D-day 모드용
    active: true                                  # false면 수집 중단(삭제 안 해도 됨)

  - id: toss
    name: 토스
    aliases: ["비바리퍼블리카", "Toss"]
    industry: ["핀테크", "인터넷은행", "간편결제"]
    active: true
```

`sources.yaml` 은 공통 뉴스 카테고리를 관리.

```yaml
general:
  - id: economy
    name: 경제
    feeds:
      - https://news.google.com/rss/headlines/section/topic/BUSINESS?hl=ko&gl=KR&ceid=KR:ko
  - id: tech
    name: IT/과학
    feeds:
      - https://news.google.com/rss/headlines/section/topic/TECHNOLOGY?hl=ko&gl=KR&ceid=KR:ko
  - id: current-affairs
    name: 시사/상식
    feeds:
      - https://news.google.com/rss?hl=ko&gl=KR&ceid=KR:ko
```

---

## 4. 뉴스 소스 후보

| 소스 | 장점 | 단점 | 우선순위 |
|---|---|---|---|
| **Google News RSS** (검색 URL `rss/search?q=...`) | API 키 불필요, 한국어 지원, 즉시 사용 가능 | 본문 없음(제목+링크+요약 일부) | **1순위 (MVP)** |
| **Naver 뉴스 검색 API** | 국내 뉴스 커버리지 최고, 무료 (일 25,000회) | 클라이언트 ID/Secret 발급 필요 | 2순위 |
| 언론사 공식 RSS (연합, 한경, 매경 등) | 안정적, 카테고리별 제공 | 회사명 필터링은 직접 해야 함 | 공통 뉴스용 |
| DART 공시 RSS | 상장사라면 공시(실적/투자)가 면접 소재로 강력 | 비상장사 불가 | 확장 |

→ **MVP는 Google News RSS 하나로 시작**, 키 발급 후 Naver 추가.

---

## 5. 기술 스택 (제안)

- **언어**: Python 3.11+
- **라이브러리**: `feedparser`(RSS), `requests`, `pyyaml`, `python-dateutil`
- **저장**: `data/seen.json` (이미 수집한 기사 URL 해시) – DB 없이 파일로 충분
- **스케줄러**: **GitHub Actions cron** (매일 아침 07:00 KST) → 리포트를 레포에 커밋
  - 서버/노트북 켜둘 필요 없음, 무료, 이 레포가 이미 GitHub에 있음
- **LLM(선택)**: Anthropic Claude API – 요약·예상 질문 생성. API 키는 GitHub Secrets에 보관
- **알림(선택)**: Slack Incoming Webhook 또는 Telegram Bot – 한 줄 요약 + 리포트 링크

---

## 6. 폴더 구조

```
news_automate/
├── companies.yaml              # 지원 회사 목록 (사용자가 편집하는 파일)
├── sources.yaml                # 공통 뉴스 카테고리/피드
├── requirements.txt
├── src/news_automate/
│   ├── config.py               # yaml 로드/검증
│   ├── models.py               # Article 데이터클래스
│   ├── collectors/
│   │   ├── google_news.py      # 회사별 검색 RSS
│   │   ├── naver.py            # Naver 검색 API (2단계)
│   │   └── rss.py              # 공통 카테고리 RSS
│   ├── dedupe.py               # URL/제목 유사도 기반 중복 제거, seen.json 관리
│   ├── scoring.py              # 관련도 점수
│   ├── summarize.py            # (선택) LLM 요약/예상 질문
│   ├── report.py               # 마크다운 리포트 생성
│   └── main.py                 # CLI 진입점: `python -m news_automate run`
├── data/seen.json
├── reports/
│   └── 2026-10-11/
│       ├── README.md           # 그날의 목차
│       ├── samsung-sds.md
│       ├── toss.md
│       └── general.md
└── .github/workflows/daily.yml
```

---

## 7. 리포트 예시 (회사별 파일)

```markdown
# 삼성SDS – 2026-10-11 뉴스 브리핑  (면접 D-9)

## 🔥 회사 직접 관련 (5건)
- **삼성SDS, 생성형 AI 플랫폼 '패브릭스' 금융권 확대** – 한국경제 · 10-10
  > 요약 한 줄 …  [링크]

## 🏭 산업/경쟁사 동향 (5건)
- LG CNS, 클라우드 MSP 매출 30% 성장 – 전자신문 · 10-09

## 📌 이번 주 핵심 이슈 (LLM 요약, 선택)
1. …

## ❓ 예상 면접 질문 (LLM, 선택)
1. "최근 패브릭스의 금융권 확대에 대해 어떻게 생각하나요?"
```

공통 파일(`general.md`)은 경제/IT/시사 카테고리별 상위 10건씩.

---

## 8. CLI 사용법 (설계)

```bash
python -m news_automate run                 # 전체 수집 + 리포트
python -m news_automate run --company toss  # 특정 회사만
python -m news_automate run --days 3        # 최근 3일치
python -m news_automate add "카카오"          # companies.yaml에 블록 추가 (대화형)
python -m news_automate list                # 등록된 회사 목록
```

---

## 9. 단계별 진행 계획

| 단계 | 내용 | 결과물 |
|---|---|---|
| **1. MVP** | companies.yaml + Google News RSS 수집 + 중복 제거 + 마크다운 리포트 + GitHub Actions 매일 실행 | 매일 아침 `reports/날짜/`에 리포트가 자동 커밋됨 |
| 2. 품질 | Naver 검색 API 추가, 관련도 점수, 제목 유사도 중복 제거 | 더 정확한 국내 뉴스 |
| 3. 면접 특화 | Claude API 요약, "핵심 이슈", "예상 면접 질문", D-day 모드 | 읽기만 하면 되는 브리핑 |
| 4. 전달 | Slack/Telegram 알림, DART 공시 연동 | 폰으로 바로 확인 |

---

## 10. 결정이 필요한 것

1. **뉴스 소스**: Google News RSS만으로 시작할지, Naver API 키를 발급받아 처음부터 같이 쓸지
   - 추천: 1단계는 Google News RSS, 키 발급되면 2단계에서 Naver 추가
2. **LLM 요약 사용 여부**: Claude API 키가 있는지 / 비용 허용 범위 (하루 회사 5개 기준 수십 원 수준)
3. **전달 채널**: 레포 커밋만으로 충분한지, Slack/Telegram/이메일 중 원하는 것
4. **실행 시각**: 매일 07:00 KST 추천 (면접 당일 아침 확인용)
5. **처음 등록할 회사 목록**: 회사명 + 지원 직무를 알려주면 키워드 세팅까지 같이 할 수 있음
