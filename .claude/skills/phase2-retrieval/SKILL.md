---
name: phase2-retrieval
description: "Phase 2: Retrieval Engine 구현. Gemini 임베딩, pgvector 하이브리드 검색, 법령/KOSHA 데이터 적재. Vector DB 기반 검색 서비스."
---

# Phase 2: Retrieval Engine (AI 솔루션 - 검색)

## 목표

법령 데이터와 KOSHA 가이드를 벡터 DB에 적재하고, 하이브리드 검색(벡터 + 키워드) API를 구현한다.

## 사전 조건

- Phase 1 완료 (Docker 환경, DB 스키마)
- `GOOGLE_API_KEY` 환경변수 설정 (Gemini 임베딩용)

## 작업 순서

### Step 1: 데이터 준비

`retrieval_engine/data/` 디렉토리에 CSV 파일 배치:

1. **labor_law.csv** — 근로기준법, 최저임금법 조항
   - 참고: `basic-labor-standards-self-Inspection/retrieval_engine/data/labor_law.csv`
2. **minimum_wage_history.csv** — 연도별 최저임금
   - 참고: `basic-labor-standards-self-Inspection/retrieval_engine/data/minimum_wage_history.csv`
3. **kosha_guide.csv** — KOSHA 안전보건 가이드 (산재위험용)
   - 컬럼: guide_code, title, content, risk_category

### Step 2: 임베딩 서비스

`app/services/embedding_service.py`:

- Google Gemini `gemini-embedding-001` 모델 사용
- 텍스트 → 768차원 벡터 변환
- 배치 임베딩 지원 (대량 인제스트용)
- 참고: `basic-labor-standards-self-Inspection/retrieval_engine/app/services/embedding_service.py`

### Step 3: 벡터 저장소

`app/repositories/vector_repository.py`:

- pgvector 연동 (SQLAlchemy + asyncpg)
- 코사인 유사도 검색 (`<=>` 연산자)
- L2 거리 검색 (`<->` 연산자)
- 벡터 인덱스 생성 (IVFFlat 또는 HNSW)
- 참고: `basic-labor-standards-self-Inspection/retrieval_engine/app/repositories/vector_repository.py`

### Step 4: 하이브리드 검색 서비스

`app/services/retrieval_service.py`:

- **벡터 검색**: 쿼리 임베딩 → pgvector 유사도 검색
- **키워드 검색**: PostgreSQL full-text search (`tsvector`, `tsquery`)
- **결과 병합**: RRF(Reciprocal Rank Fusion) 또는 가중 합산
- **중복 제거**: 같은 조항 중복 방지
- 참고: `basic-labor-standards-self-Inspection/retrieval_engine/app/services/retrieval_service.py`

### Step 5: 인용 매핑 서비스

`app/services/citation_service.py`:

- 검색된 텍스트 → 법령 조항 번호 매핑
- `law_name` + `article_number` 기반 인용 정보 구성
- 참고: `basic-labor-standards-self-Inspection/retrieval_engine/app/services/citation_service.py`

### Step 6: API 라우트

`app/api/routes/retrieval.py`:

```python
POST /retrieval/search
  Request:  { query: str, top_k: int = 5, search_type: "labor" | "kosha" }
  Response: { results: [{ content, score, citation }] }

POST /retrieval/context-for-report
  Request:  { topics: [str], search_type: "labor" | "kosha" }
  Response: { contexts: [{ topic, results }] }
```

### Step 7: 데이터 인제스트

`app/services/ingest_service.py`:

- 서버 시작 시 CSV → 임베딩 → DB 적재 (`AUTO_INGEST_ON_STARTUP=true`)
- 이미 적재된 데이터는 스킵 (idempotent)
- 법령 데이터: `law_documents` 테이블
- KOSHA 데이터: `kosha_guides` 테이블

### Step 8: 테스트

```python
# tests/unit/test_embedding_service.py
# tests/unit/test_retrieval_service.py
# tests/integration/test_search_api.py
```

- 임베딩 변환 정상 동작
- 하이브리드 검색 결과 반환
- 인용 매핑 정확도
- API 엔드포인트 200 응답

## 완료 기준

- [ ] CSV 데이터 → pgvector DB 자동 인제스트
- [ ] `POST /retrieval/search` — 쿼리에 대해 관련 법령/KOSHA 결과 반환
- [ ] `POST /retrieval/context-for-report` — 다중 토픽 검색 동작
- [ ] 하이브리드 검색 (벡터 + 키워드) 동작 확인
- [ ] 인용 매핑 (법령명 + 조항번호) 정상
- [ ] 단위 테스트 80%+ 커버리지

## 참고

- `basic-labor-standards-self-Inspection/retrieval_engine/` 전체 구조 이식
- Gemini Embedding 모델: `gemini-embedding-001` (768차원)
