# HANDOFF: 프론트엔드 담당

> ax-frontend 수정 담당자를 위한 문서.
> 백엔드 Gateway API를 호출하고, 결과를 렌더링하는 방법.

---

## 담당 범위

```
┌─────────────────────────────────────────────────────────────┐
│                     프론트엔드 영역                           │
│                     (ax-frontend)                           │
│                                                             │
│  ┌────────────────────────────────────────┐                 │
│  │  API Routes (Next.js)                  │                 │
│  │  · /api/labor-contract/analyze         │────▶ Gateway    │
│  │  · /api/analyze                        │      :8080      │
│  │  · /api/labor-contract/subscribe/*     │◀─── SSE 이벤트  │
│  └────────────────────────────────────────┘                 │
│                                                             │
│  ┌────────────────────────────────────────┐                 │
│  │  Features                              │                 │
│  │  · labor-contract/ (근로계약서)         │                 │
│  │  · hazard-assessment/ (산재위험)        │                 │
│  └────────────────────────────────────────┘                 │
│                                                             │
│  ┌────────────────────────────────────────┐                 │
│  │  Shared                                │                 │
│  │  · useSSESubscription (신규)           │                 │
│  │  · llm-response-parser (변경)          │                 │
│  └────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

**내가 바꾸는 것:** API Route, SSE 훅, 결과 렌더링, 응답 파서
**내가 바꾸지 않는 것:** 파일 업로드 UI, 로딩 UI 구조 (기존 유지)

---

## 1. 변경 전후 비교

### API 호출 방식

```
[기존]
프론트 ──── base64 이미지 ──▶ Gemma3 API (직접)
프론트 ◀── 마크다운 텍스트 ── Gemma3 API

[변경]
프론트 ──── 파일 업로드 ────▶ Gateway /upload
프론트 ◀── { job_id } ────── Gateway
프론트 ──── SSE 구독 ───────▶ Gateway /subscribe/{job_id}
프론트 ◀── 진행률 이벤트 ─── Gateway (SSE)
프론트 ──── 결과 요청 ──────▶ Gateway /{job_id}/result
프론트 ◀── 구조화 JSON ───── Gateway
```

### 응답 형식

```
[기존] Gemma3가 반환하는 것:
  마크다운 텍스트 → 프론트에서 파싱 필요

[변경] Gateway가 반환하는 것:
  구조화 JSON → 바로 렌더링 가능 (파싱 불필요)
```

---

## 2. 근로계약서 분석 흐름

### 전체 시퀀스

```
사용자           프론트                    Gateway
  │                │                        │
  │  파일 선택     │                        │
  ├───────────────▶│                        │
  │                │                        │
  │                │  POST /api/labor-      │
  │                │  contract/upload       │
  │                │  (FormData)            │
  │                ├───────────────────────▶│
  │                │                        │
  │                │  202 { job_id }        │
  │                │◀───────────────────────┤
  │                │                        │
  │  로딩 시작     │  POST /subscribe/      │
  │◀───────────────│  {job_id}              │
  │                ├───────────────────────▶│
  │                │                        │
  │  30% 표시      │  SSE: OCR_PROCESSING   │
  │◀───────────────│◀───────────────────────┤
  │                │                        │
  │  60% 표시      │  SSE: VECTOR_SEARCH    │
  │◀───────────────│◀───────────────────────┤
  │                │                        │
  │  90% 표시      │  SSE: LLM_PROCESSING   │
  │◀───────────────│◀───────────────────────┤
  │                │                        │
  │                │  SSE: COMPLETED        │
  │                │◀───────────────────────┤
  │                │                        │
  │                │  GET /{job_id}/result   │
  │                ├───────────────────────▶│
  │                │                        │
  │  결과 화면     │  LaborContractAnalyze  │
  │◀───────────────│  Response (JSON)       │
  │                │◀───────────────────────┤
```

### Step 1: 파일 업로드

```typescript
// src/app/api/labor-contract/analyze/route.ts

export async function POST(request: NextRequest) {
  const formData = await request.formData()

  // Gateway에 파일 전달
  const res = await fetch(`${GATEWAY_API_URL}/api/labor-contract/upload`, {
    method: 'POST',
    body: formData,   // contract_file, paystub_file?(선택)
  })

  const { job_id } = await res.json()
  return NextResponse.json({ job_id }, { status: 202 })
}
```

### Step 2: SSE 구독

```typescript
// src/shared/lib/hooks/useSSESubscription.ts  (신규 생성)

type AnalysisStatus =
  | 'PENDING'
  | 'OCR_PROCESSING'
  | 'VECTOR_SEARCH'
  | 'LLM_PROCESSING'
  | 'COMPLETED'
  | 'ERROR'

export function useSSESubscription(
  jobId: string | null,
  serviceType: 'labor-contract' | 'hazard-analysis'
) {
  const [status, setStatus] = useState<AnalysisStatus>('PENDING')
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    if (!jobId) return

    const eventSource = new EventSource(
      `/api/${serviceType}/subscribe/${jobId}`
    )

    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data)
      setStatus(data.status)

      if (data.status === 'ERROR') {
        setError(data.message ?? '분석 중 오류가 발생했습니다')
        eventSource.close()
      }
      if (data.status === 'COMPLETED') {
        eventSource.close()
      }
    }

    eventSource.onerror = () => {
      setError('서버 연결이 끊어졌습니다')
      eventSource.close()
    }

    return () => eventSource.close()
  }, [jobId, serviceType])

  return { status, error }
}
```

### Step 3: 로딩 화면 매핑

| SSE status | 진행률 | 표시 텍스트 |
|------------|--------|------------|
| `PENDING` | 0% | 작업 대기 중... |
| `OCR_PROCESSING` | 30% | 근로조건 분석 중... |
| `VECTOR_SEARCH` | 60% | 관련 법령 검색 중... |
| `LLM_PROCESSING` | 90% | 분석 리포트 생성 중... |
| `COMPLETED` | 100% | 분석 완료 |
| `ERROR` | - | 오류 메시지 표시 |

### Step 4: 결과 조회

```typescript
// COMPLETED 수신 후
const res = await fetch(`/api/labor-contract/${jobId}/result`)
const data: LaborContractAnalyzeResponse = await res.json()
// → SummaryPhase에 전달
```

---

## 3. 응답 스키마 & 렌더링 매핑

### LaborContractAnalyzeResponse

```typescript
interface LaborContractAnalyzeResponse {
  summary?: string
  sections: Section[]
  masked_files?: MaskedFile[]
}

interface Section {
  title: string
  status: '준수' | '미준수' | '주의'
  details: Detail[]
  legal_basis: LegalBasis
}

interface Detail {
  label: string
  value: string
  recommendation?: string
}

interface LegalBasis {
  laws: Law[]
  criteria: Criteria[]
}

interface Law {
  law_name: string
  article: string
  content: string
}

interface Criteria {
  label: string
  value: string
}

interface MaskedFile {
  file_id: string
  file_name: string
  file_type: 'pdf' | 'image'
}
```

### 화면 매핑

```
┌───────────────────────────────────────────────────────────┐
│ 업로드된 파일                            ← masked_files[] │
│  [파일카드: file_name]  [파일카드: file_name]             │
│  클릭 → GET /api/labor-contract/masked/{file_id}         │
├───────────────────────────────────────────────────────────┤
│ 결과 요약                                   ← summary    │
│  · {summary 텍스트}                                       │
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│                                                           │
│  sections.map((section) => (             ← sections[]     │
│                                                           │
│  ┌────────────────────────────────────────────────────┐   │
│  │ {section.title}               [{section.status}]   │   │
│  │                                ↑ 배지 색상:        │   │
│  │                                준수=녹 미준수=적    │   │
│  │                                주의=황              │   │
│  │                                                    │   │
│  │  section.details.map((d) => (   ← details[]       │   │
│  │    · {d.label} : {d.value}                         │   │
│  │    d.recommendation &&                             │   │
│  │      → 권고: {d.recommendation}                    │   │
│  │  ))                                                │   │
│  │                                                    │   │
│  │                           [법적근거 보기 →]        │   │
│  │                            ↓ 클릭 시 팝업          │   │
│  │                                                    │   │
│  │  ┌─ LegalBasisPopup ────────────────────────────┐ │   │
│  │  │ 1. 적용 법령             ← legal_basis.laws  │ │   │
│  │  │  · {law_name} {article}                      │ │   │
│  │  │    "{content}"                               │ │   │
│  │  │                                              │ │   │
│  │  │ 2. 판단기준          ← legal_basis.criteria  │ │   │
│  │  │  · {label}: {value}                          │ │   │
│  │  └──────────────────────────────────────────────┘ │   │
│  └────────────────────────────────────────────────────┘   │
│                                                           │
│  ))                                                       │
│                                                           │
├───────────────────────────────────────────────────────────┤
│ ※ 본 결과는 AI 분석에 기반한 참고용 자료이며,             │
│   법적 효력이 없습니다. (프론트 하드코딩)                  │
└───────────────────────────────────────────────────────────┘
```

### 급여명세서가 있을 때

sections 배열에 "계약-급여 대조 결과" 섹션이 추가로 포함된다.
프론트에서는 별도 분기 없이 동일하게 `sections.map()` 으로 렌더링하면 된다.

```json
{
  "title": "4. 계약-급여 대조 결과",
  "status": "주의",
  "details": [
    { "label": "기본급 대조", "value": "계약서: 2,600,000원 / 명세서: 2,600,000원 → 일치" },
    { "label": "식대", "value": "계약서: 300,000원 / 명세서: 200,000원 → 불일치",
      "recommendation": "계약서에 명시된 금액과 실제 지급액이 다릅니다." }
  ],
  "legal_basis": { ... }
}
```

---

## 4. 산재위험 분석 흐름

### 응답 스키마

```typescript
type HazardAnalysisResponse = HazardImageResult[]

interface HazardImageResult {
  image_index: number
  thumbnail_url: string
  risk_counts: {
    very_high: number
    high: number
    caution: number
    good: number
  }
  risks: HazardRisk[]
}

interface HazardRisk {
  minor_category_from_image: string
  risk_level: '매우 위험' | '위험' | '주의' | '양호'
  action_item: string        // KOSHA 가이드 조치사항
  action_code: string        // KOSHA GUIDE 코드
}
```

### 호출 흐름 (근로계약서와 동일 패턴)

```
POST /api/hazard-analysis/upload     → { jobId }
POST /api/hazard-analysis/subscribe/{jobId}  → SSE 이벤트
GET  /api/hazard-analysis/{jobId}/result     → HazardAnalysisResponse[]
GET  /api/hazard-analysis/{fileId}           → 이미지 바이너리
```

SSE status 매핑:

| SSE status | 표시 텍스트 |
|------------|------------|
| `OCR_PROCESSING` | 현장 사진 분석 중... |
| `VECTOR_SEARCH` | KOSHA 가이드 검색 중... |
| `LLM_PROCESSING` | 위험성 평가 리포트 생성 중... |

---

## 5. 수정 파일 목록

### 변경 파일

| 파일 | 변경 내용 |
|------|-----------|
| `src/app/api/labor-contract/analyze/route.ts` | Gemma3 직접 호출 → Gateway `/upload` 프록시 |
| `src/app/api/analyze/route.ts` | Gemma3 직접 호출 → Gateway `/upload` 프록시 |
| `src/features/labor-contract/segments/summary/` | 마크다운 렌더링 → sections JSON 렌더링 |
| `src/features/labor-contract/segments/summary/ui/types.ts` | 타입 수정 (위 스키마로) |
| `src/shared/lib/llm-response-parser.ts` | 마크다운 파서 → JSON 직접 사용 (대폭 간소화) |
| `.env.local` | `GEMMA3_API_URL` → `GATEWAY_API_URL` |

### 신규 파일

| 파일 | 내용 |
|------|------|
| `src/shared/lib/hooks/useSSESubscription.ts` | SSE 구독 공통 훅 |
| `src/app/api/labor-contract/subscribe/[jobId]/route.ts` | SSE 프록시 (CORS 우회) |
| `src/app/api/hazard-analysis/subscribe/[jobId]/route.ts` | SSE 프록시 (CORS 우회) |

### 변경 없는 파일

| 파일 | 이유 |
|------|------|
| `src/features/labor-contract/segments/input/` | 파일 업로드 UI 그대로 |
| `src/features/labor-contract/segments/summary/ui/LegalBasisPopup.tsx` | 법적근거 팝업 구조 동일 (타입만 맞춤) |
| `src/shared/ui/` | 기존 UI 컴포넌트 재사용 |

---

## 6. 환경 변수

```env
# .env.local

# 기존 (제거)
# GEMMA3_API_URL=http://211.233.14.18:8000

# 신규
GATEWAY_API_URL=http://localhost:8080
```

---

## 7. 백엔드 담당자에게 확인할 것

| 확인 항목 | 상세 |
|-----------|------|
| CORS 설정 | 프론트 origin (`http://localhost:3000`) 허용 여부 |
| SSE Content-Type | `text/event-stream` 헤더 확인 |
| 파일 업로드 필드명 | FormData 키: `contract_file`, `paystub_file`, `images` |
| 파일 크기 제한 | 최대 10MB 확인 |
| 에러 응답 형식 | `{ "error": string }` 통일 여부 |

---

## 8. 기존 코드와 달라지는 핵심 포인트

```
[기존]
1. 프론트가 이미지를 base64로 변환
2. Gemma3에 직접 전송 (동기, 1회 호출)
3. 마크다운 텍스트 응답 수신
4. 정규표현식으로 파싱 (실패 가능)
5. 파싱 결과를 렌더링

[변경]
1. 프론트가 파일을 FormData로 전송            ← 더 간단
2. Gateway가 job_id 즉시 반환 (비동기)        ← 블로킹 없음
3. SSE로 진행률 실시간 수신                    ← UX 개선
4. COMPLETED 시 구조화 JSON 조회              ← 파싱 불필요
5. JSON을 바로 렌더링                         ← 안정적
```
