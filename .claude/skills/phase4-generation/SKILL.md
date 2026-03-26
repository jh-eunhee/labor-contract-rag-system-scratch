---
name: phase4-generation
description: "Phase 4: Generation Engine 구현. Gemma3 LLM으로 근로계약서 분석 리포트 및 산재위험 평가 리포트 생성. 프롬프트 엔지니어링."
---

# Phase 4: Generation Engine (AI 솔루션 - 생성)

## 목표

Rules Engine 결과 + Retrieval Engine 컨텍스트를 조합하여 Gemma3 LLM으로 최종 분석 리포트를 생성한다.

## 사전 조건

- Phase 2 (Retrieval Engine) 완료
- Phase 3 (Rules Engine) 완료
- Gemma3 API 접근 가능: `http://211.233.14.18:8000/`

## 작업 순서

### Step 1: LLM 서비스

`app/services/llm_service.py`:

- **Gemma3 27B API** 호출 (`/v1/chat/completions`)
- OpenAI 호환 API 형식
- temperature: 0.2 (결정적 출력)
- max_tokens: 4000
- 에러 핸들링: context length 초과, rate limit, 연결 오류

```python
GEMMA3_API_URL = "http://211.233.14.18:8000"

async def generate(messages: list[dict], temperature: float = 0.2) -> str:
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{GEMMA3_API_URL}/v1/chat/completions",
            json={
                "model": "gemma3",
                "messages": messages,
                "temperature": temperature,
                "max_tokens": 4000,
            },
            timeout=120.0,
        )
    return response.json()["choices"][0]["message"]["content"]
```

### Step 2: Retrieval Client

`app/clients/retrieval_client.py`:

- Retrieval Engine(`http://retrieval-engine:8000`) 호출
- `POST /retrieval/search` — 단일 쿼리 검색
- `POST /retrieval/context-for-report` — 다중 토픽 검색
- 참고: `basic-labor-standards-self-Inspection/generation_engine/app/clients/retrieval_client.py`

### Step 3: 프롬프트 엔지니어링 - 근로계약서

`prompts/labor_contract.py`:

Rules 결과 + 법령 RAG 컨텍스트를 받아 아래 형식의 구조화된 JSON을 출력하도록 프롬프트 설계:

```
시스템 프롬프트:
  - 너는 노동법 전문가 AI 어시스턴트이다
  - 규칙 엔진 분석 결과와 관련 법령을 기반으로 리포트를 생성한다
  - 반드시 아래 JSON 스키마에 맞춰 응답한다

유저 프롬프트:
  - [규칙 엔진 결과] sections + risk_flags
  - [관련 법령] retrieval 검색 결과
  - 지시사항:
    1. summary: 전체 요약 1-2문장
    2. sections: 각 항목별 title, status, details, legal_basis
    3. 법적근거는 검색된 법령에서만 인용
    4. recommendation은 "주의" 또는 "미준수" 항목에만 포함
```

출력 JSON은 PLAN.md의 `LaborContractAnalyzeResponse` 스키마를 따른다.

### Step 4: 프롬프트 엔지니어링 - 산재위험

`prompts/hazard_assessment.py`:

이미지 캡셔닝 결과 + KOSHA RAG 컨텍스트를 받아 위험성 평가:

```
시스템 프롬프트:
  - 너는 산업안전보건 전문가 AI 어시스턴트이다
  - 현장 사진 분석 결과와 KOSHA 가이드를 기반으로 위험성을 평가한다

유저 프롬프트:
  - [이미지 캡셔닝 결과] 위험 요소 설명
  - [KOSHA 가이드] 관련 안전 기준
  - 지시사항:
    1. 위험 요소별 risk_level 판정 (매우 위험/위험/주의/양호)
    2. action_item: KOSHA 가이드 기반 구체적 조치사항
    3. action_code: 해당 KOSHA GUIDE 코드
```

출력 JSON은 PLAN.md의 `HazardAnalysisResponse` 스키마를 따른다.

### Step 5: 생성 서비스 오케스트레이션

`app/services/generation_service.py`:

```python
async def generate_labor_report(rules_result, contract_info):
    # 1. Retrieval Engine에서 관련 법령 검색
    topics = [section.title for section in rules_result.sections]
    legal_context = await retrieval_client.context_for_report(topics, "labor")

    # 2. 프롬프트 조합
    prompt = build_labor_prompt(rules_result, legal_context)

    # 3. Gemma3 LLM 호출
    raw_response = await llm_service.generate(prompt)

    # 4. JSON 파싱 및 검증
    return parse_labor_response(raw_response)

async def generate_hazard_report(caption_result):
    # 1. KOSHA 가이드 검색
    context = await retrieval_client.search(caption_result, "kosha")

    # 2. 프롬프트 조합
    prompt = build_hazard_prompt(caption_result, context)

    # 3. Gemma3 LLM 호출
    raw_response = await llm_service.generate(prompt)

    # 4. JSON 파싱 및 검증
    return parse_hazard_response(raw_response)
```

### Step 6: API 라우트

`app/api/routes/generation.py`:

```python
POST /generation/labor-report
  Request:  { rules_result: RulesResult, contract_info: ContractInfo }
  Response: LaborContractAnalyzeResponse

POST /generation/hazard-report
  Request:  { captions: [{ image_index, description }] }
  Response: HazardAnalysisResponse[]
```

### Step 7: 테스트

```python
# tests/unit/test_prompt_service.py
- 프롬프트 조합 정상 동작
- JSON 출력 파싱 정확도

# tests/unit/test_llm_service.py
- Gemma3 API 호출 (mock)
- 에러 핸들링 (타임아웃, context length)

# tests/integration/test_generation_api.py
- Rules 결과 입력 → LaborContractAnalyzeResponse 스키마 준수
- 캡셔닝 결과 입력 → HazardAnalysisResponse 스키마 준수
```

## 완료 기준

- [ ] Gemma3 API 연동 정상 (`http://211.233.14.18:8000/v1/chat/completions`)
- [ ] `POST /generation/labor-report` — 구조화 JSON 리포트 생성
- [ ] `POST /generation/hazard-report` — 위험성 평가 JSON 생성
- [ ] Retrieval Engine 연동 (법령/KOSHA 컨텍스트 주입)
- [ ] 출력 JSON이 프론트엔드 스키마와 일치
- [ ] LLM 응답 JSON 파싱 실패 시 재시도 로직
- [ ] 단위 테스트 80%+ 커버리지

## 참고

- `basic-labor-standards-self-Inspection/generation_engine/` 구조 참고
- Gemma3 API는 OpenAI 호환 형식 (`/v1/chat/completions`)
- 프론트 기존 Gemma3 호출: `ax-frontend/src/app/api/labor-contract/analyze/route.ts` 프롬프트 참고
