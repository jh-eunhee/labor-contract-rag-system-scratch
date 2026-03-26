# Generation Engine 변경 명세

기존 레퍼런스 시스템(basic-labor-standards-self-Inspection)과 새 시스템(hello-rag-system)의 Generation Engine 차이를 정리한다.

---

## 1. 핵심 차이 요약

| 구분 | 기존 (레퍼런스) | 신규 (hello-rag-system) |
|------|----------------|----------------------|
| **출력 목적** | 항목별 상세 법리 분석 (전문가용) | 요약된 준수 여부 리포트 (일반 사용자용) |
| **출력 형식** | 마크다운 6개 섹션 (자유 서술) | 구조화 JSON (sections 배열) |
| **LLM** | Gemini 2.5 Flash | Gemma3 27B (자체 호스팅) |
| **API 형식** | Gemini SDK (`generate_content`) | OpenAI 호환 (`/v1/chat/completions`) |
| **temperature** | 0.0 | 0.2 |
| **출력 길이** | 제한 없음 (상세 서술) | max_tokens: 4000 (간결) |
| **프론트 파싱** | 마크다운 텍스트 렌더링 | JSON 직접 사용 (파싱 불필요) |
| **급여명세서** | 단독 분석 | 근로계약서와 항목 대조 비교 |

---

## 2. 입력 구조: 근로계약서 + 급여명세서 대조

### 2.1 업로드 케이스

| 케이스 | 입력 | 분석 범위 |
|--------|------|-----------|
| **1. 근로계약서만** | contract_file | 계약 조건 기반 법적 준수 여부 판정 |
| **2. 근로계약서 + 급여명세서** | contract_file + paystub_file | 계약 조건 vs 실제 지급 내역 대조 → 불일치 탐지 |
| **3. 급여명세서만** | paystub_file | 최저임금 준수 여부 등 기본 판정만 가능 |

**급여명세서는 필수가 아니지만, 정확한 분석을 위해 권장한다.**

### 2.2 대조 비교 항목

근로계약서와 급여명세서를 함께 제출하면 아래 항목을 크로스체크한다:

| 대조 항목 | 근로계약서 (계약 조건) | 급여명세서 (실제 지급) | 불일치 시 판정 |
|-----------|----------------------|---------------------|---------------|
| **기본급** | `base_salary` | `base_salary` | 미준수 — 계약 대비 실제 지급액 부족 |
| **연장근로수당** | 계약서 내 연장근로 조항 유무 | `overtime_pay` 지급 여부/금액 | 주의 — 포괄임금 의심 또는 미지급 |
| **야간근로수당** | 야간근로 조항 유무 | `night_pay` 지급 여부/금액 | 주의 — 미지급 가능성 |
| **휴일근로수당** | 휴일근로 조항 유무 | `holiday_pay` 지급 여부/금액 | 주의 — 미지급 가능성 |
| **총 지급액** | 기본급 + 수당 합산 | `total_payment` | 미준수 — 계약 금액과 실지급액 차이 |
| **공제 항목** | (명시 안 될 수 있음) | `deductions` (4대보험, 소득세 등) | 참고 — 과다 공제 여부 확인 |
| **수당 항목** | `allowances` (식대, 교통비 등) | 명세서 내 수당 항목 | 주의 — 계약서에 있으나 미지급, 또는 반대 |

### 2.3 대조 분석이 추가하는 sections

급여명세서가 있을 때만 추가되는 분석 섹션:

```json
{
  "title": "4. 계약-급여 대조 결과",
  "status": "주의",
  "details": [
    {
      "label": "기본급 대조",
      "value": "계약서: 월 2,600,000원 / 명세서: 월 2,600,000원 → 일치"
    },
    {
      "label": "연장근로수당",
      "value": "계약서: 별도 조항 없음 (포괄임금 의심) / 명세서: 0원 → 미지급",
      "recommendation": "연장근로가 발생했다면 통상시급의 1.5배를 지급해야 합니다."
    },
    {
      "label": "식대",
      "value": "계약서: 월 300,000원 / 명세서: 월 200,000원 → 불일치 (100,000원 차이)",
      "recommendation": "계약서에 명시된 금액과 실제 지급액이 다릅니다. 확인이 필요합니다."
    }
  ],
  "legal_basis": {
    "laws": [
      {
        "law_name": "근로기준법",
        "article": "제43조 (임금 지급)",
        "content": "임금은 통화로 직접 근로자에게 그 전액을 지급하여야 한다."
      }
    ],
    "criteria": [
      { "label": "대조 기준", "value": "근로계약서 명시 금액 vs 급여명세서 실지급액" }
    ]
  }
}
```

---

## 3. 기존 시스템 출력 구조

### 3.1 GenerateResponse

```python
{
    "ok": bool,
    "answer": str,                    # 마크다운 6개 섹션 (자유 서술 텍스트)
    "search_queries": list[str],
    "citations": list[CitableArticle],
    "retrieved_chunks": list[RetrievedChunk],
    "grouped_legal_basis": dict,
    "disclaimer": str
}
```

### 3.2 answer 필드 (마크다운 6개 섹션)

LLM이 아래 6개 섹션을 **자유 서술 마크다운**으로 생성:

```markdown
## 1. 사안 요약
근로계약서에 명시된 근로조건을 정리하면...

## 2. 핵심 판단
규칙 엔진 분석 결과, 통상시급은 X원으로 최저임금 Y원을 상회하여...

## 3. 법령 근거에 따른 설명
근로기준법 제17조(근로조건의 명시)에 따르면...

## 4. 추가 확인 필요 사항
- 실제 연장근로 시간 기록...

## 5. 결론
전반적으로 법적 기준을 준수하고 있으나...

## 6. 주요 근거 요약
- 근로기준법 제17조: 근로조건 명시 의무
```

### 3.3 특징

- **전문가 지향**: 법리 분석이 상세하고 근거 조문을 본문에 인라인으로 인용
- **자유 서술**: 각 섹션의 길이와 내용이 케이스마다 다름
- **citations 별도 제공**: 본문과 별개로 `CitableArticle` 배열로 인용 법령 목록 제공
- **rules_result 최우선**: 규칙 엔진 결과를 1차 판단으로, RAG는 보충 근거로 사용

---

## 4. 신규 시스템 출력 구조

### 4.1 LaborContractAnalyzeResponse

프론트엔드가 직접 렌더링할 수 있는 **구조화 JSON**:

```json
{
  "summary": "전반적으로 법적 기준을 준수하고 있으나, 제 5조(연장근로) 조항은 향후 분쟁 소지를 줄이기 위해 수정을 고려해 보시는 것이 좋습니다.",
  "sections": [
    {
      "title": "1. 최저임금 준수 여부",
      "status": "준수",
      "details": [
        { "label": "계약상 급여", "value": "월 2,900,000원 (기본급 2,600,000원 + 식대 300,000원)" },
        { "label": "월 소정근로", "value": "209시간 (주 40시간 기준)" },
        { "label": "검토 결과", "value": "계약서에 명시된 임금은 법적 최저임금 이상이며, 정상으로 확인되었습니다." }
      ],
      "legal_basis": {
        "laws": [
          { "law_name": "최저임금법", "article": "제6조 (최저임금의 효력)", "content": "사용자는 최저임금액 이상의 임금을 지급하여야 한다." }
        ],
        "criteria": [
          { "label": "적용 연도", "value": "2025년" },
          { "label": "시간급 최저임금", "value": "10,030원" }
        ]
      }
    },
    {
      "title": "2. 주 52시간제 (연장근로)",
      "status": "주의",
      "details": [
        { "label": "기본 근무 시간", "value": "09:00 ~ 18:00 (휴게 1시간) / 주 5일 (실 근로시간 40시간)" },
        { "label": "검토 대상 조항", "value": "제 5조 (연장근로): '을'은 '갑'의 연장, 야간, 휴일근로 지시에 동의한 것으로 간주한다." },
        { "label": "검토 결과", "value": "해당 조항은 '포괄적 동의'로 해석될 여지가 있습니다." },
        {
          "label": "권고",
          "value": "근로기준법상 연장근로는 당사자 간의 '개별 합의'를 원칙으로 합니다.",
          "recommendation": "\"연장근로 발생 시 별도 합의 절차를 거친다\"와 같이 구체적인 절차를 명시하는 것을 권장합니다."
        }
      ],
      "legal_basis": {
        "laws": [
          { "law_name": "근로기준법", "article": "제53조 (연장 근로의 제한)", "content": "당사자 간에 합의하면 1주 간에 12시간을 한도로 근로시간을 연장할 수 있다." }
        ],
        "criteria": [
          { "label": "연장근로 한도", "value": "주 12시간" }
        ]
      }
    },
    {
      "title": "3. 기타 검토 의견",
      "status": "준수",
      "details": [
        { "label": "기재사항", "value": "업무 내용, 소정근로시간, 휴일, 휴가 등 기본 기재 사항은 모두 포함되어 있습니다." },
        { "label": "유의사항", "value": "주휴수당은 '일주일간 소정근로일 모두 출근' 기준으로 계산되며, 일부 항목은 근무기록 확인이 필요할 수 있습니다." }
      ],
      "legal_basis": {
        "laws": [
          { "law_name": "근로기준법", "article": "제17조 (근로조건의 명시)", "content": "사용자는 근로계약을 체결할 때에 임금, 소정근로시간, 휴일, 연차 유급휴가 등을 명시하여야 한다." }
        ],
        "criteria": []
      }
    }
  ],
  "masked_files": [
    { "file_id": "abc123def456", "file_name": "2025_JH제약 근로계약서_홍길동.pdf", "file_type": "pdf" }
  ]
}
```

**급여명세서 함께 제출 시**, sections에 "4. 계약-급여 대조 결과" 섹션이 추가된다 (섹션 2.3 참조).

### 4.2 특징

- **일반 사용자 지향**: 법률 용어를 최소화하고 핵심만 전달
- **구조화 JSON**: 프론트가 파싱 없이 바로 렌더링
- **status 배지**: 각 섹션에 "준수"/"미준수"/"주의" 명시
- **법적근거 분리**: 본문에 인라인 인용 대신 `legal_basis` 객체로 분리 (팝업 표시)
- **계약-급여 대조**: 급여명세서가 있으면 항목별 크로스체크 결과 추가

---

## 5. 프롬프트 변경

### 5.1 기존 프롬프트 (레퍼런스)

```
역할: 한국 노동법 전문가
입력: rules_result + retrieval_context + user_question
출력: 마크다운 6개 섹션 자유 서술
제약:
  - rules_result가 최우선 판단 근거
  - 새로운 사실을 만들지 마라
  - 불확실하면 "추가 확인 필요"
  - 인용 법령은 retrieval_context에서만
```

### 5.2 신규 프롬프트

```
역할: 한국 노동법 AI 어시스턴트 (일반 사용자 대상)
입력: rules_result + retrieval_context + (선택) payslip_comparison
출력: 반드시 아래 JSON 스키마로 응답

지시사항:
  1. summary: 전체 요약 1-2문장. 핵심 결론 + 주요 주의사항
  2. sections: rules_result의 각 판정 항목을 아래 형식으로 변환
     - title: 번호 + 항목명
     - status: "준수" | "미준수" | "주의"
     - details: 핵심 수치/사실을 label-value 쌍으로 (3-5개)
     - legal_basis: retrieval_context에서 해당 항목의 관련 법령만 추출
  3. 급여명세서가 있는 경우:
     - 근로계약서 항목과 급여명세서 항목을 대조
     - 기본급, 수당, 총 지급액의 일치/불일치를 detail로 표시
     - 불일치 항목은 양쪽 금액을 병기: "계약서: X원 / 명세서: Y원 → 불일치"
     - "계약-급여 대조 결과" 섹션을 별도 추가
  4. 권고(recommendation)는 "주의" 또는 "미준수" 항목에만 포함
  5. 법률 용어는 쉬운 말로 풀어서 설명
  6. 추측이나 가정을 하지 마라. 데이터에 없으면 생략

제약 (기존 유지):
  - rules_result가 최우선 판단 근거
  - retrieval_context에 없는 법령을 인용하지 마라
  - 불확실한 정보는 "확인이 필요할 수 있습니다"로 표현
```

---

## 6. 데이터 흐름 변경

### 6.1 기존 흐름

```
Rules Result + User Question
    ↓
Generation Engine → Retrieval Client (context-for-report)
    ↓
Retrieval Engine → 법령 검색
    ↓
Prompt Service → 6개 섹션 마크다운 프롬프트
    ↓
Gemini API → 마크다운 자유 서술
    ↓
GenerateResponse { answer: "## 1. 사안 요약\n...", citations: [...] }
```

### 6.2 신규 흐름

```
Rules Result + (선택) Payslip Comparison
    ↓
Gateway Orchestrator → Retrieval Client (context-for-report)
    ↓
Retrieval Engine → 법령 검색
    ↓
Generation Engine:
  입력 조합:
    - rules_result (규칙 판정)
    - retrieval_context (법령 근거)
    - payslip_comparison (계약 vs 급여 대조 결과, 있을 때만)
    ↓
  Prompt Service → JSON 스키마 강제 프롬프트
    ↓
  Gemma3 API (/v1/chat/completions) → JSON 문자열
    ↓
  후처리: JSON 파싱 + 스키마 검증 + 실패 시 재시도
    ↓
LaborContractAnalyzeResponse {
  summary: "전반적으로...",
  sections: [
    { title, status, details, legal_basis },  ← 기본 분석
    { title: "계약-급여 대조 결과", ... }      ← 급여명세서 있을 때만
  ],
  masked_files: [...]
}
```

---

## 7. Rules Engine의 대조 분석 역할

**급여명세서 대조는 Rules Engine이 수행하고, Generation Engine은 결과를 받아 리포트로 변환한다.**

### 7.1 Rules Engine 대조 로직

```python
# rules_engine/app/engine/labor_rules.py

def compare_contract_payslip(contract: ContractInfo, payslip: PayslipInfo) -> ComparisonResult:
    mismatches = []

    # 기본급 대조
    if contract.base_salary != payslip.base_salary:
        mismatches.append({
            "item": "기본급",
            "contract_value": contract.base_salary,
            "payslip_value": payslip.base_salary,
            "difference": contract.base_salary - payslip.base_salary,
            "severity": "미준수" if contract.base_salary > payslip.base_salary else "주의"
        })

    # 수당 항목 대조
    for name, amount in contract.allowances.items():
        payslip_amount = payslip.allowances.get(name, 0)
        if amount != payslip_amount:
            mismatches.append({
                "item": name,
                "contract_value": amount,
                "payslip_value": payslip_amount,
                "difference": amount - payslip_amount,
                "severity": "미준수" if amount > payslip_amount else "주의"
            })

    # 연장/야간/휴일 수당 지급 여부
    # (계약서에 연장근로 조항이 있는데 수당이 0원이면 → 주의)

    return ComparisonResult(
        has_payslip=True,
        matches=[...],
        mismatches=mismatches,
        overall_status="준수" if not mismatches else max severity
    )
```

### 7.2 Generation Engine이 받는 입력

```python
# Generation Engine에 전달되는 데이터
{
    "rules_result": {
        "sections": [...],          # 기본 규칙 판정 (최저임금, 52시간 등)
        "risk_flags": [...],
        "comparison": {             # 급여명세서 있을 때만 포함
            "has_payslip": True,
            "mismatches": [
                { "item": "식대", "contract_value": 300000, "payslip_value": 200000, ... }
            ],
            "overall_status": "주의"
        }
    },
    "retrieval_context": { ... }
}
```

---

## 8. 변환 매핑 (기존 → 신규)

| 기존 (GenerateResponse) | 신규 (LaborContractAnalyzeResponse) | 변환 방식 |
|------------------------|--------------------------------------|-----------|
| `answer` 중 "5. 결론" | `summary` | LLM이 1-2문장으로 요약 |
| `answer` 중 "2. 핵심 판단" | `sections[].status` + `details` | LLM이 label-value 구조로 변환 |
| `answer` 중 "3. 법령 근거" | `sections[].legal_basis.laws` | retrieval 결과에서 직접 매핑 |
| `citations[]` | `sections[].legal_basis.laws` | 섹션별로 분배 (별도 목록 X) |
| `answer` 중 "4. 추가 확인" | `details[].recommendation` | 해당 섹션의 detail에 포함 |
| (없음) | "계약-급여 대조 결과" 섹션 | Rules Engine 대조 결과 → LLM 변환 |
| `disclaimer` | 프론트 고정 문구 | 백엔드에서 보내지 않음 |
| (없음) | `masked_files` | Gateway에서 마스킹 후 추가 |

---

## 9. LLM 호출 변경

### 9.1 기존 (Gemini SDK)

```python
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=prompt,
    config={"temperature": 0.0, "top_p": 0.95}
)
return response.text
```

### 9.2 신규 (Gemma3 OpenAI 호환)

```python
async def generate(messages: list[dict]) -> dict:
    async with httpx.AsyncClient(timeout=120.0) as client:
        response = await client.post(
            "http://211.233.14.18:8000/v1/chat/completions",
            json={
                "model": "gemma3",
                "messages": messages,
                "temperature": 0.2,
                "max_tokens": 4000,
            }
        )
    content = response.json()["choices"][0]["message"]["content"]
    return parse_llm_json(content)
```

### 9.3 JSON 파싱 안전장치

```python
def parse_llm_json(raw: str) -> dict:
    # 1차: 직접 파싱
    # 2차: 코드블록 제거 후 파싱
    # 3차: 첫 { ~ 마지막 } 추출
    # 실패 시 재시도 요청
```

---

## 10. 기존 레퍼런스에서 유지하는 것

| 항목 | 설명 |
|------|------|
| **rules_result 우선 원칙** | 규칙 엔진 판정이 1차 근거, RAG는 보충 |
| **환각 방지 프롬프트** | "데이터에 없는 사실을 만들지 마라" |
| **인용 필터링** | 저가치 조항 필터 (목적, 정의, 부칙 등) |
| **인용 스코어링** | 주요 법령/조항에 가중치 부여 |
| **retrieval 호출 구조** | `context-for-report`로 다중 토픽 검색 |
| **쿼리 정규화/중복제거** | 검색 쿼리 전처리 로직 |
