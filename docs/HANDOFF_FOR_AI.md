# HANDOFF: AI 솔루션 담당

> Retrieval Engine, Rules Engine, Generation Engine 구현 담당자를 위한 문서

---

## 담당 범위

```
┌────────────────────────────────────────────────────┐
│                  AI 솔루션 영역                       │
│                                                    │
│  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │  Retrieval  │  │   Rules    │  │  Generation  │ │
│  │  Engine     │  │   Engine   │  │  Engine      │ │
│  │  :8000      │  │   :8002    │  │  :8001       │ │
│  └──────┬─────┘  └──────┬─────┘  └──────┬───────┘  │
│         │               │               │          │
│    pgvector          임금 계산       Gemma3 LLM      │
│    법령/KOSHA        규칙 판정       리포트 생성         │
│                                                    │
└────────────────────────────────────────────────────┘
         ▲               ▲               ▲
         │               │               │
    Gateway(백엔드)가 호출하는 내부 API들
```

**내가 만드는 것:** 3개 내부 엔진의 API + 비즈니스 로직
**내가 만들지 않는 것:** Gateway, SSE, 파일 업로드, OCR, PDF 마스킹 (→ 백엔드 담당)

---

## 1. Retrieval Engine (:8000)

### 역할

법령/KOSHA 데이터를 pgvector에 적재하고, 쿼리에 대해 관련 법령을 검색하여 반환한다.

### 프로세스

```
                         Retrieval Engine
                    ┌─────────────────────────┐
                    │                         │
  쿼리 텍스트 ─────▶.  │  Gemini Embedding       │
                    │       │                 │
                    │       ▼                 │
                    │  ┌──────────┐           │
                    │  │ pgvector │           │
                    │  │ 벡터검색  │            │
                    │  └────┬─────┘           │
                    │       │                 │
                    │       ▼                 │
                    │  키워드 검색 병합           │
                    │       │                 │
                    │       ▼                 │
                    │  인용 매핑                │
                    │  (조항번호 부여)           │
                    │       │                 │
  검색 결과 ◀───────. │       ▼                 │
                    │  결과 반환                │
                    └─────────────────────────┘
```

### API 명세

#### `POST /retrieval/search`

단일 쿼리 검색

```
REQUEST:
{
  "query": "최저임금 위반",
  "top_k": 5,
  "search_type": "labor"          ← "labor" | "kosha"
}

RESPONSE:
{
  "results": [
    {
      "content": "사용자는 최저임금액 이상의 임금을 지급하여야 한다.",
      "score": 0.92,
      "citation": {
        "law_name": "최저임금법",
        "article_number": "제6조",
        "article_title": "최저임금의 효력"
      }
    },
    ...
  ]
}
```

#### `POST /retrieval/context-for-report`

다중 토픽 검색 (Generation Engine이 호출)

```
REQUEST:
{
  "topics": ["최저임금 준수 여부", "연장근로 제한", "근로조건 명시"],
  "search_type": "labor"
}

RESPONSE:
{
  "contexts": [
    {
      "topic": "최저임금 준수 여부",
      "results": [
        {
          "content": "...",
          "score": 0.92,
          "citation": { "law_name": "최저임금법", "article_number": "제6조", ... }
        }
      ]
    },
    ...
  ]
}
```

### 데이터

| 테이블 | 용도 | 데이터 소스 |
|--------|------|-------------|
| `law_documents` | 근로기준법, 최저임금법 조항 | `data/labor_law.csv` |
| `kosha_guides` | KOSHA 안전보건 가이드 | `data/kosha_guide.csv` |

참고 코드: `basic-labor-standards-self-Inspection/retrieval_engine/`

---

## 2. Rules Engine (:8002)

### 역할

구조화된 근로계약서/급여명세서 데이터를 받아 규칙 기반으로 준수 여부를 판정한다.
급여명세서가 있으면 근로계약서 항목과 대조하여 불일치를 탐지한다.

### 프로세스

```
                           Rules Engine
                      ┌─────────────────────────┐
                      │                         │
ContractInfo ────────▶│  임금 계산                │
(구조화 데이터)       │    · 통상시급 산출             │
                      │    · 최저임금 비교         │
                      │         │               │
                      │         ▼               │
                      │  근로시간 규칙 판정         │
                      │    · 주 52시간           │
                      │    · 휴게시간             │
                      │         │               │
                      │         ▼               │
                      │  리스크 탐지               │
                      │    · 포괄임금             │
                      │    · 3.3% 프리랜서        │
                      │         │               │
PayslipInfo ─────────▶│        ▼               │
(선택)                │  계약-급여 대조            │
                      │    · 기본급 일치?        │
                      │    · 수당 항목 일치?     │
                      │    · 총 지급액 일치?     │
                      │        │              │
RulesResult ◀─────────│        ▼               │
                      │  결과 반환                │
                      └─────────────────────────┘
```

### API 명세

#### `POST /analyze`

```
REQUEST:
{
  "contract": {                          ← 필수 (또는 payslip만)
    "contract_type": "정규직",
    "wage_type": "월급",
    "base_salary": 2600000,
    "work_hours_per_week": 40,
    "work_days_per_week": 5,
    "contract_start": "2025-03-01",
    "contract_end": null,
    "allowances": { "식대": 300000 },
    "special_clauses": [
      "업무상 필요한 경우 연장, 야간, 휴일근로 지시에 동의한 것으로 간주"
    ]
  },
  "payslip": {                           ← 선택 (있으면 대조 분석 수행)
    "total_payment": 2700000,
    "base_salary": 2600000,
    "overtime_pay": 0,
    "night_pay": 0,
    "holiday_pay": 0,
    "deductions": { "국민연금": 117000, "건강보험": 91000 },
    "allowances": { "식대": 200000 }
  },
  "assumptions": {
    "weekly_holiday_included": true
  }
}

RESPONSE:
{
  "sections": [
    {
      "title": "최저임금 준수 여부",
      "status": "준수",                  ← "준수" | "미준수" | "주의"
      "details": [
        { "label": "통상시급", "value": "12,440원" },
        { "label": "법정 최저임금", "value": "10,030원 (2025년)" },
        { "label": "판정", "value": "최저임금 이상" }
      ]
    },
    {
      "title": "주 52시간제",
      "status": "주의",
      "details": [
        { "label": "소정근로시간", "value": "주 40시간" },
        { "label": "연장근로 조항", "value": "포괄적 동의 조항 발견" },
        { "label": "판정", "value": "포괄적 동의로 해석될 여지 있음" }
      ]
    }
  ],
  "risk_flags": ["포괄임금 의심"],
  "comparison": {                        ← payslip이 있을 때만 포함
    "has_payslip": true,
    "mismatches": [
      {
        "item": "식대",
        "contract_value": 300000,
        "payslip_value": 200000,
        "difference": 100000,
        "severity": "미준수"
      }
    ],
    "matches": [
      { "item": "기본급", "value": 2600000 }
    ],
    "overall_status": "미준수"
  }
}
```

### 대조 비교 규칙

| 비교 항목 | 계약서 필드 | 명세서 필드 | 판정 기준 |
|-----------|------------|------------|-----------|
| 기본급 | `contract.base_salary` | `payslip.base_salary` | 계약 > 명세 → 미준수 |
| 수당 | `contract.allowances[항목]` | `payslip.allowances[항목]` | 계약 > 명세 → 미준수 |
| 연장수당 | 연장근로 조항 유무 | `payslip.overtime_pay` | 조항 있는데 0원 → 주의 |
| 야간수당 | 야간근로 조항 유무 | `payslip.night_pay` | 조항 있는데 0원 → 주의 |
| 휴일수당 | 휴일근로 조항 유무 | `payslip.holiday_pay` | 조항 있는데 0원 → 주의 |
| 총 지급액 | 기본급+수당 합산 | `payslip.total_payment` | 합산 > 명세 → 미준수 |

참고 코드: `basic-labor-standards-self-Inspection/rules_engine/`

---

## 3. Generation Engine (:8001)

### 역할

Rules Engine 결과 + Retrieval Engine 법령 컨텍스트를 조합하여 Gemma3 LLM으로 최종 리포트를 생성한다.

### 프로세스

```
                      Generation Engine
                ┌────────────────────────────────┐
                │                                │
rules_result ──▶│  토픽 추출                     │
                │  (sections[].title 목록)        │
                │       │                        │
                │       ▼                        │
                │  Retrieval Engine 호출          │
                │  POST /retrieval/              │
                │       context-for-report       │
                │       │                        │
                │       ▼                        │
                │  프롬프트 조합                  │
                │    · rules_result              │
                │    · retrieval_context         │
                │    · payslip_comparison        │
                │      (있을 때만)                │
                │       │                        │
                │       ▼                        │
                │  Gemma3 API 호출               │
                │  POST /v1/chat/completions     │
                │  (http://211.233.14.18:8000)   │
                │       │                        │
                │       ▼                        │
                │  JSON 파싱 + 스키마 검증        │
                │  (실패 시 재시도)                │
                │       │                        │
JSON 리포트 ◀──│       ▼                        │
                │  결과 반환                     │
                └────────────────────────────────┘
```

### API 명세

#### `POST /generation/labor-report`

근로계약서 분석 리포트 생성

```
REQUEST:
{
  "rules_result": {                       ← Rules Engine 응답 전체
    "sections": [...],
    "risk_flags": [...],
    "comparison": { ... }                 ← 급여명세서 있을 때만
  },
  "contract_info": {                      ← 원본 계약 정보
    "contract_type": "정규직",
    "base_salary": 2600000,
    ...
  }
}

RESPONSE (= LaborContractAnalyzeResponse):
{
  "summary": "전반적으로 법적 기준을 준수하고 있으나, ...",
  "sections": [
    {
      "title": "1. 최저임금 준수 여부",
      "status": "준수",
      "details": [
        { "label": "계약상 급여", "value": "월 2,900,000원 (...)" },
        { "label": "월 소정근로", "value": "209시간 (주 40시간 기준)" },
        { "label": "검토 결과", "value": "법적 최저임금 이상, 정상 확인" }
      ],
      "legal_basis": {
        "laws": [
          { "law_name": "최저임금법", "article": "제6조 (최저임금의 효력)", "content": "..." }
        ],
        "criteria": [
          { "label": "적용 연도", "value": "2025년" }
        ]
      }
    },
    // ... 추가 섹션
    // 급여명세서 있으면 "계약-급여 대조 결과" 섹션 추가
  ]
}
```

#### `POST /generation/hazard-report`

산재위험 평가 리포트 생성

```
REQUEST:
{
  "captions": [
    {
      "image_index": 1,
      "description": "바닥에 액체가 고여있고 작업자가 보호장구 없이..."
    }
  ]
}

RESPONSE (= HazardAnalysisResponse[]):
[
  {
    "image_index": 1,
    "risk_counts": { "very_high": 1, "high": 2, "caution": 1, "good": 0 },
    "risks": [
      {
        "minor_category_from_image": "바닥 오염에 의해 작업자가 미끄러질 수 있다",
        "risk_level": "위험",
        "action_item": "바닥에 고인 액체 및 오염물을 즉시 제거하고...",
        "action_code": "KOSHA GUIDE H-xx-xxxx"
      }
    ]
  }
]
```

---

## 4. 엔진 간 데이터 흐름 전체

### 근로계약서 분석 (급여명세서 포함 케이스)

```
Gateway가 호출하는 순서:

①  Gateway → Rules Engine
    INPUT:  ContractInfo + PayslipInfo
    OUTPUT: RulesResult { sections, risk_flags, comparison }

②  Gateway → Generation Engine
    INPUT:  RulesResult + ContractInfo
            │
            └─▶ Generation Engine 내부에서
                Retrieval Engine 호출 ─────▶ 법령 검색 결과
            │
            └─▶ Gemma3 LLM 호출
    OUTPUT: LaborContractAnalyzeResponse { summary, sections, ... }

③  Gateway가 masked_files 추가 후 프론트에 반환
```

### 산재위험 분석

```
①  Gateway → 이미지 캡셔닝 (Gemma3) → captions

②  Gateway → Generation Engine
    INPUT:  captions
            │
            └─▶ Generation Engine 내부에서
                Retrieval Engine 호출 ─────▶ KOSHA Guide 검색
            │
            └─▶ Gemma3 LLM 호출
    OUTPUT: HazardAnalysisResponse[]

③  Gateway가 thumbnail_url 추가 후 프론트에 반환
```

---

## 5. 백엔드 담당자에게 넘기는 것

내(AI)가 만든 3개 엔진 API를 백엔드 담당자가 Gateway에서 호출한다.

| 내가 제공하는 API | 백엔드가 호출하는 시점 |
|------------------|---------------------|
| `POST :8002/analyze` | OCR 구조화 완료 후 |
| `POST :8001/generation/labor-report` | Rules 결과 받은 후 |
| `POST :8001/generation/hazard-report` | 이미지 캡셔닝 완료 후 |
| `POST :8000/retrieval/search` | Generation Engine이 내부 호출 (백엔드 직접 호출 X) |

---

## 6. 프론트 담당자에게 전달할 것

내가 직접 프론트와 통신하지 않는다. Gateway를 통해 아래 형태로 전달된다:

| 최종 응답 | 원본 | 프론트 렌더링 |
|-----------|------|--------------|
| `summary` | Generation Engine 생성 | 결과 요약 영역 |
| `sections[]` | Generation Engine 생성 | 섹션 카드 + 상태 배지 |
| `legal_basis` | Retrieval Engine 검색 → Generation Engine 포함 | 법적근거 팝업 |
| `comparison` | Rules Engine 대조 → Generation Engine이 섹션으로 변환 | "계약-급여 대조" 섹션 |
| `masked_files` | Gateway에서 추가 (AI 담당 아님) | 파일 카드 |
