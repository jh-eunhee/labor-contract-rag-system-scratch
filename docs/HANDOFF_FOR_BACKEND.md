# HANDOFF: 백엔드 담당

> Gateway, 오케스트레이션, SSE, OCR, PDF 마스킹 구현 담당자를 위한 문서

---

## 담당 범위

```
┌──────────────────────────────────────────────────────────────┐
│                        백엔드 영역                            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                    Gateway (:8080)                      │  │
│  │                                                        │  │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌────────┐  │  │
│  │  │ API      │  │Orchestr- │  │ SSE    │  │ File   │  │  │
│  │  │ Routes   │  │ator      │  │Manager │  │Storage │  │  │
│  │  └──────────┘  └──────────┘  └────────┘  └────────┘  │  │
│  │                                                        │  │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────────────┐   │  │
│  │  │ OCR      │  │ Image    │  │ PDF Masking        │   │  │
│  │  │ Service  │  │ Caption  │  │ Service            │   │  │
│  │  └──────────┘  └──────────┘  └────────────────────┘   │  │
│  └────────────────────────────────────────────────────────┘  │
│         │               │               │                    │
│         ▼               ▼               ▼                    │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │  Rules     │  │ Retrieval  │  │ Generation │  ← AI 담당  │
│  │  Engine    │  │ Engine     │  │ Engine     │              │
│  │  :8002     │  │ :8000      │  │ :8001      │              │
│  └────────────┘  └────────────┘  └────────────┘             │
└──────────────────────────────────────────────────────────────┘
```

**내가 만드는 것:** Gateway API, 오케스트레이션, SSE, OCR, 캡셔닝, 마스킹
**내가 만들지 않는 것:** 3개 AI 엔진 내부 로직 (→ AI 담당)
**내가 호출하는 것:** AI 엔진 API + Gemma3 API (캡셔닝용)

---

## 1. 프론트엔드에 제공하는 API

### 1.1 기초노동질서 자율점검

```
POST /api/labor-contract/upload           파일 업로드 → job_id 반환
POST /api/labor-contract/subscribe/{id}   SSE 진행률 구독
GET  /api/labor-contract/{id}/result      분석 결과 조회
GET  /api/labor-contract/masked/{fileId}  마스킹 파일 다운로드
```

### 1.2 산재위험요소 자율점검

```
POST /api/hazard-analysis/upload          이미지 업로드 → jobId 반환
POST /api/hazard-analysis/subscribe/{id}  SSE 진행률 구독
GET  /api/hazard-analysis/{id}/result     분석 결과 조회
GET  /api/hazard-analysis/{fileId}        이미지 다운로드
```

---

## 2. 근로계약서 파이프라인 상세

### 전체 시퀀스

```
프론트                Gateway                     AI 엔진들
  │                     │                            │
  │  POST /upload       │                            │
  │  (파일 전송)        │                            │
  ├────────────────────▶│                            │
  │                     │  Job 생성 (UUID)           │
  │  { job_id }         │  DB INSERT                 │
  │◀────────────────────│  (status: PENDING)         │
  │                     │                            │
  │  POST /subscribe    │                            │
  │  (SSE 연결)         │                            │
  ├────────────────────▶│                            │
  │                     │                            │
  │  ┌──────── 비동기 파이프라인 시작 ────────────┐  │
  │  │                  │                         │  │
  │  │ ① OCR 구조화     │                         │  │
  │  SSE: OCR_PROCESSING│  PDF → 텍스트 추출      │  │
  │◀─┤ (30%)            │  텍스트 → ContractInfo   │  │
  │  │                  │  텍스트 → PayslipInfo    │  │
  │  │                  │  (급여명세서 있을 때)     │  │
  │  │                  │                         │  │
  │  │ ② Rules 분석     │  POST :8002/analyze     │  │
  │  │                  ├────────────────────────▶│  │
  │  │                  │  RulesResult            │  │
  │  │                  │◀────────────────────────┤  │
  │  │                  │  (sections + comparison) │  │
  │  │                  │                         │  │
  │  │ ③ 법령 검색      │                         │  │
  │  SSE: VECTOR_SEARCH │  (Generation Engine이    │  │
  │◀─┤ (60%)            │   내부에서 Retrieval     │  │
  │  │                  │   Engine 호출)           │  │
  │  │                  │                         │  │
  │  │ ④ 리포트 생성    │  POST :8001/generation/ │  │
  │  SSE: LLM_PROCESSING│       labor-report      │  │
  │◀─┤ (90%)            ├────────────────────────▶│  │
  │  │                  │  LaborContract          │  │
  │  │                  │  AnalyzeResponse        │  │
  │  │                  │◀────────────────────────┤  │
  │  │                  │                         │  │
  │  │ ⑤ 마스킹        │                         │  │
  │  │                  │  PDF/이미지 PII 마스킹   │  │
  │  │                  │  masked_files 생성       │  │
  │  │                  │                         │  │
  │  │ ⑥ 완료          │                         │  │
  │  SSE: COMPLETED     │  result + masked_files  │  │
  │◀─┤ (100%)           │  DB UPDATE              │  │
  │  └──────────────────│─────────────────────────┘  │
  │                     │                            │
  │  GET /{id}/result   │                            │
  ├────────────────────▶│                            │
  │  LaborContract      │  DB에서 result 조회        │
  │  AnalyzeResponse    │                            │
  │◀────────────────────│                            │
  │                     │                            │
```

### 각 단계별 입출력

#### ① OCR 구조화 (내가 구현)

```
INPUT:  contract_file (PDF/이미지), paystub_file? (PDF/이미지)
PROCESS:
  1. PyMuPDF(fitz)로 텍스트 추출
  2. 텍스트 없으면 → 이미지 변환 → Gemma3 비전 캡셔닝
  3. 추출 텍스트 → 정규표현식 + LLM 보조로 구조화
OUTPUT: ContractInfo, PayslipInfo?
```

```python
# ContractInfo (→ Rules Engine에 전달)
{
  "contract_type": "정규직",
  "wage_type": "월급",
  "base_salary": 2600000,
  "work_hours_per_week": 40,
  "work_days_per_week": 5,
  "contract_start": "2025-03-01",
  "contract_end": null,
  "allowances": { "식대": 300000 },
  "special_clauses": ["연장근로 포괄 동의 조항..."]
}

# PayslipInfo (선택, → Rules Engine에 전달)
{
  "total_payment": 2700000,
  "base_salary": 2600000,
  "overtime_pay": 0,
  "night_pay": 0,
  "holiday_pay": 0,
  "deductions": { "국민연금": 117000 },
  "allowances": { "식대": 200000 }
}
```

#### ② Rules Engine 호출 (AI가 구현, 내가 호출)

```
내가 보내는 것:
  POST http://rules-engine:8002/analyze
  BODY: { "contract": ContractInfo, "payslip": PayslipInfo?, "assumptions": {} }

내가 받는 것:
  RulesResult {
    "sections": [{ title, status, details }],
    "risk_flags": ["포괄임금 의심"],
    "comparison": {                    ← payslip 있을 때만
      "has_payslip": true,
      "mismatches": [{ item, contract_value, payslip_value, difference, severity }],
      "matches": [{ item, value }],
      "overall_status": "미준수"
    }
  }
```

#### ④ Generation Engine 호출 (AI가 구현, 내가 호출)

```
내가 보내는 것:
  POST http://generation-engine:8001/generation/labor-report
  BODY: {
    "rules_result": RulesResult,       ← ②에서 받은 것 그대로
    "contract_info": ContractInfo      ← ①에서 만든 것
  }

내가 받는 것:
  LaborContractAnalyzeResponse {
    "summary": "...",
    "sections": [{
      "title": "1. 최저임금 준수 여부",
      "status": "준수",
      "details": [{ label, value, recommendation? }],
      "legal_basis": {
        "laws": [{ law_name, article, content }],
        "criteria": [{ label, value }]
      }
    }]
  }
```

#### ⑤ 마스킹 + 최종 조합 (내가 구현)

```
내가 하는 것:
  1. 원본 PDF/이미지에서 PII 마스킹 (이름, 주민번호, 전화번호, 주소)
  2. 마스킹 파일 저장 → file_id 발급
  3. Generation 결과에 masked_files 추가

최종 결과 (DB에 저장, 프론트에 반환):
{
  "summary": "...",
  "sections": [...],                   ← Generation Engine 결과
  "masked_files": [                    ← 내가 추가
    { "file_id": "abc123", "file_name": "contract_masked.pdf", "file_type": "pdf" }
  ]
}
```

---

## 3. 산재위험 파이프라인 상세

### 전체 시퀀스

```
프론트                Gateway                     AI 엔진들
  │                     │                            │
  │  POST /upload       │                            │
  │  (이미지들)         │                            │
  ├────────────────────▶│                            │
  │  { jobId }          │  Job 생성                  │
  │◀────────────────────│                            │
  │                     │                            │
  │  POST /subscribe    │                            │
  ├────────────────────▶│                            │
  │                     │                            │
  │  ┌──────── 비동기 파이프라인 ─────────────────┐  │
  │  │                  │                         │  │
  │  │ ① 이미지 캡셔닝  │                         │  │
  │  SSE: OCR_PROCESSING│  각 이미지 → Gemma3     │  │
  │◀─┤ (30%)            │  비전 API 호출          │  │
  │  │                  │  → 위험요소 설명 텍스트  │  │
  │  │                  │                         │  │
  │  │ ② KOSHA 검색     │                         │  │
  │  SSE: VECTOR_SEARCH │  (Generation Engine이    │  │
  │◀─┤ (60%)            │   Retrieval 내부 호출)   │  │
  │  │                  │                         │  │
  │  │ ③ 리포트 생성    │  POST :8001/generation/ │  │
  │  SSE: LLM_PROCESSING│       hazard-report     │  │
  │◀─┤ (90%)            ├────────────────────────▶│  │
  │  │                  │  HazardAnalysis         │  │
  │  │                  │  Response[]             │  │
  │  │                  │◀────────────────────────┤  │
  │  │                  │                         │  │
  │  │ ④ 완료          │  thumbnail_url 추가     │  │
  │  SSE: COMPLETED     │  DB UPDATE              │  │
  │◀─┤ (100%)           │                         │  │
  │  └──────────────────│─────────────────────────┘  │
  │                     │                            │
  │  GET /{id}/result   │                            │
  ├────────────────────▶│                            │
  │  HazardAnalysis     │                            │
  │  Response[]         │                            │
  │◀────────────────────│                            │
```

### 이미지 캡셔닝 (내가 구현)

```
INPUT:  이미지 바이너리 (png, jpg)
PROCESS:
  1. base64 인코딩
  2. Gemma3 비전 API 호출 (http://211.233.14.18:8000/v1/chat/completions)
  3. 프롬프트: "산업안전보건 관점에서 이 현장 사진의 위험 요소를 식별하라"
OUTPUT: 텍스트 설명 (위험요소 나열)

→ Generation Engine에 { captions: [{ image_index, description }] } 로 전달
```

### Generation Engine 호출 결과에 thumbnail_url 추가

```
Generation Engine이 반환:
[
  {
    "image_index": 1,
    "risk_counts": { ... },
    "risks": [{ minor_category_from_image, risk_level, action_item, action_code }]
  }
]

내가 thumbnail_url 추가:
[
  {
    "image_index": 1,
    "thumbnail_url": "/api/hazard-analysis/file-abc123",   ← 내가 추가
    "risk_counts": { ... },
    "risks": [...]
  }
]
```

---

## 4. SSE 이벤트 명세

프론트가 `POST /api/{service}/subscribe/{job_id}`로 연결하면 아래 이벤트를 전송:

```
event: status
data: {"status": "PENDING"}

event: status
data: {"status": "OCR_PROCESSING"}     ← 30%

event: status
data: {"status": "VECTOR_SEARCH"}      ← 60%

event: status
data: {"status": "LLM_PROCESSING"}     ← 90%

event: status
data: {"status": "COMPLETED"}          ← 100%, 프론트가 result 조회

# 에러 시:
event: status
data: {"status": "ERROR", "message": "파일 형식을 인식할 수 없습니다"}
```

---

## 5. Job 상태 관리

```sql
-- analysis_jobs 테이블
CREATE TABLE analysis_jobs (
    id UUID PRIMARY KEY,
    service_type VARCHAR(50),          -- 'labor-contract' | 'hazard-analysis'
    status VARCHAR(30) DEFAULT 'PENDING',
    progress INTEGER DEFAULT 0,
    result JSONB,                      -- 최종 결과 저장
    error_message TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

상태 전이:

```
PENDING → OCR_PROCESSING → VECTOR_SEARCH → LLM_PROCESSING → COMPLETED
                                                            ↘ ERROR
```

---

## 6. 파일 저장 구조

```
uploads/
  {job_id}/
    contract.pdf          원본 근로계약서
    paystub.pdf           원본 급여명세서 (선택)
    image_1.jpg           원본 현장 사진
    image_2.jpg

masked/
  {file_id}.pdf           마스킹된 PDF
  {file_id}.png           마스킹된 이미지

thumbnails/
  {file_id}.jpg           썸네일 이미지
```

---

## 7. AI 담당자에게 요청할 것

| 요청 | 상세 |
|------|------|
| Rules Engine API 스키마 확정 | ContractInfo/PayslipInfo 필드명, comparison 응답 구조 |
| Generation Engine API 스키마 확정 | labor-report / hazard-report 요청/응답 확정 |
| Retrieval Engine 헬스체크 | 서비스 기동 확인용 `/health` 엔드포인트 |
| 에러 응답 형식 통일 | `{ "error": string, "detail": string }` 형식 합의 |

---

## 8. 프론트 담당자에게 제공할 것

| 제공 항목 | 상세 |
|-----------|------|
| API 엔드포인트 4+4개 | 위 섹션 1 참조 |
| SSE 이벤트 형식 | 위 섹션 4 참조 |
| 응답 JSON 스키마 | `LaborContractAnalyzeResponse`, `HazardAnalysisResponse[]` |
| 파일 업로드 제한 | 최대 10MB, pdf/png/jpg 허용 |
| CORS 설정 | 프론트 origin 허용 필요 → 프론트에서 알려줄 것 |
