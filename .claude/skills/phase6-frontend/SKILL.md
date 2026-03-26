---
name: phase6-frontend
description: "Phase 6: ax-frontend 연동. API URL 변경, SSE 구독 훅 생성, 결과 화면 수정 (sections 기반 렌더링, 법적근거 팝업, 상태 배지)."
---

# Phase 6: 프론트엔드 연동

## 목표

기존 ax-frontend의 API 호출을 FastAPI Gateway로 변경하고, SSE 진행률 구독 및 새 Response 스키마에 맞춰 결과 화면을 수정한다.

## 사전 조건

- Phase 5 완료 (Gateway API 동작)
- ax-frontend 프로젝트: `~/Documents/Projects/ax-frontend`

## 작업 순서

### Step 1: 환경 변수 변경

`.env.local`:
```env
# 기존
# GEMMA3_API_URL=http://211.233.14.18:8000

# 변경: FastAPI Gateway로 통합
GATEWAY_API_URL=http://localhost:8080
```

### Step 2: API Route 변경 - 근로계약서

`src/app/api/labor-contract/analyze/route.ts` 수정:

**기존:** Gemma3 직접 호출 (이미지 base64 → LLM)
**변경:** Gateway 프록시

```typescript
// 기존: Gemma3 직접 호출
const response = await fetch(`${GEMMA3_API_URL}/v1/chat/completions`, { ... })

// 변경: Gateway 프록시
export async function POST(request: NextRequest) {
  const formData = await request.formData()

  // 1. 파일 업로드 → job_id 획득
  const uploadRes = await fetch(`${GATEWAY_API_URL}/api/labor-contract/upload`, {
    method: 'POST',
    body: formData,
  })
  const { job_id } = await uploadRes.json()

  return NextResponse.json({ job_id }, { status: 202 })
}
```

### Step 3: API Route 변경 - 산재위험

`src/app/api/analyze/route.ts` 수정:

동일 패턴으로 Gateway 프록시로 변경:
```typescript
// Gateway로 이미지 전달
const uploadRes = await fetch(`${GATEWAY_API_URL}/api/hazard-analysis/upload`, {
  method: 'POST',
  body: formData,
})
const { jobId } = await uploadRes.json()
return NextResponse.json({ jobId }, { status: 202 })
```

### Step 4: SSE 구독 공통 훅

`src/shared/lib/hooks/useSSESubscription.ts` 신규 생성:

```typescript
export function useSSESubscription(jobId: string | null, serviceType: 'labor-contract' | 'hazard-analysis') {
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
      if (data.status === 'COMPLETED' || data.status === 'ERROR') {
        eventSource.close()
      }
    }

    eventSource.onerror = () => {
      setError('연결이 끊어졌습니다')
      eventSource.close()
    }

    return () => eventSource.close()
  }, [jobId, serviceType])

  return { status, error }
}
```

### Step 5: 로딩 화면 연동

기존 로딩 화면 컴포넌트에 SSE status 매핑:

| SSE Status | 로딩 단계 | 진행률 | 표시 텍스트 |
|------------|----------|--------|------------|
| `PENDING` | 0 | 0% | 작업 대기 중... |
| `OCR_PROCESSING` | 1 | 30% | 근로조건 분석 중... / 이미지 분석 중... |
| `VECTOR_SEARCH` | 2 | 60% | 법령 검색 중... / KOSHA 가이드 검색 중... |
| `LLM_PROCESSING` | 3 | 90% | 리포트 생성 중... |
| `COMPLETED` | 4 | 100% | 분석 완료 |
| `ERROR` | - | - | 오류 발생 |

### Step 6: 결과 화면 수정 - 근로계약서

`src/features/labor-contract/segments/summary/` 수정:

**기존:** LLM 마크다운 응답 직접 렌더링
**변경:** 구조화 JSON 기반 렌더링

```
┌─────────────────────────────────────────┐
│ 업로드된 파일                            │
│  [파일카드] [마스킹PDF카드]              │  ← masked_files[]
├─────────────────────────────────────────┤
│ 결과 요약                                │  ← summary
│  · 요약 텍스트                           │
├─────────────────────────────────────────┤
│ 1. 최저임금 준수 여부      [준수 ✓]      │  ← sections[0]
│  · label: value                          │  ← details[]
│  · label: value                          │
│  · 권고: recommendation                  │
│                       [법적근거 보기 →]   │  ← legal_basis
├─────────────────────────────────────────┤
│ 2. 주 52시간제            [주의 ⚠]       │  ← sections[1]
│  ...                                     │
├─────────────────────────────────────────┤
│ ※ 면책 고지 (고정 문구)                   │
└─────────────────────────────────────────┘
```

핵심 컴포넌트:
- `SectionCard` — sections[] 순회, title + status 배지 + details 렌더링
- `StatusBadge` — 준수(녹색) / 미준수(적색) / 주의(황색)
- `LegalBasisPopup` — 법적근거 보기 클릭 시 laws[] + criteria[] 팝업
- `MaskedFileCard` — masked_files[] 파일 미리보기 카드

### Step 7: 결과 화면 수정 - 산재위험

`src/features/hazard-assessment/` 수정:

기존 응답 구조와 새 `HazardAnalysisResponse[]` 매핑 확인.
이미지별 risk_counts + risks[] 렌더링.

### Step 8: 응답 파서 변경

`src/shared/lib/llm-response-parser/` 수정:

- 기존: LLM 마크다운 텍스트 파싱 (정규표현식 기반)
- 변경: 구조화 JSON 직접 사용 (파싱 불필요, 타입 검증만)

### Step 9: SSE 프록시 라우트 (필요 시)

CORS 우회를 위해 Next.js API Route에서 SSE 프록시:

```typescript
// src/app/api/labor-contract/subscribe/[jobId]/route.ts
export async function POST(request, { params }) {
  const backendUrl = `${GATEWAY_API_URL}/api/labor-contract/subscribe/${params.jobId}`
  const response = await fetch(backendUrl, { method: 'POST' })

  return new Response(response.body, {
    headers: { 'Content-Type': 'text/event-stream' },
  })
}
```

### Step 10: 테스트

```
- 파일 업로드 → job_id 반환 확인
- SSE 구독 → 상태 업데이트 수신 확인
- COMPLETED 후 결과 조회 → 정상 렌더링
- 법적근거 팝업 동작
- 마스킹 파일 미리보기
- 에러 상태 핸들링
```

## 완료 기준

- [ ] Gemma3 직접 호출 제거, Gateway 호출로 전환 완료
- [ ] `useSSESubscription` 훅 동작 (상태 실시간 업데이트)
- [ ] 로딩 화면에 진행률 단계별 표시
- [ ] 근로계약서 결과: sections 기반 렌더링 + 상태 배지 + 법적근거 팝업
- [ ] 마스킹 PDF 미리보기 동작
- [ ] 산재위험 결과: 이미지별 위험 평가 렌더링
- [ ] 에러 상태 시 사용자 안내 표시

## 참고

- 프론트 프로젝트: `~/Documents/Projects/ax-frontend`
- 기존 API Route: `src/app/api/labor-contract/analyze/route.ts`
- 기존 결과 화면: `src/features/labor-contract/segments/summary/`
- 결과 화면 스크린샷: PLAN.md 섹션 7.1 참조
