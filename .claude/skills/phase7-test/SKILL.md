---
name: phase7-test
description: "Phase 7: 통합 테스트, E2E 테스트, 검색 정확도 튜닝, 보안 검토. 전체 파이프라인 검증 및 최적화."
---

# Phase 7: 테스트 및 최적화

## 목표

전체 파이프라인(파일 업로드 → RAG → 리포트 생성)의 통합 테스트를 수행하고, 검색 정확도와 보안을 점검한다.

## 사전 조건

- Phase 1~6 모두 완료
- Docker Compose 전체 기동 상태

## 작업 순서

### Step 1: 통합 테스트 - 근로계약서 파이프라인

`tests/integration/test_labor_pipeline.py`:

```python
# 테스트 시나리오:
async def test_labor_contract_full_pipeline():
    # 1. 샘플 근로계약서 PDF 업로드
    response = await client.post("/api/labor-contract/upload", files=...)
    assert response.status_code == 202
    job_id = response.json()["job_id"]

    # 2. SSE 구독 → COMPLETED 대기
    async for event in sse_subscribe(f"/api/labor-contract/subscribe/{job_id}"):
        if event["status"] == "COMPLETED":
            break

    # 3. 결과 조회
    result = await client.get(f"/api/labor-contract/{job_id}/result")
    assert result.status_code == 200

    # 4. 응답 스키마 검증
    data = result.json()
    assert "summary" in data
    assert len(data["sections"]) > 0
    for section in data["sections"]:
        assert section["status"] in ["준수", "미준수", "주의"]
        assert len(section["details"]) > 0
        assert "legal_basis" in section

    # 5. 마스킹 파일 접근
    if data.get("masked_files"):
        file_id = data["masked_files"][0]["file_id"]
        file_res = await client.get(f"/api/labor-contract/masked/{file_id}")
        assert file_res.status_code == 200
```

### Step 2: 통합 테스트 - 산재위험 파이프라인

`tests/integration/test_hazard_pipeline.py`:

```python
async def test_hazard_analysis_full_pipeline():
    # 1. 현장 사진 업로드
    # 2. SSE 구독 → COMPLETED 대기
    # 3. 결과 조회
    # 4. 응답 검증
    data = result.json()
    assert isinstance(data, list)
    for item in data:
        assert "image_index" in item
        assert "risk_counts" in item
        assert "risks" in item
        for risk in item["risks"]:
            assert risk["risk_level"] in ["매우 위험", "위험", "주의", "양호"]
            assert "action_code" in risk  # KOSHA 코드
```

### Step 3: E2E 테스트 (Playwright)

`tests/e2e/`:

```typescript
// labor-contract.spec.ts
test('근로계약서 분석 전체 흐름', async ({ page }) => {
  await page.goto('/labor-contract')

  // 파일 업로드
  await page.setInputFiles('input[type="file"]', 'fixtures/sample_contract.pdf')
  await page.click('button:has-text("분석")')

  // 로딩 단계 표시 확인
  await expect(page.locator('text=근로조건 분석 중')).toBeVisible()
  await expect(page.locator('text=법령 검색 중')).toBeVisible()
  await expect(page.locator('text=리포트 생성 중')).toBeVisible()

  // 결과 화면 확인
  await expect(page.locator('text=결과 요약')).toBeVisible({ timeout: 300000 })
  await expect(page.locator('text=최저임금')).toBeVisible()

  // 법적근거 팝업
  await page.click('text=법적근거 보기')
  await expect(page.locator('.legal-basis-popup')).toBeVisible()
})

// hazard-analysis.spec.ts
test('산재위험 분석 전체 흐름', async ({ page }) => {
  await page.goto('/')
  // 이미지 업로드 → 결과 확인
})
```

### Step 4: 검색 정확도 튜닝

Retrieval Engine 검색 품질 개선:

1. **쿼리 테스트 세트 작성**
   - "최저임금 위반" → 최저임금법 제6조 반환되는지
   - "연장근로 동의" → 근로기준법 제53조 반환되는지
   - "추락 위험" → 관련 KOSHA 가이드 반환되는지

2. **파라미터 튜닝**
   - `top_k` 값 조정 (5 → 3~10 범위)
   - 벡터 vs 키워드 가중치 비율
   - 유사도 임계값 (score threshold)

3. **청크 크기 조정**
   - 법령 조항 단위 vs 문단 단위
   - KOSHA 가이드 세분화 수준

### Step 5: 성능 테스트

```python
# 응답 시간 목표:
# - 파일 업로드 → job_id: < 1초
# - 전체 파이프라인: < 5분
# - SSE 이벤트 간격: < 30초
# - 결과 조회: < 1초

async def test_response_times():
    start = time.time()
    # ... 파이프라인 실행
    assert time.time() - start < 300  # 5분 이내
```

### Step 6: 보안 검토

체크리스트:

- [ ] 파일 업로드 검증: 허용 확장자 (pdf, png, jpg), 크기 제한 (10MB)
- [ ] Job ID: UUID 형식 검증, 타인 Job 접근 방지
- [ ] SQL Injection: SQLAlchemy 파라미터 바인딩 사용
- [ ] 업로드 파일 경로 순회 공격 방지
- [ ] PII 마스킹: 주민번호, 전화번호, 주소 패턴 검출
- [ ] 에러 응답: 내부 스택 트레이스 노출 금지
- [ ] CORS 설정: 허용 origin 제한
- [ ] Rate Limiting: 업로드 API에 제한 적용
- [ ] 환경변수: `.env` 파일 gitignore 확인
- [ ] Gemma3 API URL: 내부 네트워크 접근만 허용

### Step 7: 에러 시나리오 테스트

```python
# 잘못된 파일 형식 업로드
# 빈 파일 업로드
# 존재하지 않는 job_id 조회
# Gemma3 API 타임아웃
# Retrieval Engine 다운 시 Gateway 처리
# DB 연결 실패 시 처리
```

## 완료 기준

- [ ] 근로계약서 전체 파이프라인 통합 테스트 통과
- [ ] 산재위험 전체 파이프라인 통합 테스트 통과
- [ ] E2E 테스트 (Playwright) 주요 시나리오 통과
- [ ] 검색 정확도: 주요 쿼리 10개 중 8개 이상 관련 법령 1위 반환
- [ ] 전체 파이프라인 5분 이내 완료
- [ ] 보안 체크리스트 전항목 통과
- [ ] 에러 시나리오 핸들링 정상
- [ ] 테스트 커버리지 80%+

## 참고

- E2E 테스트: `ax-frontend/playwright.config.ts` 설정 참고
- 보안 검토: `~/.claude/rules/security.md` 체크리스트 따름
