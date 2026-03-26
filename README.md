# Hello RAG System

Vector DB 기반 RAG 시스템 + FastAPI 백엔드.
근로계약서/임금명세서 분석과 산재위험요소 자율점검을 제공한다.

---

## 문서

| 문서 | 설명 |
|------|------|
| [HANDOFF_OVERALL](docs/HANDOFF_OVERALL.md) | 전체 통합 뷰 — 파트 간 데이터 흐름, Mermaid 시퀀스, 협업 체크리스트 |
| [HANDOFF_FOR_AI](docs/HANDOFF_FOR_AI.md) | AI 담당 — Retrieval / Rules / Generation 엔진 API 입출력 명세 |
| [HANDOFF_FOR_BACKEND](docs/HANDOFF_FOR_BACKEND.md) | 백엔드 담당 — Gateway, 오케스트레이션, SSE, OCR, 마스킹 |
| [HANDOFF_FOR_FRONTEND](docs/HANDOFF_FOR_FRONTEND.md) | 프론트 담당 — API 변경, SSE 훅, 결과 렌더링 수정 |
| [GENERATION_ENGINE_DIFF](docs/GENERATION_ENGINE_DIFF.md) | Generation Engine 변경 명세 — 기존 vs 신규 출력 구조, 프롬프트, 계약-급여 대조 |

---

## Mermaid 다이어그램 미리보기

`HANDOFF_OVERALL.md`에 포함된 Mermaid 다이어그램을 브라우저에서 바로 확인할 수 있다.

### 사용법

1. [mermaid.live](https://mermaid.live) 접속
2. 왼쪽 에디터에 기존 코드를 지우고, 아래 코드 블록 중 하나를 복사/붙여넣기
3. 오른쪽에 실시간으로 다이어그램이 렌더링됨
4. 상단 메뉴에서 PNG/SVG 다운로드 가능

### 근로계약서 분석 시퀀스

```text
sequenceDiagram
    participant U as 사용자
    participant F as Frontend(Next.js)
    participant G as Gateway(:8080)
    participant R as Rules Engine(:8002)
    participant V as Retrieval Engine(:8000)
    participant L as Generation Engine(:8001)
    participant M as Gemma3 LLM

    U->>F: 근로계약서 + 급여명세서(선택) 업로드

    rect rgb(240, 248, 255)
        Note over F,G: Step 1. 파일 업로드
        F->>G: POST /api/labor-contract/upload
        G-->>F: 202 { job_id }
    end

    F->>G: POST /subscribe/{job_id}
    Note over F,G: SSE 연결 수립

    rect rgb(255, 250, 240)
        Note over G,M: Step 2. 비동기 파이프라인 (3-5분)

        G->>G: OCR 구조화 추출
        G-->>F: SSE: OCR_PROCESSING (30%)

        G->>R: POST /analyze { contract, payslip? }
        R->>R: 임금 계산 + 규칙 판정 + 계약-급여 대조
        R-->>G: RulesResult { sections, comparison? }

        G-->>F: SSE: VECTOR_SEARCH (60%)

        G->>L: POST /generation/labor-report
        L->>V: POST /retrieval/context-for-report
        V->>V: 하이브리드 검색 (벡터 + 키워드)
        V-->>L: 법령 검색 결과

        G-->>F: SSE: LLM_PROCESSING (90%)

        L->>M: POST /v1/chat/completions
        M-->>L: 구조화 JSON 응답
        L->>L: JSON 파싱 + 스키마 검증
        L-->>G: LaborContractAnalyzeResponse

        G->>G: PDF 마스킹 + masked_files 추가
        G-->>F: SSE: COMPLETED (100%)
    end

    rect rgb(240, 255, 240)
        Note over F,G: Step 3. 결과 조회
        F->>G: GET /{job_id}/result
        G-->>F: { summary, sections[], masked_files[] }
    end

    F->>U: 결과 화면 렌더링
```

### 산재위험요소 분석 시퀀스

```text
sequenceDiagram
    participant U as 사용자
    participant F as Frontend(Next.js)
    participant G as Gateway(:8080)
    participant V as Retrieval Engine(:8000)
    participant L as Generation Engine(:8001)
    participant M as Gemma3 LLM

    U->>F: 현장 사진 업로드 (1장 이상)

    rect rgb(240, 248, 255)
        Note over F,G: Step 1. 이미지 업로드
        F->>G: POST /api/hazard-analysis/upload
        G-->>F: 202 { jobId }
    end

    F->>G: POST /subscribe/{job_id}
    Note over F,G: SSE 연결 수립

    rect rgb(255, 250, 240)
        Note over G,M: Step 2. 비동기 파이프라인

        G-->>F: SSE: OCR_PROCESSING (30%)
        G->>M: 이미지 캡셔닝 (Gemma3 비전)
        M-->>G: 위험요소 설명 텍스트

        G-->>F: SSE: VECTOR_SEARCH (60%)

        G->>L: POST /generation/hazard-report
        L->>V: POST /retrieval/search (KOSHA)
        V-->>L: KOSHA Guide 검색 결과

        G-->>F: SSE: LLM_PROCESSING (90%)

        L->>M: POST /v1/chat/completions
        M-->>L: 구조화 JSON 응답
        L-->>G: HazardAnalysisResponse[]

        G->>G: thumbnail_url 추가
        G-->>F: SSE: COMPLETED (100%)
    end

    rect rgb(240, 255, 240)
        Note over F,G: Step 3. 결과 조회
        F->>G: GET /{job_id}/result
        G-->>F: HazardAnalysisResponse[]
    end

    F->>U: 결과 화면 렌더링
```

### SSE 상태 흐름

```text
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

### 응답 스키마 (클래스 다이어그램)

```text
classDiagram
    class LaborContractAnalyzeResponse {
        +string summary
        +Section[] sections
        +MaskedFile[] masked_files
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
        +string recommendation
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

### 파트 의존 관계

```text
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
