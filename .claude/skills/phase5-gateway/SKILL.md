---
name: phase5-gateway
description: "Phase 5: API Gateway 구현. 프론트엔드 API 라우팅, 비동기 파이프라인 오케스트레이션, SSE 진행률 스트리밍, OCR, 이미지 캡셔닝, PDF 마스킹."
---

# Phase 5: Gateway + 오케스트레이션 (백엔드 API)

## 목표

프론트엔드와 통신하는 단일 API Gateway를 구축하고, 3개 내부 엔진을 오케스트레이션하는 비동기 파이프라인을 구현한다.

## 사전 조건

- Phase 2~4 완료 (3개 엔진 동작)
- API 명세: PLAN.md 섹션 3 참조

## 작업 순서

### Step 1: API 라우트 - 기초노동질서

`app/api/routes/labor_contract.py`:

```python
POST /api/labor-contract/upload
  - 파일 수신 (contract_file, paystub_file?)
  - Job 생성 (UUID) → analysis_jobs 테이블 INSERT
  - 비동기 분석 파이프라인 시작 (BackgroundTasks)
  - Response: { "job_id": "uuid" }  (202 Accepted)

POST /api/labor-contract/subscribe/{job_id}
  - SSE 스트리밍 연결
  - analysis_jobs.status 변경 시 이벤트 전송
  - Response: SSE stream { "status": "OCR_PROCESSING" }

GET /api/labor-contract/{job_id}/result
  - analysis_jobs.result (JSONB) 반환
  - Response: LaborContractAnalyzeResponse

GET /api/labor-contract/masked/{fileId}
  - 마스킹된 PDF/이미지 파일 반환
  - Response: FileResponse (binary)
```

### Step 2: API 라우트 - 산재위험요소

`app/api/routes/hazard_analysis.py`:

```python
POST /api/hazard-analysis/upload
  - 이미지 파일(들) 수신
  - Job 생성 (UUID)
  - 비동기 분석 파이프라인 시작
  - Response: { "jobId": "uuid" }  (202 Accepted)

POST /api/hazard-analysis/subscribe/{job_id}
  - SSE 스트리밍 연결
  - Response: SSE stream { "status": "VECTOR_SEARCH" }

GET /api/hazard-analysis/{job_id}/result
  - Response: HazardAnalysisResponse[]

GET /api/hazard-analysis/{fileId}
  - 이미지 파일 반환
  - Response: FileResponse (binary)
```

### Step 3: OCR 서비스

`app/services/ocr_service.py`:

- PDF/이미지 → 텍스트 추출
- PyMuPDF(fitz)로 PDF 텍스트 직접 추출 (선 처리)
- 텍스트 없는 스캔 PDF → Gemma3 비전 모델로 이미지 캡셔닝
- 추출된 텍스트 → ContractInfo/PayslipInfo 구조화

```python
async def extract_contract_info(file: UploadFile) -> ContractInfo:
    # 1. PDF → 텍스트 추출 (fitz)
    # 2. 텍스트가 없으면 → 이미지 변환 → Gemma3 캡셔닝
    # 3. 텍스트 → 구조화 (정규표현식 + LLM 보조)
    # 4. ContractInfo 반환
```

### Step 4: 이미지 캡셔닝 서비스

`app/services/image_caption.py`:

- 현장 사진 → Gemma3 비전 모델로 위험 요소 설명 생성
- base64 인코딩 → `/v1/chat/completions` (이미지 포함)
- 프롬프트: "이 작업 현장 사진에서 산업안전보건 관점의 위험 요소를 식별하라"

```python
async def caption_hazard_image(image_bytes: bytes) -> str:
    base64_image = base64.b64encode(image_bytes).decode()
    messages = [
        {"role": "user", "content": [
            {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{base64_image}"}},
            {"type": "text", "text": HAZARD_CAPTION_PROMPT}
        ]}
    ]
    return await llm_service.generate(messages)
```

### Step 5: PDF 마스킹 서비스

`app/services/pdf_masking.py`:

- 개인정보(이름, 주민번호, 주소, 전화번호) 탐지
- PyMuPDF(fitz)로 해당 영역 마스킹 (검정 박스)
- 마스킹된 PDF 저장 → fileId로 접근 가능
- 이미지 파일의 경우 Pillow로 마스킹

### Step 6: 오케스트레이터

`app/services/orchestrator.py`:

**근로계약서 파이프라인:**
```python
async def run_labor_analysis(job_id: str, files):
    try:
        # 1단계: OCR 구조화 (20%)
        update_status(job_id, "OCR_PROCESSING")
        contract_info = await ocr_service.extract_contract_info(files.contract)
        payslip_info = await ocr_service.extract_payslip_info(files.paystub)  # optional

        # 2단계: Rules Engine 호출 (40%)
        rules_result = await rules_client.analyze(contract_info, payslip_info)

        # 3단계: Retrieval + Generation (60% → 90%)
        update_status(job_id, "VECTOR_SEARCH")
        # ... retrieval 검색

        update_status(job_id, "LLM_PROCESSING")
        report = await generation_client.labor_report(rules_result, contract_info)

        # 4단계: PDF 마스킹 (90%)
        masked_files = await pdf_masking.mask_pii(files)

        # 5단계: 완료 (100%)
        save_result(job_id, report, masked_files)
        update_status(job_id, "COMPLETED")

    except Exception as e:
        update_status(job_id, "ERROR", error=str(e))
```

**산재위험 파이프라인:**
```python
async def run_hazard_analysis(job_id: str, images):
    try:
        # 1단계: 이미지 캡셔닝 (30%)
        update_status(job_id, "OCR_PROCESSING")
        captions = [await image_caption.caption(img) for img in images]

        # 2단계: KOSHA Guide 검색 (60%)
        update_status(job_id, "VECTOR_SEARCH")
        # ... retrieval 검색

        # 3단계: 위험성 평가 리포트 (90%)
        update_status(job_id, "LLM_PROCESSING")
        report = await generation_client.hazard_report(captions)

        # 4단계: 완료 (100%)
        save_result(job_id, report)
        update_status(job_id, "COMPLETED")

    except Exception as e:
        update_status(job_id, "ERROR", error=str(e))
```

### Step 7: SSE 스트리밍

`app/services/sse_manager.py`:

- `sse-starlette` 라이브러리 사용
- Job별 상태 변경 이벤트 발행
- 클라이언트 연결 관리 (연결/해제)
- 타임아웃 처리 (5분)

### Step 8: 내부 엔진 클라이언트

`app/clients/`:

```python
# rules_client.py    → http://rules-engine:8002/analyze
# retrieval_client.py → http://retrieval-engine:8000/retrieval/*
# generation_client.py → http://generation-engine:8001/generation/*
```

- httpx.AsyncClient 사용
- 타임아웃 설정 (각 엔진별)
- 에러 핸들링 및 재시도

### Step 9: 파일 저장소

업로드/마스킹 파일 저장:
- `uploads/{job_id}/` — 원본 파일
- `masked/{file_id}/` — 마스킹 파일
- Docker volume으로 영속화

### Step 10: 테스트

```python
# tests/unit/test_orchestrator.py
# tests/unit/test_ocr_service.py
# tests/unit/test_pdf_masking.py
# tests/integration/test_labor_contract_api.py
# tests/integration/test_hazard_analysis_api.py
# tests/integration/test_sse.py
```

## 완료 기준

- [ ] `POST /api/labor-contract/upload` → 202 + job_id 반환
- [ ] `POST /api/labor-contract/subscribe/{job_id}` → SSE 이벤트 수신
- [ ] `GET /api/labor-contract/{job_id}/result` → LaborContractAnalyzeResponse 반환
- [ ] `GET /api/labor-contract/masked/{fileId}` → 마스킹 파일 반환
- [ ] 산재위험 API 4개 엔드포인트 동일하게 동작
- [ ] 비동기 파이프라인: PENDING → OCR → VECTOR → LLM → COMPLETED 순서
- [ ] 에러 발생 시 ERROR 상태 + 에러 메시지
- [ ] 전체 파이프라인 통합 테스트 통과

## 참고

- API 명세: PLAN.md 섹션 3
- 시퀀스 다이어그램: PLAN.md 섹션 2.4
- 파일 업로드: png, jpg 이미지 + pdf 지원
