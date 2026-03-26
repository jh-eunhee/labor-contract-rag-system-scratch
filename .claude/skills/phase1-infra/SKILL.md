---
name: phase1-infra
description: "Phase 1: Docker 환경 구성, PostgreSQL+pgvector 셋업, 프로젝트 구조 생성, 용어 정리. RAG 시스템 구축의 첫 단계."
---

# Phase 1: 인프라 및 환경 셋업

## 목표

Docker Compose 기반 개발 환경을 구성하고, 4개 FastAPI 서비스의 기본 뼈대를 만든다.

## 사전 조건

- Docker / Docker Compose 설치됨
- PLAN.md 검토 완료

## 작업 순서

### Step 1: 용어 및 구조 정리

프로젝트 루트에 아래 용어를 기반으로 작업한다:

| 용어 | 설명 |
|------|------|
| Gateway | 프론트엔드와 통신하는 단일 진입점 (Port 8080) |
| Retrieval Engine | 벡터 DB 기반 법령/KOSHA 검색 서비스 (Port 8000) |
| Generation Engine | Gemma3 LLM으로 리포트 생성 (Port 8001) |
| Rules Engine | 임금 계산, 근로시간 규칙 판정 (Port 8002) |
| pgvector | PostgreSQL 벡터 유사도 검색 확장 |
| 하이브리드 검색 | 벡터 검색 + 키워드 검색 조합 |
| SSE | Server-Sent Events, 진행률 스트리밍 |
| Job | 비동기 분석 작업 단위 (UUID) |

### Step 2: Docker Compose 구성

`docker-compose.yml` 생성:

```yaml
services:
  db:
    image: pgvector/pgvector:pg16
    ports: ["5432:5432"]
    environment:
      POSTGRES_DB: hello_rag
      POSTGRES_USER: rag_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rag_user -d hello_rag"]
      interval: 5s
      retries: 5

  retrieval-engine:
    build: ./retrieval_engine
    ports: ["8000:8000"]
    depends_on:
      db: { condition: service_healthy }
    env_file: .env

  generation-engine:
    build: ./generation_engine
    ports: ["8001:8001"]
    env_file: .env

  rules-engine:
    build: ./rules_engine
    ports: ["8002:8002"]
    env_file: .env

  gateway:
    build: ./gateway
    ports: ["8080:8080"]
    depends_on:
      - retrieval-engine
      - generation-engine
      - rules-engine
    env_file: .env

volumes:
  pgdata:
```

### Step 3: DB 초기화 스크립트

`db/init/` 디렉토리에 SQL 파일 생성:

1. **01_extensions.sql** — `CREATE EXTENSION IF NOT EXISTS vector;`
2. **02_schema.sql** — `law_documents`, `kosha_guides`, `analysis_jobs` 테이블
3. **03_seed.sql** — 기본 데이터 (빈 파일, Phase 2에서 인제스트)

스키마는 PLAN.md 섹션 6.1 참조.

### Step 4: 4개 FastAPI 서비스 뼈대

각 서비스(`retrieval_engine/`, `generation_engine/`, `rules_engine/`, `gateway/`)에 대해:

1. `Dockerfile` — Python 3.11-slim 기반
2. `requirements.txt` — fastapi, uvicorn, 서비스별 의존성
3. `main.py` — FastAPI 앱 생성, 라우터 등록
4. `app/api/routes/health.py` — `GET /health` 엔드포인트
5. `app/config.py` — 환경변수 로딩 (pydantic-settings)
6. `app/models/schemas.py` — 빈 스키마 파일

### Step 5: 환경변수

`.env.example` 생성 (PLAN.md 섹션 10 참조), `.env`는 `.gitignore`에 추가.

### Step 6: 검증

```bash
docker compose up --build
# 모든 서비스 헬스체크 통과 확인
curl http://localhost:8000/health  # retrieval
curl http://localhost:8001/health  # generation
curl http://localhost:8002/health  # rules
curl http://localhost:8080/health  # gateway
```

## 완료 기준

- [ ] `docker compose up`으로 5개 컨테이너(db + 4 서비스) 모두 기동
- [ ] 각 서비스 `/health` 응답 200 OK
- [ ] PostgreSQL에 pgvector 확장 활성화, 테이블 3개 생성 확인
- [ ] `.env.example` 존재
- [ ] `.gitignore`에 `.env`, `.venv`, `__pycache__` 포함

## 참고

- `basic-labor-standards-self-Inspection/docker-compose.yml` 구조 참고
- `basic-labor-standards-self-Inspection/retrieval_engine/init/` DB 초기화 참고
