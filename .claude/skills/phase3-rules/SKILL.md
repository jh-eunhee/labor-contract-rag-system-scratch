---
name: phase3-rules
description: "Phase 3: Rules Engine 구현. 임금 계산(통상시급, 최저임금), 근로시간 규칙, 포괄임금 감지, 리스크 탐지. 규칙 기반 분석 서비스."
---

# Phase 3: Rules Engine (AI 솔루션 - 규칙)

## 목표

근로계약서와 임금명세서를 규칙 기반으로 분석하여 임금 준수 여부, 근로시간 위반, 리스크를 판정한다.

## 사전 조건

- Phase 1 완료 (Docker 환경)

## 작업 순서

### Step 1: 데이터 모델 정의

`app/models/schemas.py`:

```python
class ContractInfo(BaseModel):
    contract_type: str           # 정규직, 계약직, 일용직 등
    wage_type: str               # 월급, 일급, 시급
    base_salary: int             # 기본급
    work_hours_per_week: float   # 주 소정근로시간
    work_days_per_week: int      # 주 소정근로일수
    contract_start: date
    contract_end: date | None
    allowances: dict[str, int]   # 수당 항목 (식대, 교통비 등)
    special_clauses: list[str]   # 특약 조항 텍스트

class PayslipInfo(BaseModel):
    total_payment: int           # 총 지급액
    base_salary: int             # 기본급
    overtime_pay: int            # 연장근로수당
    night_pay: int               # 야간근로수당
    holiday_pay: int             # 휴일근로수당
    deductions: dict[str, int]   # 공제 항목

class AnalyzeRequest(BaseModel):
    contract: ContractInfo
    payslip: PayslipInfo | None
    assumptions: dict[str, bool] # 사용자 체크박스 (주휴수당 포함 여부 등)

class RulesResult(BaseModel):
    sections: list[SectionResult]
    risk_flags: list[str]

class SectionResult(BaseModel):
    title: str
    status: str                  # "준수" | "미준수" | "주의"
    details: list[DetailItem]

class DetailItem(BaseModel):
    label: str
    value: str
    recommendation: str | None
```

### Step 2: 최저임금 테이블

`app/data/minimum_wage_table.py`:

- 연도별 최저임금 데이터 (2020~2026)
- 월급/일급/시급 환산 로직
- 참고: `basic-labor-standards-self-Inspection/rules_engine/app/data/minimum_wage_table.py`

### Step 3: 임금 계산기

`app/engine/wage_calculator.py`:

- **통상시급 계산**: 기본급 ÷ 월 소정근로시간 (209시간 = 주40시간 × 4.345주)
- **최저임금 비교**: 통상시급 vs 해당 연도 최저임금
- **수당 산정**: 연장(1.5배), 야간(1.5배), 휴일(1.5배) 근로수당
- **주휴수당**: 소정근로 개근 시 유급 휴일 수당
- 참고: `basic-labor-standards-self-Inspection/rules_engine/app/engine/labor_rules.py`

### Step 4: 근로 규칙 판정

`app/engine/labor_rules.py`:

1. **최저임금 준수 여부** — 통상시급 ≥ 법정 최저임금
2. **주 52시간제** — 소정 40시간 + 연장 12시간 한도
3. **휴게시간** — 4시간당 30분, 8시간당 1시간
4. **주휴일** — 주 15시간 이상 근로 시 유급 주휴일
5. **포괄임금 감지** — 연장/야간/휴일 수당이 기본급에 포함된 구조
6. **3.3% 프리랜서 감지** — 사업소득 원천징수 구조
7. **건설 공수 감지** — 일용직 공수 계산 구조

각 항목에 대해 `SectionResult` (title, status, details) 반환.

### Step 5: API 라우트

`app/api/routes/analyze.py`:

```python
POST /analyze
  Request:  AnalyzeRequest
  Response: RulesResult
```

- ContractInfo + PayslipInfo 수신
- 모든 규칙 순회 → sections 리스트 생성
- risk_flags (포괄임금, 3.3% 등) 반환

### Step 6: 테스트 (TDD)

```python
# tests/unit/test_wage_calculator.py
- 통상시급 계산 정확도
- 최저임금 비교 (준수/미준수 케이스)
- 수당 계산 (연장, 야간, 휴일)

# tests/unit/test_labor_rules.py
- 주 52시간 초과 감지
- 포괄임금 감지
- 각 status ("준수"/"미준수"/"주의") 반환 확인

# tests/integration/test_analyze_api.py
- 정상 계약서 → 전체 "준수"
- 최저임금 위반 계약서 → "미준수" 포함
- 포괄임금 의심 → "주의" 포함
```

## 완료 기준

- [ ] `POST /analyze` — ContractInfo 입력 시 SectionResult 배열 반환
- [ ] 최저임금 준수/위반 정확히 판정
- [ ] 통상시급 계산 정확 (월급 ÷ 209시간)
- [ ] 주 52시간 초과 감지
- [ ] 포괄임금/3.3% 리스크 플래그 감지
- [ ] 단위 테스트 80%+ 커버리지

## 참고

- `basic-labor-standards-self-Inspection/rules_engine/` 전체 구조 이식
- 임금 계산 핵심: 월 소정근로시간 = 주 소정근로시간 × 4.345
