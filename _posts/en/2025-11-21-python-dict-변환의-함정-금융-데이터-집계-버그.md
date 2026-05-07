---
title: "The Pitfall of Python dict() Conversion - A Financial Data Aggregation Bug"
date: 2025-11-21
categories: [Debugging]
tags: [python, django, data-integrity]
mermaid: true
---

## Discovery

A bug was found in the loan underwriting system's logic for passing health insurance payment data to an external scoring engine. **Different values were being passed for the same application depending on the underwriting path (pre-screening vs. automated underwriting).**

```text
-- Automated underwriting (public MyData)
"HEALTH_INSURANCE": "0,0,0,0,0,0,0,0,-56460,163070,163070,0,0,0,..."

-- Pre-screening (partner pay scraping)
"HEALTH_INSURANCE": "0,154470,154470,154470,154470,...,163070,163070,..."
```

The original data stored in the DB was identical. **The data was diverging in the delivery logic.**

---

## Root Cause Analysis

### Bug 1: Value overwriting during dict() conversion

```python
payments = dict(
    self.health_insurance_payments.values_list(
        "payment_month", "employee_health_notice_amount",
    )
)
```

The problem lies in `dict()`.

```mermaid
flowchart TD
    A["QuerySet.values_list() 결과"] --> B["[('2025-01', 150000),<br>('2025-01', 163070),<br>('2025-02', 154470)]"]
    B --> C["dict() 변환"]
    C --> D["{'2025-01': 163070,<br>'2025-02': 154470}"]
    C --> E["⚠️ '2025-01'의 150000이<br>163070으로 덮어써짐"]
    style E fill:#ff6b6b,stroke:#333,color:#fff
```

When `dict()` receives tuples with duplicate keys, **the later value overwrites the earlier one.** In health insurance data, multiple records can exist for the same month (payment_month):

- Concurrent employment (paying at two workplaces simultaneously)
- Job change (refund from previous employer + payment at new employer)
- Separate data per underwriting path (pre-screening/automated)

```python
# How dict() works
dict([("a", 1), ("a", 2)])  # → {"a": 2}  (1 is lost)

# Intended behavior
# When multiple values exist for the same month, select the larger value
```

### Bug 2: Ignoring the underwriting process

```python
@property
def health_insurance_payments(self):
    return Payment.objects.filter(
        cert__submission__deal_application=self._deal_application,
        payment_month__gte=self.start_year_month,
    )
```

This query **retrieves all submission data regardless of the currently active underwriting process.**

```mermaid
flowchart TD
    subgraph da["하나의 대출 신청"]
        A["가심사 제출<br>(제휴 페이 스크래핑)"]
        B["자동심사 제출<br>(공공 마이데이터)"]
    end
    subgraph payments["Payment 테이블"]
        C["가심사 Payment 데이터"]
        D["자동심사 Payment 데이터"]
    end
    A --> C
    B --> D
    
    E["기존 쿼리: 전부 다 가져옴"] --> C
    E --> D
    F["올바른 쿼리: 현재 심사의<br>최신 제출 데이터만"] --> D
    style E fill:#ff6b6b,stroke:#333,color:#fff
    style F fill:#51cf66,stroke:#333,color:#fff
```

Data from pre-screening and automated underwriting was mixed together in the QuerySet, and during the `dict()` conversion, which value survived depended on ordering.

---

## Fix

### Fix 1: Query only the latest submission data

```python
@property
def health_insurance_submission(self):
    return self.health_insurance_submissions.latest("created")
```

Filtering was added to retrieve only the Payment records linked to the **most recent submission** for the current underwriting path.

### Fix 2: Select the larger value for same-month data

Instead of `dict()`, explicit aggregation logic was implemented.

```python
# AS-IS: dict() - later value overwrites (non-deterministic)
payments = dict(queryset.values_list("payment_month", "amount"))

# TO-BE: When multiple values exist for the same month, select the largest (deterministic)
result = {}
for month, amount in queryset.values_list("payment_month", "amount"):
    if month not in result or amount > result[month]:
        result[month] = amount
```

```mermaid
flowchart LR
    subgraph input["입력 데이터 (같은 월 중복)"]
        A["2025-02: 0 (전 직장 환급)"]
        B["2025-02: 163070 (현 직장 납부)"]
    end
    subgraph as_is["AS-IS: dict()"]
        C["순서에 따라 하나만 남음<br>(비결정적)"]
    end
    subgraph to_be["TO-BE: 큰 값 선택"]
        D["163070 선택<br>(결정적)"]
    end
    input --> as_is
    input --> to_be
```

**Why the larger value?** In the case of concurrent employment (paying at two workplaces simultaneously), using the higher-income workplace as the basis is more conservative. Additionally, actual payment amounts are more meaningful data than refunds (negative amounts).

---

## Lessons

### 1. Check for key duplication when converting QuerySets with dict()

```python
# Unsafe - unless key uniqueness is guaranteed
dict(queryset.values_list("key", "value"))

# Alternative 1: Verify key uniqueness
assert queryset.values("key").distinct().count() == queryset.count()

# Alternative 2: Explicit aggregation
from collections import defaultdict
grouped = defaultdict(list)
for key, value in queryset.values_list("key", "value"):
    grouped[key].append(value)
```

### 2. Question the filtering scope of ORM queries

Always ask: "Is this query fetching exactly the data range I need?" Especially in structures where a single entity (loan application) can have data from multiple contexts (pre-screening/automated underwriting), filtering to the current context is essential.

### 3. In financial data, "non-deterministic" is a bug

The fact that `dict()` conversion results depend on QuerySet ordering means that **the same input can produce different outputs.** In a typical application this might not be a major issue, but in a loan underwriting system where results directly determine approval or rejection, non-deterministic behavior is a serious bug.
