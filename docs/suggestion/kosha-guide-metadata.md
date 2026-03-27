# KOSHA GUIDE 벡터화 시 출처 메타데이터 추가 안내

## 핵심 원칙

> **"임베딩에 메타데이터를 넣는다" → 사실 아님**

- 임베딩은 **순수 텍스트만 벡터화**
- 메타데이터는 **별도 저장**하고, 검색 후 필터/출처로 사용

```
[text] ──> embedding vector (의미 검색 담당)
        └─ metadata (별도 저장 → 필터링, 출처 표시, 재인덱싱 담당)
```

### 역할 분리

| 역할 | 담당 |
|------|------|
| 의미 검색 | embedding |
| 필터링 | metadata |
| 출처 표시 | metadata |
| 재인덱싱 | metadata |

---

## 흔한 실수 3가지

### 1. 메타데이터를 텍스트에 섞어서 임베딩

```python
# WRONG
text = f"[H-68-2022 p.12] {content}"
embedding = embed(text)
```

- embedding 품질 저하 (의미 없는 토큰이 noise)
- 모델이 "page 12"를 의미로 학습함

### 2. embedding API에 metadata 파라미터를 넣으려 함

```python
# WRONG - 대부분의 embedding 모델은 metadata 파라미터가 없음
embedding = embed(text, metadata=...)
```

### 3. metadata 없이 embedding만 저장

나중에 "이 문장 어디서 나온 거냐?", "이 PDF만 재인덱싱", "특정 카테고리만 검색" 전부 불가능.

### 예외: 문맥 의미 강화용 제목 포함은 OK

```text
"밀폐공간 질식재해 예방 가이드: 산소 농도는 18% 이상 유지해야 한다."
```

제목은 의미 정보이므로 임베딩 텍스트에 포함해도 괜찮다.

---

## 기술 스택

| 항목 | 선택 |
|------|------|
| 벡터 스토어 | PostgreSQL + pgvector |
| 임베딩 모델 | Google Gemini `gemini-embedding-001` (768차원) |
| 임베딩 SDK | `google-genai` |
| 인증 | `GOOGLE_API_KEY` 환경변수 |

- 레퍼런스: `basic-labor-standards-self-Inspection/retrieval_engine/app/services/embedding_service.py`

---

## 스키마 설계

### 권장: JSONB 메타데이터 컬럼 방식

```sql
CREATE TABLE kosha_guides (
    id SERIAL PRIMARY KEY,
    guide_code VARCHAR(50) UNIQUE,
    title VARCHAR(200),
    content TEXT,
    risk_category VARCHAR(100),
    metadata JSONB DEFAULT '{}',
    embedding vector(768),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_kosha_metadata ON kosha_guides USING GIN (metadata);
```

### 저장 데이터 예시

```json
{
  "content": "밀폐공간에서 작업 시 산소 농도를 측정해야 한다.",
  "embedding": [0.123, -0.532, ...],
  "metadata": {
    "source_document": "KOSHA_GUIDE_H-68-2022.pdf",
    "page_number": 12,
    "section": "제3장 예방대책",
    "source_row": 1,
    "ingested_at": "2026-03-27T10:00:00"
  }
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `source_document` | string | 원본 PDF/문서 파일명 |
| `page_number` | integer | 원본 문서 내 페이지 번호 |
| `section` | string | 원본 문서 내 섹션/장 제목 |
| `source_row` | integer | CSV 적재 시 행 번호 |
| `ingested_at` | string | 적재 일시 (ISO 8601) |

---

## CSV 파일 구조

출처 정보를 CSV에 미리 포함시킨다.

```csv
guide_code,title,content,risk_category,original_file,page_number,section
H-68-2022,밀폐공간 질식재해 예방,밀폐공간에서 작업 시...,질식위험,KOSHA_GUIDE_H-68-2022.pdf,12,제3장 예방대책
H-01-2023,낙상 방지 안내,바닥에 고인 액체를...,낙상위험,KOSHA_GUIDE_H-01-2023.pdf,5,제2장 안전조치
```

---

## 구현 코드

### EmbeddingService (레퍼런스 기반)

순수 텍스트만 받아서 벡터만 반환한다. 메타데이터는 관여하지 않는다.

```python
# retrieval_engine/app/services/embedding_service.py

from __future__ import annotations
from typing import Any
from app.core.config import settings


class EmbeddingService:
    def __init__(self) -> None:
        self._client = None

    def is_available(self) -> bool:
        if not settings.GOOGLE_API_KEY:
            return False
        try:
            from google import genai  # noqa: F401
        except ImportError:
            return False
        return True

    def _get_client(self) -> Any:
        if self._client is not None:
            return self._client
        if not settings.GOOGLE_API_KEY:
            raise RuntimeError("GOOGLE_API_KEY is not configured.")
        try:
            from google import genai
        except ImportError as exc:
            raise RuntimeError("google-genai package is not installed.") from exc

        self._client = genai.Client(api_key=settings.GOOGLE_API_KEY)
        return self._client

    def embed_text(self, text: str) -> list[float]:
        """순수 텍스트 -> 768차원 벡터. 메타데이터는 여기 들어오지 않는다."""
        client = self._get_client()
        response = client.models.embed_content(
            model=settings.GEMINI_EMBED_MODEL,
            contents=text,
        )
        if not getattr(response, "embeddings", None):
            raise RuntimeError("Embedding response is empty.")

        embedding = response.embeddings[0].values
        if len(embedding) != settings.VECTOR_DIMENSION:
            raise RuntimeError(
                f"Dimension mismatch: expected={settings.VECTOR_DIMENSION}, got={len(embedding)}"
            )
        return list(embedding)

    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        """대량 인제스트용 배치 임베딩."""
        return [self.embed_text(text) for text in texts]
```

### IngestService: 임베딩(텍스트) + 메타데이터(별도) 분리 저장

```python
# retrieval_engine/app/services/ingest_service.py

import csv
from pathlib import Path
from datetime import datetime

from app.services.embedding_service import EmbeddingService
from app.repositories.vector_repository import VectorRepository


class IngestService:
    def __init__(
        self,
        embedding_service: EmbeddingService,
        vector_repository: VectorRepository,
    ):
        self.embedding_service = embedding_service
        self.vector_repository = vector_repository

    async def ingest_kosha_csv(self, csv_path: str) -> int:
        file_path = Path(csv_path)
        rows = self._read_csv(file_path)

        # 임베딩: 순수 content 텍스트만 벡터화
        texts = [row["content"] for row in rows]
        embeddings = self.embedding_service.embed_batch(texts)

        # 메타데이터: 임베딩과 별도로 구성
        records = [
            {
                "guide_code": row["guide_code"],
                "title": row["title"],
                "content": row["content"],
                "risk_category": row["risk_category"],
                "embedding": embedding,
                "metadata": {
                    "source_document": row.get("original_file"),
                    "page_number": int(row["page_number"]) if row.get("page_number") else None,
                    "section": row.get("section"),
                    "source_row": idx + 1,
                    "ingested_at": datetime.now().isoformat(),
                },
            }
            for idx, (row, embedding) in enumerate(zip(rows, embeddings))
        ]

        inserted = await self.vector_repository.bulk_insert_kosha(records)
        return inserted

    def _read_csv(self, file_path: Path) -> list[dict]:
        with open(file_path, encoding="utf-8") as f:
            return list(csv.DictReader(f))
```

### VectorRepository: 저장 + 검색 (메타데이터 활용)

```python
# retrieval_engine/app/repositories/vector_repository.py

import json
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


class VectorRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def bulk_insert_kosha(self, records: list[dict]) -> int:
        query = text("""
            INSERT INTO kosha_guides
                (guide_code, title, content, risk_category, metadata, embedding)
            VALUES
                (:guide_code, :title, :content, :risk_category,
                 :metadata::jsonb, :embedding::vector)
            ON CONFLICT (guide_code) DO NOTHING
        """)

        for record in records:
            await self.session.execute(query, {
                **record,
                "metadata": json.dumps(record["metadata"], ensure_ascii=False),
                "embedding": str(record["embedding"]),
            })

        await self.session.commit()
        return len(records)

    async def search_kosha(
        self, query_embedding: list[float], top_k: int = 5
    ) -> list[dict]:
        """벡터 유사도로 검색 후, 메타데이터에서 출처 정보를 추출한다."""
        query = text("""
            SELECT
                guide_code, title, content, risk_category, metadata,
                1 - (embedding <=> :query::vector) AS score
            FROM kosha_guides
            ORDER BY embedding <=> :query::vector
            LIMIT :top_k
        """)

        result = await self.session.execute(query, {
            "query": str(query_embedding),
            "top_k": top_k,
        })

        return [
            {
                "guide_code": row.guide_code,
                "title": row.title,
                "content": row.content,
                "risk_category": row.risk_category,
                "score": float(row.score),
                "citation": {
                    "guide_code": row.guide_code,
                    "title": row.title,
                    "source_document": row.metadata.get("source_document"),
                    "page_number": row.metadata.get("page_number"),
                    "section": row.metadata.get("section"),
                },
            }
            for row in result.fetchall()
        ]

    async def search_kosha_with_filter(
        self, query_embedding: list[float], source_document: str, top_k: int = 5
    ) -> list[dict]:
        """메타데이터 필터링 + 벡터 유사도 검색 조합."""
        query = text("""
            SELECT
                guide_code, title, content, risk_category, metadata,
                1 - (embedding <=> :query::vector) AS score
            FROM kosha_guides
            WHERE metadata->>'source_document' = :source_doc
            ORDER BY embedding <=> :query::vector
            LIMIT :top_k
        """)

        result = await self.session.execute(query, {
            "query": str(query_embedding),
            "source_doc": source_document,
            "top_k": top_k,
        })

        return [
            {
                "guide_code": row.guide_code,
                "title": row.title,
                "content": row.content,
                "score": float(row.score),
                "citation": {
                    "source_document": row.metadata.get("source_document"),
                    "page_number": row.metadata.get("page_number"),
                    "section": row.metadata.get("section"),
                },
            }
            for row in result.fetchall()
        ]
```

---

## 프론트엔드 활용

검색 결과의 `citation` 객체를 활용하여 출처를 표시한다.

```
근거: KOSHA GUIDE H-68-2022 (밀폐공간 질식재해 예방)
      출처: KOSHA_GUIDE_H-68-2022.pdf, p.12, 제3장 예방대책
```

---

## 참고

- JSONB 방식은 스키마 변경 없이 메타데이터 필드를 자유롭게 확장할 수 있다.
- GIN 인덱스를 통해 `metadata->>'source_document'` 같은 조건 검색도 가능하다.
- 법령 데이터(`law_documents`)에도 동일한 패턴을 적용할 수 있다.
- 레퍼런스 프로젝트의 `EmbeddingService`는 이미 순수 텍스트만 받는 정석 구조를 따르고 있다.
