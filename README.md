
# 🚀 K-Defense Global Expansion Strategy Platform

![K-Defense Global Market Map](assets/global-market-map.png)




데이터 기반 분석을 통해 **한국 방산 기업의 글로벌 진출 전략**을 도출하는 프로젝트입니다.  
글로벌 방산 시장 데이터를 수집하고 분석하여 **권역별 수요를 파악하고 기업별 전략을 제안**합니다.

---

# 📊 Project Overview

최근 글로벌 국방비 증가와 국제 안보 환경 변화로 인해  
**방산 시장 수요가 급격히 확대**되고 있습니다.

본 프로젝트는 다음을 수행합니다.

- 🌍 글로벌 방산 시장 데이터 수집
- 📊 권역별 시장 세분화
- 🔎 방산 수요 분석
- 🏭 한국 방산 기업과 글로벌 시장 매칭
- 📈 전략 도출 및 웹 기반 시각화

---

# 🧭 Project Workflow


DATA → SELECTION → ANALYSIS → STRATEGY → WEB

---

# 📂 1. Data Collection


수집 데이터

- DART API (기업 공시 데이터)
- News API
- RSS Feed
- 글로벌 방산 전시회
- 글로벌 방산 시장 데이터

---

# 🧪 Data Processing

수집한 원천 데이터를 분석 가능한 형태로 가공하는 과정입니다.
전처리는 Jupyter Notebook에서 수행했고, 산출물은 CSV로 저장해 웹 플랫폼에서 사용합니다.

## 1-1. 기업 계약 데이터 정제 (`현대로템전처리.ipynb`)

DART 공시에서 받은 기업별 계약 원본 CSV를 방산 계약만 남기고 통합합니다.

1. **로드** — `현대로템_clear해야대.csv`를 `cp949` 인코딩으로 읽음
2. **방산 계약 선별** — 계약 97건 중 방산에 해당하는 30개 행만 수기 선별해 추출
3. **수기 보정**
   - 계약명이 비공개인 건은 `경영상 비밀 유지`로 표기
   - 중복 행 2건 제거
   - 공시 형식 문제로 깨진 계약금액 2건을 원 공시값으로 교정
   - → `현대로템_국방계약_clear.csv`
4. **6개사 통합** — 풍산 / 현대로템 / 한화에어로 / 한화시스템 / KAI / LIG넥스원의 정제 CSV에 각각 `기업명` 컬럼을 붙여 `concat`, 빈 컬럼(`Unnamed: 9`) 제거
   - → **`국방산업기업_통합_clear.csv`** (전체 분석의 기준 테이블)

이 통합 테이블을 기업별·국가별로 집계해 **국가별 거래 비중** 도넛 차트를 생성합니다
(금액 기준 / 계약 횟수 기준 × 전체 / 방산만 = 기업당 4종).
예: 한화에어로스페이스 2016~2025 방산 계약은 **폴란드가 금액 기준 70.4%**로 압도적이고,
루마니아 6.7%, 미국 4.1%, 영국·독일·싱가폴 5.5% 순입니다.

## 1-2. 국가별 국방비·무기이전 분석 (`국방비지출.ipynb`)

국가를 **어디에 팔 것인가** 기준으로 점수화하는 핵심 전처리입니다.

**① SIPRI 국방비/무기이전 원본 정제**
- `국방비지출_및_무기이전_*.xlsx` 로드 → 설명용 0행 제거
- 병합셀로 깨진 지역/국가 컬럼을 `country`(국가) / `region`(권역)으로 복원, `region`은 `ffill()`
- `2024`, `2023.1` 같은 모호한 컬럼명을 `arms_export_2024`, `milex_gdp_2023` 등 의미 기반으로 리네이밍
- 값 정제: 괄호 주석 제거, 천단위 콤마 제거, `-`는 결측 처리 후 `float` 변환
- `소계` 행 제거
- 결측 처리 정책을 값의 성격에 따라 분리
  - 비율 지표(GDP 대비 국방비 등) → **평균값**으로 대체
  - 금액 지표(무기 수출입액) → **0**으로 대체 (거래 없음을 의미)

**② 공공데이터포털 경제지표 결합**
- `OverviewEconomicService` API를 `numOfRows=100`으로 페이지네이션하며 전량 수집 (`totalCount` 기준, 페이지당 0.2초 sleep)
- GDP / 1인당 GDP / 성장률 / 수출입액을 숫자형 변환, 핵심 GDP 결측은 **중앙값**으로 보정
- 국가명 공백을 제거한 `country_norm` 키를 만들어 SIPRI 테이블과 `left join`

**③ 파생 지표 생성**

| 지표 | 정의 |
|------|------|
| `milex_burden` | 2023년 GDP 대비 국방비 비율 (국방비 부담 수준) |
| `milex_trend` | `milex_gdp_2023 - milex_gdp_2022` (국방비 증감 추세) |
| `arms_import_dependency` | `수입 / (수입 + 수출 + 1)` — 자체 생산 없이 수입에 의존하는 정도 |
| `log_gdp`, `log_gdp_pc` | GDP 스케일 왜곡 보정용 `log1p` 변환 |

**④ 한국 방산 적합도 점수 (`korea_fit_score`)**

```
korea_fit_score = (1 / 1인당 GDP) × 0.3
                + 국방비 증감 추세     × 0.4
                + 무기 수입 의존도     × 0.3
```

"국방비를 늘리는 중이고, 무기를 수입에 의존하며, 고가 장비 구매력이 제한적인 국가"에 높은 점수를 주는 설계입니다.
이 점수로 전체 국가를 정렬해 상위권을 후보 시장으로 추립니다.

**⑤ 검증 시각화**
- 경제·군사 지표 간 상관 히트맵 (금액 지표는 `log1p` 변환본으로 재확인)
- 무기 수입 상위 15개국, 추정 국방비 절대 규모 Top 15
- GDP 대비 국방비 2022 → 2023 변화 추이

**⑥ SIPRI 권역별 이전량으로 그룹 확정**
- `regional-transfers_*.csv`를 `skiprows=10`으로 로드(상단 메타 헤더 제거), 병합셀 `ffill` 복원
- 연도 컬럼을 long-format으로 `melt`, `Regional total` 행을 플래그로 분리해 전세계 총량과 국가별 수출량을 각각 집계
- **2022년(러시아-우크라이나 전쟁) 기준선**을 그어 전후 수출량 변화를 확인

최종적으로 시장을 3그룹으로 확정했습니다.

| 그룹 | 기준 | 국가 |
|------|------|------|
| 수출 10% 이상 | 이미 주요 수출국 | 미국, 영국, 폴란드 |
| 수출 10% 미만 | 진입 여지 있음 | 독일, 태국, 브라질, 사우디, UAE, 에스토니아, 루마니아 |
| 신시장 | 미수출 | 에콰도르, 칠레, 우즈베키스탄, 콜롬비아, 아르헨티나 |

## 1-3. 무기 수요 키워드 추출 (`뉴스크롤링.ipynb`)

각 국가가 **무엇을 원하는가**를 뉴스 텍스트에서 역산합니다.

1. **전시회 목록 기준** — `전시회.csv`의 국가별 방산 전시회명(`tit`)을 검색 키워드로 사용
   (대상 국가 집합과 전시회 데이터를 집합 연산으로 대조해 커버리지 확인)
2. **NewsAPI 수집** — 전시회명으로 영문 기사 최대 100건 수집
3. **텍스트 전처리** — 기사 제목에서 알파벳 외 문자 정규식 제거 → 소문자 토큰화 → NLTK 불용어 제거 → `WordNetLemmatizer`로 원형 복원
4. **무기 키워드 필터** — 항공/지상/해상/무장/기술/일반군사 6개 범주로 정의한 `WEAPON_KEYWORDS` 사전에 걸리는 토큰만 남김
5. **집계** — `(nation, exhibit, weapon_keyword, count)` 레코드로 쌓은 뒤 국가 단위로 `groupby`, 국가별 상위 5개 키워드 추출
6. **시각화** — 국가별 워드클라우드 + Top 10 막대그래프
   - → `조사국가별전시회별무기키워드.csv`, `중동국가별전시회별무기키워드.csv`

이 결과가 아래 **3. Market Analysis**의 권역별 수요(동유럽 = 자주포·전차, 중동 = 드론·미사일, 남미 = 항공기)로 이어집니다.

## 데이터 처리 요약

```
DART 공시 CSV ─┐
               ├─ 수기 방산 선별 + 6개사 통합 → 국방산업기업_통합_clear.csv → 국가별 거래 비중 차트
               ┘

SIPRI xlsx ─┐
            ├─ 정제 + 국가명 정규화 조인 → 파생지표 → korea_fit_score → 시장 3그룹
공공데이터 GDP API ┘

전시회.csv → NewsAPI 기사 → 토큰화·표제어 → 무기 키워드 필터 → 국가별 수요 Top N
```

> ⚠️ 노트북 원본에는 NewsAPI 키와 공공데이터포털 ServiceKey가 하드코딩되어 있습니다.
> 재현 시에는 본인 키를 발급받아 환경변수(`.env`)로 주입하세요.

---

# 🌎 2. Market Selection

글로벌 방산 시장을 **3개 그룹**으로 세분화합니다.

| Group | Description |
|------|-------------|
| Group 1 | 기술 선진국 |
| Group 2 | 신흥 방산 수요 시장 |
| Group 3 | 신규 시장 |


---

# 🔍 3. Market Analysis

권역별 주요 방산 수요 분석

| Region | Demand |
|------|------|
| Eastern Europe | K9 Self-Propelled Artillery / K2 Tank |
| Middle East | Drone / Missile |
| South America | Jet / Aircraft |

---

# 🎯 4. Strategy

한국 방산 기업과 글로벌 시장 매칭 전략

| Region | Target Company |
|------|----------------|
| USA / UK / Germany | Hanwha Aerospace |
| Eastern Europe | Hyundai Rotem |
| Middle East | LIG Nex1 |
| South America | Korea Aerospace Industries |

---

# 💻 5. Web Platform

분석 결과를 **웹 플랫폼으로 시각화하여 제공**

Features

- 글로벌 방산 시장 분석
- 국가별 방산 수요 시각화
- 기업-시장 매칭 전략
- 데이터 기반 전략 제안

---

# 📁 Repository Structure

```
K-Defense-Global-Expansion
│
├─ frontend         # React 기반 프론트엔드
├─ backend          # Python 기반 백엔드 API
├─ assets           # 이미지 / 시각화 결과
├─ docs
│   └─ reference
│       └─ k_방산 글로벌화 전략.pdf
└─ README.md
```

---

# 🏭 Target Companies

본 프로젝트는 다음 한국 방산 기업을 중심으로 분석합니다.

- Hanwha Aerospace
- Hyundai Rotem
- LIG Nex1
- Korea Aerospace Industries
- Hanwha Systems
- Poongsan

---

# ⚡ Getting Started

로컬에서 프로젝트를 실행하는 방법입니다. (Windows / Python 3.10+ / Node 18+)

## 1. Clone

```bash
git clone https://github.com/pbzz1/K-Defense-Dashboard.git
cd K-Defense-Dashboard
```

## 2. Backend (Flask, port 5000)

```bash
cd backend
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

`backend/.env` 파일을 만들고 한국투자증권 OpenAPI 키를 넣습니다.
(키가 없으면 서버가 시작 시 `RuntimeError`로 종료됩니다.)

```
KIS_APP_KEY=your_app_key
KIS_APP_SECRET=your_app_secret
```

```bash
python app.py
```

→ http://localhost:5000

## 3. Frontend (React, port 3000)

새 터미널에서:

```bash
cd frontend
npm install
npm start
```

→ http://localhost:3000

프론트엔드는 API 주소를 `http://localhost:5000`으로 하드코딩해서 호출하므로,
백엔드를 먼저 띄운 상태에서 사용해야 합니다.

## Notes

- 차트의 한글 라벨은 `C:/Windows/Fonts/malgun.ttf`(맑은 고딕)를 사용합니다. macOS / Linux에서는 `backend/app.py`의 `FONT_PATH`를 수정해야 합니다.
- `backend/services/marketstack.py`는 `MARKETSTACK_API_KEY` 설정이 필요하며 현재 `app.py`에 연결되어 있지 않습니다.
- 배포용 정적 빌드는 `cd frontend && npm run build`.

---

# 🛠 Tech Stack

| Category | Technology |
|--------|------------|
| Language | Python |
| Frontend | React |
| Data | Web Crawling |
| API | DART API / News API |
| Data Source | RSS |
| Analysis | Data Analysis |
| Visualization | Data Visualization |

---

# 📚 Reference

docs/reference/k_방산 글로벌화 전략 (박사님과 싱싱미역줄기들).pdf

---

# 👨‍💻 Team

**ACON Project**

- 김다은
- 김태훈
- 양유민
- 조성진
- 조준형
