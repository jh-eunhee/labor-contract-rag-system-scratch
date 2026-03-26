# HANDOFF: 전체 통합 뷰

> AI / 백엔드 / 프론트엔드 3개 파트가 어떻게 데이터를 주고받는지 한눈에 보는 문서.
> 상세 내용은 각 파트별 HANDOFF 문서 참조.

---

## 1. 전체 시스템 구조

```
┌─────────────┐      ┌──────────────┐      ┌──────────────────────────────┐
│  Frontend   │      │   Gateway    │      │         AI Engines           │
│  (Next.js)  │ REST │   (:8080)    │ HTTP │                              │
│             │─────▶│              │─────▶│  Rules    :8002              │
│             │◀─────│  오케스트레이션 │◀─────│  Retrieval :8000             │
│             │ SSE  │  OCR/마스킹   │      │  Generation :8001            │
└─────────────┘      └──────────────┘      └──────────────────────────────┘
                            │                         │
                     ┌──────┴───────┐          ┌──────┴───────┐
                     │  PostgreSQL  │          │   Gemma3     │
                     │  + pgvector  │          │   (외부 LLM) │
                     └──────────────┘          └──────────────┘
```

### 파트별 담당

| 파트 | 담당 | 상세 문서 |
|------|------|-----------|
| **AI** | Retrieval / Rules / Generation 엔진 내부 로직 | `HANDOFF_FOR_AI.md` |
| **백엔드** | Gateway, 오케스트레이션, SSE, OCR, 마스킹 | `HANDOFF_FOR_BACKEND.md` |
| **프론트** | API 호출 변경, SSE 구독, 결과 렌더링 | `HANDOFF_FOR_FRONTEND.md` |

---

## 2. 근로계약서 분석 - Mermaid Sequence

```mermaid
sequenceDiagram
    participant U as 사용자
    participant F as Frontend<br/>(Next.js)
    participant G as Gateway<br/>(:8080)
    participant R as Rules Engine<br/>(:8002)
    participant V as Retrieval Engine<br/>(:8000)
    participant L as Generation Engine<br/>(:8001)
    participant M as Gemma3 LLM

    U->>F: 근로계약서 + 급여명세서(선택) 업로드

    rect rgb(240, 248, 255)
        Note over F,G: Step 1. 파일 업로드
        F->>G: POST /api/labor-contract/upload<br/>(FormData: contract_file, paystub_file?)
        G-->>F: 202 { job_id: "uuid" }
    end

    F->>G: POST /api/labor-contract/subscribe/{job_id}
    Note over F,G: SSE 연결 수립

    rect rgb(255, 250, 240)
        Note over G,M: Step 2. 비동기 파이프라인 (3-5분)

        G->>G: OCR 구조화 추출
        G-->>F: SSE: OCR_PROCESSING (30%)
        Note right of G: PDF → 텍스트 → ContractInfo + PayslipInfo?

        G->>R: POST /analyze<br/>{ contract: ContractInfo, payslip?: PayslipInfo }
        R->>R: 임금 계산 + 규칙 판정 + 계약-급여 대조
        R-->>G: RulesResult<br/>{ sections, risk_flags, comparison? }

        G-->>F: SSE: VECTOR_SEARCH (60%)

        G->>L: POST /generation/labor-report<br/>{ rules_result, contract_info }
        L->>V: POST /retrieval/context-for-report<br/>{ topics: [섹션 타이틀들] }
        V->>V: 하이브리드 검색 (벡터 + 키워드)
        V-->>L: { contexts: [법령 검색 결과] }

        G-->>F: SSE: LLM_PROCESSING (90%)

        L->>M: POST /v1/chat/completions<br/>(rules_result + retrieval_context → JSON 프롬프트)
        M-->>L: 구조화 JSON 응답
        L->>L: JSON 파싱 + 스키마 검증
        L-->>G: LaborContractAnalyzeResponse<br/>{ summary, sections[] }

        G->>G: PDF 마스킹 (PII 제거)
        G->>G: result에 masked_files 추가
        G-->>F: SSE: COMPLETED (100%)
    end

    rect rgb(240, 255, 240)
        Note over F,G: Step 3. 결과 조회
        F->>G: GET /api/labor-contract/{job_id}/result
        G-->>F: LaborContractAnalyzeResponse<br/>{ summary, sections[], masked_files[] }
    end

    F->>U: 결과 화면 렌더링<br/>(요약 + 섹션 카드 + 법적근거 팝업)
```

---

## 3. 산재위험요소 분석 - Mermaid Sequence

```mermaid
sequenceDiagram
    participant U as 사용자
    participant F as Frontend<br/>(Next.js)
    participant G as Gateway<br/>(:8080)
    participant V as Retrieval Engine<br/>(:8000)
    participant L as Generation Engine<br/>(:8001)
    participant M as Gemma3 LLM

    U->>F: 현장 사진 업로드 (1장 이상)

    rect rgb(240, 248, 255)
        Note over F,G: Step 1. 이미지 업로드
        F->>G: POST /api/hazard-analysis/upload<br/>(FormData: images[])
        G-->>F: 202 { jobId: "uuid" }
    end

    F->>G: POST /api/hazard-analysis/subscribe/{job_id}
    Note over F,G: SSE 연결 수립

    rect rgb(255, 250, 240)
        Note over G,M: Step 2. 비동기 파이프라인

        G-->>F: SSE: OCR_PROCESSING (30%)
        G->>M: 이미지 캡셔닝 (Gemma3 비전)<br/>각 이미지 → base64 → 위험요소 설명
        M-->>G: 캡셔닝 텍스트 (위험요소 나열)

        G-->>F: SSE: VECTOR_SEARCH (60%)

        G->>L: POST /generation/hazard-report<br/>{ captions: [{ image_index, description }] }
        L->>V: POST /retrieval/search<br/>{ query: 캡셔닝 텍스트, search_type: "kosha" }
        V-->>L: KOSHA Guide 검색 결과

        G-->>F: SSE: LLM_PROCESSING (90%)

        L->>M: POST /v1/chat/completions<br/>(캡셔닝 + KOSHA 컨텍스트 → 위험성 평가 프롬프트)
        M-->>L: 구조화 JSON 응답
        L-->>G: HazardAnalysisResponse[]<br/>{ image_index, risk_counts, risks[] }

        G->>G: thumbnail_url 추가
        G-->>F: SSE: COMPLETED (100%)
    end

    rect rgb(240, 255, 240)
        Note over F,G: Step 3. 결과 조회
        F->>G: GET /api/hazard-analysis/{job_id}/result
        G-->>F: HazardAnalysisResponse[]
    end

    F->>U: 결과 화면 렌더링<br/>(이미지별 위험 카운트 + 위험 목록)
```

---

## 4. 파트 간 데이터 인터페이스

### 4.1 프론트 → 백엔드 (API 호출)

| API | 프론트가 보내는 것 | 백엔드가 반환하는 것 |
|-----|-------------------|---------------------|
| `POST /api/labor-contract/upload` | FormData { contract_file, paystub_file? } | `{ job_id }` |
| `POST /api/labor-contract/subscribe/{id}` | (연결만) | SSE 이벤트 스트림 |
| `GET /api/labor-contract/{id}/result` | (없음) | `LaborContractAnalyzeResponse` |
| `GET /api/labor-contract/masked/{fileId}` | (없음) | Binary (PDF/Image) |
| `POST /api/hazard-analysis/upload` | FormData { images[] } | `{ jobId }` |
| `POST /api/hazard-analysis/subscribe/{id}` | (연결만) | SSE 이벤트 스트림 |
| `GET /api/hazard-analysis/{id}/result` | (없음) | `HazardAnalysisResponse[]` |
| `GET /api/hazard-analysis/{fileId}` | (없음) | Binary (Image) |

### 4.2 백엔드 → AI (내부 API 호출)

| API | 백엔드가 보내는 것 | AI가 반환하는 것 |
|-----|-------------------|-----------------|
| `POST :8002/analyze` | `{ contract, payslip?, assumptions }` | `RulesResult { sections, risk_flags, comparison? }` |
| `POST :8001/generation/labor-report` | `{ rules_result, contract_info }` | `{ summary, sections[] }` |
| `POST :8001/generation/hazard-report` | `{ captions[] }` | `[{ image_index, risk_counts, risks[] }]` |

### 4.3 AI 내부 (Generation → Retrieval)

| API | Generation이 보내는 것 | Retrieval이 반환하는 것 |
|-----|----------------------|----------------------|
| `POST :8000/retrieval/context-for-report` | `{ topics[], search_type }` | `{ contexts: [{ topic, results[] }] }` |
| `POST :8000/retrieval/search` | `{ query, top_k, search_type }` | `{ results: [{ content, score, citation }] }` |

---

## 5. SSE 상태 흐름

```mermaid
stateDiagram-v2
    [*] --> PENDING : Job 생성
    PENDING --> OCR_PROCESSING : 파이프라인 시작
    OCR_PROCESSING --> VECTOR_SEARCH : 구조화 완료
    VECTOR_SEARCH --> LLM_PROCESSING : 검색 완료
    LLM_PROCESSING --> COMPLETED : 리포트 생성 완료
    OCR_PROCESSING --> ERROR : 오류 발생
    VECTOR_SEARCH --> ERROR : 오류 발생
    LLM_PROCESSING --> ERROR : 오류 발생
```

| 상태 | 진행률 | 근로계약서 | 산재위험 |
|------|--------|-----------|---------|
| `PENDING` | 0% | 작업 대기 중 | 작업 대기 중 |
| `OCR_PROCESSING` | 30% | 근로조건 분석 중 | 현장 사진 분석 중 |
| `VECTOR_SEARCH` | 60% | 법령 검색 중 | KOSHA 가이드 검색 중 |
| `LLM_PROCESSING` | 90% | 리포트 생성 중 | 위험성 평가 생성 중 |
| `COMPLETED` | 100% | 분석 완료 | 분석 완료 |
| `ERROR` | - | 오류 메시지 | 오류 메시지 |

---

## 6. 최종 응답 스키마

### 6.1 LaborContractAnalyzeResponse

```mermaid
classDiagram
    class LaborContractAnalyzeResponse {
        +string? summary
        +Section[] sections
        +MaskedFile[]? masked_files
    }
    class Section {
        +string title
        +string status
        +Detail[] details
        +LegalBasis legal_basis
    }
    class Detail {
        +string label
        +string value
        +string? recommendation
    }
    class LegalBasis {
        +Law[] laws
        +Criteria[] criteria
    }
    class Law {
        +string law_name
        +string article
        +string content
    }
    class Criteria {
        +string label
        +string value
    }
    class MaskedFile {
        +string file_id
        +string file_name
        +string file_type
    }

    LaborContractAnalyzeResponse --> Section
    LaborContractAnalyzeResponse --> MaskedFile
    Section --> Detail
    Section --> LegalBasis
    LegalBasis --> Law
    LegalBasis --> Criteria
```

**status 값:** `"준수"` | `"미준수"` | `"주의"`

### 6.2 HazardAnalysisResponse

```mermaid
classDiagram
    class HazardImageResult {
        +number image_index
        +string thumbnail_url
        +RiskCounts risk_counts
        +HazardRisk[] risks
    }
    class RiskCounts {
        +number very_high
        +number high
        +number caution
        +number good
    }
    class HazardRisk {
        +string minor_category_from_image
        +string risk_level
        +string action_item
        +string action_code
    }

    HazardImageResult --> RiskCounts
    HazardImageResult --> HazardRisk
```

**risk_level 값:** `"매우 위험"` | `"위험"` | `"주의"` | `"양호"`

---

## 7. 파트별 의존 관계

```mermaid
graph LR
    subgraph Frontend
        F1[API Route 변경]
        F2[SSE 훅 추가]
        F3[결과 렌더링 수정]
    end

    subgraph Backend
        B1[Gateway API]
        B2[오케스트레이션]
        B3[OCR / 캡셔닝]
        B4[PDF 마스킹]
        B5[SSE 스트리밍]
    end

    subgraph AI
        A1[Rules Engine]
        A2[Retrieval Engine]
        A3[Generation Engine]
    end

    subgraph Infra
        I1[PostgreSQL + pgvector]
        I2[Gemma3 API]
    end

    F1 -->|호출| B1
    F2 -->|구독| B5
    F3 -->|조회| B1

    B2 -->|호출| A1
    B2 -->|호출| A3
    B3 -->|호출| I2

    A3 -->|호출| A2
    A3 -->|호출| I2
    A2 -->|검색| I1
```

### 구현 순서 의존성

```
Phase 1: 인프라 (Docker, DB)
    ↓
Phase 2: Retrieval Engine ──────┐
Phase 3: Rules Engine ──────────┤ (병렬 가능)
    ↓                           ↓
Phase 4: Generation Engine (Retrieval 필요)
    ↓
Phase 5: Gateway (3개 엔진 필요)
    ↓
Phase 6: Frontend (Gateway 필요)
    ↓
Phase 7: 통합 테스트
```

---

## 8. 협업 체크리스트

### AI ↔ 백엔드 합의 필요

- [ ] `ContractInfo` / `PayslipInfo` 필드명 확정
- [ ] `RulesResult.comparison` 응답 구조 확정
- [ ] `LaborContractAnalyzeResponse` 스키마 최종 확정
- [ ] `HazardAnalysisResponse` 스키마 최종 확정
- [ ] 에러 응답 형식 통일: `{ "error": string, "detail": string }`
- [ ] 각 엔진 `/health` 엔드포인트 형식

### 백엔드 ↔ 프론트 합의 필요

- [ ] CORS 허용 origin 확인
- [ ] SSE `Content-Type: text/event-stream` 확인
- [ ] FormData 필드명: `contract_file`, `paystub_file`, `images`
- [ ] 파일 크기 제한: 10MB
- [ ] 에러 응답 → 프론트 에러 UI 매핑

### AI ↔ 프론트 (직접 통신 없음, 스키마 합의)

- [ ] `sections[].status` 값 3종 확정 ("준수" / "미준수" / "주의")
- [ ] `legal_basis` 구조 확정 (laws + criteria)
- [ ] 급여명세서 대조 섹션 타이틀 확정
