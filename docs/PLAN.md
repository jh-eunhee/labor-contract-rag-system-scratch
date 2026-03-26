# Hello RAG System - 구현 계획서

## 1. 프로젝트 개요

**목적:** Vector DB 기반 RAG 시스템 + FastAPI 백엔드 구축

두 가지 자율점검 서비스를 제공한다:
1. **기초노동질서 자율점검** — 근로계약서/임금명세서 PDF 분석 → 임금 계산 → 법령 RAG → 리포트
2. **산재위험요소 자율점검** — 현장 사진 → 이미지 캡셔닝 → KOSHA Guide RAG → 위험성 평가

**참고 프로젝트:**
- `basic-labor-standards-self-Inspection` — 3개 엔진(Rules/Retrieval/Generation) 마이크로서비스 구조, 임금 계산 로직, 하이브리드 검색
- `ax-frontend` — Next.js 16 프론트엔드 (기존 활용, API 연동만 변경)

---

## 2. 시스템 아키텍처

### 2.1 전체 구조

```
┌─────────────────┐        ┌─────────────────────────────────────────────┐
│   ax-frontend   │  REST  │          FastAPI Backend (Python)            │
│   (Next.js 16)  │───────▶│                                             │
│                 │◀───────│  ┌────────┐  ┌───────────┐  ┌────────────┐ │
│                 │  SSE   │  │ Rules  │  │ Retrieval │  │ Generation │ │
│                 │        │  │ Engine │  │  Engine   │  │  Engine    │ │
└─────────────────┘        │  └────────┘  └───────────┘  └────────────┘ │
                           │                    │               │       │
                           │         ┌──────────┴───┐    ┌──────┴─────┐ │
                           │         │ PostgreSQL   │    │  Gemma3    │ │
                           │         │ + pgvector   │    │  (외부)    │ │
                           │         └──────────────┘    └────────────┘ │
                           └─────────────────────────────────────────────┘
```

### 2.2 기초노동질서 자율점검 파이프라인

```
문서 업로드 (근로계약서 / 임금명세서)
    │
    ▼
OCR 구조화 추출 → 사용자 검수/수정
    │
    ▼
┌────────┐
│구조화   │
│모듈     │
└───┬────┘
    ├──────────────────────┐
    ▼                      ▼
┌──────────┐        ┌──────────┐
│Rules     │        │Retrieval │
│Engine    │        │Engine    │
│          │        │          │
│· 통상시급│        │· 관련 법령│
│· 최저임금│        │· 계약유형별 조문│
│· 분류    │        │· 특약 관련 근거│
│· 리스크  │        │          │
└────┬─────┘        └────┬─────┘
     │  규칙 결과 + RAG 결과  │
     └──────────┬────────┘
                ▼
         ┌──────────┐
         │Generation│  · 결과 설명
         │Engine    │  · 근거 조문 반영
         │(Gemma3)  │  · 확인 필요사항 정리
         └────┬─────┘
              ▼
       최종 사용자 리포트

케이스:
  1. 근로계약서만
  2. 임금명세서만
  3. 근로계약서 + 임금명세서 동시 업로드
```

### 2.3 산재위험요소 자율점검 파이프라인

```
현장 사진 업로드
    │
    ▼
이미지 캡셔닝 (프롬프트 엔지니어링)
    │
    ▼
┌────────┐
│구조화   │
│모듈     │
└───┬────┘
    ▼
┌──────────┐
│Retrieval │ ← KOSHA Guide 조회
│Engine    │   (연관도 높은 조치사항 검색)
└────┬─────┘
     ▼
┌──────────┐
│Generation│ ← 위험성 평가 (프롬프트 엔지니어링)
│Engine    │
│(Gemma3)  │
└────┬─────┘
     ▼
최종 사용자 아웃풋
```

### 2.4 비동기 분석 시퀀스

```
사용자 → Frontend: 파일 업로드
Frontend → Backend: POST /api/{service}/upload
Backend → Frontend: 202 Accepted + { job_id }

Frontend → Backend: SSE 구독 POST /api/{service}/subscribe/{job_id}

[분석 단계별 SSE 업데이트 (3-5분 소요)]
  PENDING           →  작업 대기 중
  OCR_PROCESSING    →  구조화/캡셔닝 (30%)
  VECTOR_SEARCH     →  법령/KOSHA 검색 (60%)
  LLM_PROCESSING    →  리포트 생성 (90%)
  COMPLETED         →  분석 완료 (100%)
  ERROR             →  오류 발생

Frontend → Backend: GET /api/{service}/{job_id}/result
Backend → Frontend: 분석 결과 JSON
```

---

## 3. API 명세 (프론트엔드 요청서 기준)

### 3.1 기초노동질서 자율점검

| # | Method | URI | 설명 | Request | Response |
|---|--------|-----|------|---------|----------|
| 1 | `POST` | `/api/labor-contract/upload` | 파일 업로드 | `{ contract_file: string, paystub_file?: string }` | `{ job_id: string }` |
| 2 | `POST` | `/api/labor-contract/subscribe/{job_id}` | SSE 상태 구독 | job_id (path) | `{ status: AnalysisStatus }` |
| 3 | `GET` | `/api/labor-contract/{job_id}/result` | 결과 반환 | job_id (path) | `LaborContractAnalyzeResponse` |
| 4 | `GET` | `/api/labor-contract/masked/{fileId}` | 마스킹 PDF/이미지 반환 | fileId (path) | Binary file |

**AnalysisStatus:**
```
PENDING          # 작업 대기 중
OCR_PROCESSING   # 근로조건 계산중 (30%)
VECTOR_SEARCH    # 법령 정보 검색 중 (60%)
LLM_PROCESSING   # 리포트 생성 중 (90%)
COMPLETED        # 모든 분석 완료 (100%)
ERROR            # 오류 발생
```

**LaborContractAnalyzeResponse:**
```typescript
{
  summary?: string;
  sections: Array<{
    title: string;
    status: "준수" | "미준수" | "주의";
    details: Array<{
      label: string;
      value: string;
      recommendation?: string;
    }>;
    legal_basis: {
      laws: Array<{
        law_name: string;
        article: string;
        content: string;
      }>;
      criteria: Array<{
        label: string;
        value: string;
      }>;
    };
  }>;
  masked_files?: Array<{
    file_id: string;
    file_name: string;
    file_type: "pdf" | "image";
  }>;
}
```

### 3.2 산재위험요소 자율점검

| # | Method | URI | 설명 | Request | Response |
|---|--------|-----|------|---------|----------|
| 1 | `POST` | `/api/hazard-analysis/upload` | 이미지 업로드 | FormData (images) | `{ jobId: string }` |
| 2 | `POST` | `/api/hazard-analysis/subscribe/{job_id}` | SSE 상태 구독 | job_id (path) | `{ status: AnalysisStatus }` |
| 3 | `GET` | `/api/hazard-analysis/{job_id}/result` | 결과 반환 | job_id (path) | `HazardAnalysisResponse[]` |
| 4 | `GET` | `/api/hazard-analysis/{fileId}` | 이미지 미리보기 반환 | fileId (path) | Binary file |

**HazardAnalysisResponse:**
```typescript
Array<{
  image_index: number;
  thumbnail_url: string;
  risk_counts: {
    very_high: number;
    high: number;
    caution: number;
    good: number;
  };
  risks: Array<{
    minor_category_from_image: string;
    risk_level: "매우 위험" | "위험" | "주의" | "양호";
    action_item: string;    // KOSHA 가이드 조치사항
    action_code: string;    // KOSHA GUIDE 코드
  }>;
}>
```

---

## 4. 기술 스택

| 구분 | 기술 | 비고 |
|------|------|------|
| Framework | **FastAPI** | 비동기, SSE 지원 |
| Language | **Python 3.11+** | |
| Vector DB | **PostgreSQL + pgvector** | 하이브리드 검색 (벡터 + 키워드) |
| ORM | **SQLAlchemy 2.0** | async 지원 |
| Embedding | **Google Gemini** (gemini-embedding-001) | 벡터 검색용 |
| LLM (생성) | **Gemma3 27B** | `http://211.233.14.18:8000/` (자체 호스팅) |
| OCR/PDF | **PyMuPDF (fitz)** | PDF 파싱, 마스킹 |
| SSE | **sse-starlette** | 진행률 스트리밍 |
| Container | **Docker + Docker Compose** | |
| 테스트 | **pytest + pytest-asyncio** | |
| Frontend | **ax-frontend (Next.js 16)** | 기존 프론트 활용 |

**LLM 역할 분리:**
- **Gemini Embedding** — 법령/KOSHA 문서 벡터화 (Retrieval Engine)
- **Gemma3 27B** — 답변/리포트 생성 (Generation Engine), 이미지 캡셔닝

---

## 5. 디렉토리 구조

```
hello-rag-system/
├── docker-compose.yml
├── .env.example
├── PLAN.md
│
├── retrieval_engine/              # 검색 엔진 (Port 8000)
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── app/
│   │   ├── api/routes/
│   │   │   ├── retrieval.py       # POST /retrieval/search
│   │   │   └── health.py
│   │   ├── services/
│   │   │   ├── embedding_service.py
│   │   │   ├── retrieval_service.py
│   │   │   └── citation_service.py
│   │   ├── repositories/
│   │   │   └── vector_repository.py
│   │   ├── models/schemas.py
│   │   └── config.py
│   └── data/
│       ├── labor_law.csv
│       ├── minimum_wage_history.csv
│       └── kosha_guide.csv
│
├── generation_engine/             # 생성 엔진 (Port 8001)
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── app/
│   │   ├── api/routes/
│   │   │   ├── generation.py      # POST /generation/answer
│   │   │   └── health.py
│   │   ├── services/
│   │   │   ├── generation_service.py
│   │   │   ├── llm_service.py     # Gemma3 API 호출
│   │   │   └── prompt_service.py
│   │   ├── clients/
│   │   │   └── retrieval_client.py
│   │   ├── models/schemas.py
│   │   └── config.py
│   └── prompts/
│       ├── labor_contract.py
│       └── hazard_assessment.py
│
├── rules_engine/                  # 규칙 엔진 (Port 8002)
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── app/
│   │   ├── api/routes/
│   │   │   ├── analyze.py         # POST /analyze
│   │   │   └── health.py
│   │   ├── engine/
│   │   │   ├── labor_rules.py
│   │   │   └── wage_calculator.py
│   │   ├── data/
│   │   │   └── minimum_wage_table.py
│   │   ├── models/schemas.py
│   │   └── config.py
│
├── gateway/                       # API Gateway (Port 8080)
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── app/
│   │   ├── api/routes/
│   │   │   ├── labor_contract.py  # /api/labor-contract/*
│   │   │   ├── hazard_analysis.py # /api/hazard-analysis/*
│   │   │   └── health.py
│   │   ├── services/
│   │   │   ├── orchestrator.py    # 파이프라인 오케스트레이션
│   │   │   ├── ocr_service.py     # PDF → 구조화 데이터
│   │   │   ├── image_caption.py   # 이미지 캡셔닝 (Gemma3)
│   │   │   └── pdf_masking.py     # PII 마스킹
│   │   ├── clients/
│   │   │   ├── rules_client.py
│   │   │   ├── retrieval_client.py
│   │   │   └── generation_client.py
│   │   ├── models/schemas.py
│   │   └── config.py
│
├── db/
│   └── init/
│       ├── 01_extensions.sql      # pgvector 확장
│       ├── 02_schema.sql          # 테이블 생성
│       └── 03_seed.sql            # 초기 데이터
│
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

---

## 6. 데이터 모델

### 6.1 Vector Store

```sql
-- 법령 데이터 (근로기준법, 최저임금법)
CREATE TABLE law_documents (
    id SERIAL PRIMARY KEY,
    law_name VARCHAR(200),
    article_number VARCHAR(50),
    article_title VARCHAR(200),
    content TEXT,
    embedding vector(768),
    category VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

-- KOSHA 가이드 (산재위험)
CREATE TABLE kosha_guides (
    id SERIAL PRIMARY KEY,
    guide_code VARCHAR(50),
    title VARCHAR(200),
    content TEXT,
    risk_category VARCHAR(100),
    embedding vector(768),
    created_at TIMESTAMP DEFAULT NOW()
);

-- 분석 작업 상태
CREATE TABLE analysis_jobs (
    id UUID PRIMARY KEY,
    service_type VARCHAR(50),        -- labor-contract | hazard-analysis
    status VARCHAR(30) DEFAULT 'PENDING',
    progress INTEGER DEFAULT 0,
    result JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### 6.2 핵심 스키마 (Python)

```python
class ContractInfo(BaseModel):
    contract_type: str           # 정규직, 계약직 등
    wage_type: str               # 월급, 일급, 시급
    base_salary: int
    work_hours_per_week: float
    work_days_per_week: int
    contract_start: date
    contract_end: date | None
    allowances: dict[str, int]

class PayslipInfo(BaseModel):
    total_payment: int
    base_salary: int
    overtime_pay: int
    night_pay: int
    holiday_pay: int
    deductions: dict[str, int]
```

---

## 7. 프론트엔드 수정 사항 (ax-frontend)

### 7.1 결과 화면 구조

분석 결과는 아래와 같은 구조로 렌더링되어야 한다:

```
┌──────────────────────────────────────────────────────────┐
│ 업로드된 파일                                             │
│  [파일 2025_JH제약 근로계약서_홍길동.jpg]                  │
│  [파일 2025_JH제약 근로계약서_홍길동.pdf]  ← 마스킹 PDF    │
├──────────────────────────────────────────────────────────┤
│ 결과 요약                                                │
│  · 전반적으로 법적 기준을 준수하고 있으나, 제 5조          │
│    (연장근로) 조항은 향후 분쟁 소지를 줄이기 위해          │
│    수정을 고려해 보시는 것이 좋습니다.                     │
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│ 1. 최저임금 준수 여부                       [준수]        │
│  · 계약상 급여 : 월 2,900,000원 (기본급 + 식대)           │
│  · 월 소정근로 : 209시간 (주 40시간 기준)                 │
│  · 검토 결과 : 법적 최저임금 이상, 정상 확인              │
│                                       [법적근거 보기 →]  │
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│ 2. 주 52시간제 (연장근로)                   [주의]        │
│  · 기본 근무 시간 : 09:00~18:00 / 주 5일                 │
│  · 검토 대상 조항 : 제 5조 (연장근로)                     │
│  · 검토 결과 : 포괄적 동의로 해석될 여지                  │
│  · 권고 : 개별 합의 원칙 명시 권장                        │
│                                       [법적근거 보기 →]  │
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│ 3. 기타 검토 의견                                        │
│  · 기본 기재사항 포함 확인                                │
│  · 주휴수당 계산 관련 유의사항                             │
├──────────────────────────────────────────────────────────┤
│ ※ 본 결과는 AI 분석에 기반한 참고용 자료이며,             │
│   법적 효력이 없습니다. 정확한 상담은 국번 없이            │
│   1350을 이용하세요.                                     │
└──────────────────────────────────────────────────────────┘
```

### 7.2 Response → UI 매핑

| Response 필드 | UI 영역 | 설명 |
|--------------|---------|------|
| `masked_files[]` | 상단 "업로드된 파일" | 원본 이미지 + 마스킹 PDF 파일 카드 |
| `summary` | "결과 요약" 섹션 | 전체 분석 요약 1-2문장 |
| `sections[].title` | 섹션 제목 | "1. 최저임금 준수 여부" 등 |
| `sections[].status` | 섹션 우측 배지 | 준수(녹색) / 미준수(적색) / 주의(황색) |
| `sections[].details[].label` | 항목명 | "계약상 급여", "월 소정근로" 등 |
| `sections[].details[].value` | 항목값 | 구체적 수치/내용 |
| `sections[].details[].recommendation` | 권고 사항 | 선택적 표시, "권고:" 접두사 |
| `sections[].legal_basis` | "법적근거 보기" 팝업 | 클릭 시 관련 법령 조항 표시 |
| (하단 고정) | 면책 고지 | 고정 문구 |

### 7.3 수정 대상 파일 및 변경 내용

#### API 연동 변경

| 파일 | 현재 | 변경 |
|------|------|------|
| `src/app/api/labor-contract/analyze/route.ts` | Gemma3 직접 호출 (base64 이미지 → LLM) | FastAPI Gateway 프록시로 변경: `POST ${GATEWAY_URL}/api/labor-contract/upload` |
| `src/app/api/analyze/route.ts` | Gemma3 직접 호출 (산재 이미지 분석) | FastAPI Gateway 프록시로 변경: `POST ${GATEWAY_URL}/api/hazard-analysis/upload` |

#### SSE 구독 추가

| 파일 | 변경 내용 |
|------|-----------|
| `src/features/labor-contract/segments/` | SSE 연결 훅 추가: `/api/labor-contract/subscribe/{job_id}` 구독 → status에 따라 로딩 단계 표시 |
| `src/features/hazard-assessment/segments/analyze/` | SSE 연결 훅 추가: `/api/hazard-analysis/subscribe/{job_id}` 구독 |
| `src/shared/lib/hooks/` | `useSSESubscription.ts` 공통 훅 신규 생성 |

#### 결과 화면 조정

| 파일 | 변경 내용 |
|------|-----------|
| `src/features/labor-contract/segments/summary/` | `LaborContractAnalyzeResponse` 스키마에 맞춰 렌더링 로직 수정 — sections 배열 순회, status 배지, legal_basis 팝업 |
| `src/features/labor-contract/segments/masking/` | `masked_files[]`로 마스킹 PDF 미리보기 연동: `GET /api/labor-contract/masked/{fileId}` |
| `src/shared/ui/` | 상태 배지 컴포넌트 (준수/미준수/주의) 필요 시 추가 |
| `src/shared/lib/llm-response-parser/` | 기존 마크다운 파서 → 구조화된 JSON 파서로 변경 (더 이상 LLM 마크다운 직접 파싱 불필요) |

#### 환경 변수 추가

```env
# .env.local (ax-frontend)
GATEWAY_API_URL=http://localhost:8080   # FastAPI Gateway
# GEMMA3_API_URL 은 더 이상 프론트에서 직접 사용하지 않음 (백엔드에서 호출)
```

---

## 8. 참고 시스템 재사용 전략

### basic-labor-standards-self-Inspection에서 이식

| 항목 | 소스 경로 | 활용 |
|------|-----------|------|
| 하이브리드 검색 | `retrieval_engine/app/services/` | 그대로 이식 |
| 임금 계산 규칙 | `rules_engine/app/engine/labor_rules.py` | 그대로 이식 |
| 최저임금 테이블 | `rules_engine/app/data/minimum_wage_table.py` | 그대로 이식 |
| 법령 CSV | `retrieval_engine/data/` | 그대로 사용 |
| DB 초기화 | `retrieval_engine/init/` | 확장 사용 |

### ax-frontend 변경 요약

| 영역 | 변경 |
|------|------|
| API 호출 | Gemma3 직접 호출 → FastAPI Gateway 호출 |
| 응답 파싱 | 마크다운 파싱 → 구조화 JSON 직접 사용 |
| SSE | 신규 추가 (진행률 구독) |
| 결과 렌더링 | sections 배열 기반 렌더링 + 법적근거 팝업 |

---

## 9. 구현 단계

### Phase 1: 인프라 셋업 (3일)
- [ ] Docker Compose (PostgreSQL + pgvector + 4개 FastAPI 서비스)
- [ ] DB 스키마 생성, 법령/KOSHA 데이터 인제스트
- [ ] 기본 FastAPI 프로젝트 구조 + 헬스체크
- [ ] `.env` 환경변수 설정

### Phase 2: Retrieval Engine (4일)
- [ ] Gemini 임베딩 서비스
- [ ] pgvector 하이브리드 검색 (벡터 + 키워드)
- [ ] 법조문 인용 매핑
- [ ] 근로기준법/최저임금법/KOSHA 데이터 적재
- [ ] 검색 API 테스트

### Phase 3: Rules Engine (4일)
- [ ] 임금 계산 (통상시급, 최저임금 비교)
- [ ] 근로시간 규칙 (주 52시간, 휴게시간)
- [ ] 포괄임금 감지, 리스크 탐지
- [ ] 단위 테스트 (80%+ 커버리지)

### Phase 4: Generation Engine (4일)
- [ ] Gemma3 LLM 서비스 (`http://211.233.14.18:8000/v1/chat/completions`)
- [ ] 근로계약서 분석 프롬프트 엔지니어링
- [ ] 산재위험 평가 프롬프트 엔지니어링
- [ ] Rules 결과 + Retrieval 컨텍스트 → 구조화 JSON 생성

### Phase 5: Gateway + 오케스트레이션 (5일)
- [ ] `/api/labor-contract/*` 라우팅 (upload, subscribe, result, masked)
- [ ] `/api/hazard-analysis/*` 라우팅 (upload, subscribe, result, file)
- [ ] OCR 서비스 (PDF → 구조화 데이터)
- [ ] 이미지 캡셔닝 서비스 (Gemma3)
- [ ] 비동기 파이프라인 + SSE 진행률 스트리밍
- [ ] PDF 마스킹 처리
- [ ] Job 상태 관리 (PENDING → COMPLETED)

### Phase 6: 프론트엔드 연동 (3일)
- [ ] ax-frontend API URL → FastAPI Gateway로 변경
- [ ] `useSSESubscription` 공통 훅 생성
- [ ] 결과 화면 sections 기반 렌더링 수정
- [ ] 법적근거 팝업 연동
- [ ] 에러 핸들링

### Phase 7: 테스트 및 최적화 (3일)
- [ ] 통합 테스트 (전체 파이프라인)
- [ ] E2E 테스트 (Playwright)
- [ ] 검색 정확도 튜닝
- [ ] 보안 검토

---

## 10. 환경 변수

```env
# Database
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=hello_rag
POSTGRES_USER=rag_user
POSTGRES_PASSWORD=

# Gemini API (임베딩 전용)
GOOGLE_API_KEY=

# Gemma3 API (답변 생성용)
GEMMA3_API_URL=http://211.233.14.18:8000

# Service Ports
RETRIEVAL_ENGINE_PORT=8000
GENERATION_ENGINE_PORT=8001
RULES_ENGINE_PORT=8002
GATEWAY_PORT=8080

# Service URLs (Docker internal)
RETRIEVAL_ENGINE_URL=http://retrieval-engine:8000
GENERATION_ENGINE_URL=http://generation-engine:8001
RULES_ENGINE_URL=http://rules-engine:8002

# Features
AUTO_INGEST_ON_STARTUP=true
```
