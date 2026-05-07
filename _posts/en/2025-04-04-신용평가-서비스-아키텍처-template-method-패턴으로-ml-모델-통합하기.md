---
title: "Credit Scoring Service Architecture - Integrating ML Models with the Template Method Pattern"
date: 2025-04-04
categories: [System Design]
tags: [architecture, ml, django]
mermaid: true
---

## Background

A loan underwriting system contains multiple types of credit scoring models. Fraud detection models, credit score models, recovery rate prediction models -- each takes different input data and calculates scores differently, but **the overall flow of "gather data, calculate a score, and save the result"** is identical across all of them.

The problem was that similar code was being duplicated every time a new model was added, and each model was implemented in its own way, making maintenance difficult.

---

## Solution: BaseScoreService Abstract Class

By applying the Template Method pattern, the common flow is defined in the parent class while only the model-specific differences are implemented in child classes.

```mermaid
classDiagram
    class BaseScoreService {
        +features_from_deal_application
        +all_feature_names
        +categorical_features_names
        +value_transformer
        +default_value_map
        +model_name
        +__init__(deal_application, version)
        +get_data_from_source()*
        +calculate_score()*
        +save_score()
        +delete_score()
        +rendered_data()
        +get_score()
    }
    class FraudScoreService {
        +get_data_from_source()
        +calculate_score()
    }
    class CreditScoreService {
        +get_data_from_source()
        +calculate_score()
    }
    class 회수율 예측 모델Service {
        +get_data_from_source()
        +calculate_score()
    }
    BaseScoreService <|-- FraudScoreService
    BaseScoreService <|-- CreditScoreService
    BaseScoreService <|-- 회수율 예측 모델Service
```

### Class Attributes: "What does this model use?"

Each service class declares six attributes to specify what data it uses.

| Attribute | Role |
|-----------|------|
| `features_from_deal_application` | Values to retrieve from the loan application |
| `all_feature_names` | Complete list of features |
| `categorical_features_names` | Categorical data (enums) |
| `value_transformer` | Data requiring value transformation |
| `default_value_map` | Default value mapping |
| `model_name` | Model name used at the ML inference service endpoint |

### Methods: "Common Flow" vs. "Model-Specific Implementation"

```mermaid
flowchart TD
    subgraph common["공통 (BaseScoreService)"]
        A["__init__: 모델 버전 조회 + 인스턴스화"]
        B["save_score: 결과 저장"]
        C["delete_score: soft delete"]
        D["rendered_data: 조회"]
    end
    subgraph override["모델별 구현 (override 필수)"]
        E["get_data_from_source: 데이터 수집"]
        F["calculate_score: 점수 계산"]
    end
    A --> E --> F --> B
```

Simply implement `get_data_from_source()` and `calculate_score()`, and the new model is operational. The parent class handles everything else.

---

## Model Version Management

ML models change versions. Even for the same "fraud detection," v1 and v2 may use different features and call different ML inference service endpoints.

```mermaid
sequenceDiagram
    participant Service as ScoreService
    participant DB as ml_model_registry
    participant SM as ML 추론 서비스

    Service->>DB: model_name + version으로 조회
    DB-->>Service: 모델 인스턴스 (엔드포인트 정보 포함)
    Service->>Service: get_data_from_source()
    Service->>SM: 엔드포인트 호출 (feature 전달)
    SM-->>Service: 점수 반환
    Service->>DB: save_score()
```

During `__init__`, the `ml_model_registry` table is queried to retrieve the model instance for that version. To deploy a new version, you simply add a record to the DB and change the configuration.

---

## Data Delivery Strategy: "Don't Preprocess"

Initially, data preprocessing (normalization, encoding, etc.) was done in the service layer. However, this approach required modifying the preprocessing code every time the model version changed.

```mermaid
flowchart LR
    subgraph before["AS-IS"]
        A["데이터 수집"] --> B["전처리<br>(정규화, 인코딩)"] --> C["ML 추론 서비스"]
    end
    subgraph after["TO-BE"]
        D["데이터 수집"] --> E["all_feature_names에<br>전부 넣어서 전달"] --> F["ML 추론 서비스<br>(전처리는 모델 내부)"]
    end
```

Decision: Send all parameters via `all_feature_names`, and let the ML inference service model handle preprocessing internally. The service layer focuses solely on "gathering data and sending it."

---

## New Model Addition Checklist

As the system grows in complexity, the biggest cost becomes "what do I need to do to add a new model?" Thanks to the Template Method pattern, this process has been standardized.

```mermaid
flowchart TD
    A["1. 클래스 속성 6종 정의"] --> B["2. settings.py에<br>모델명 + ML 추론 서비스 엔드포인트 추가"]
    B --> C["3. ml_model_registry<br>테이블에 레코드 추가"]
    C --> D["4. get_data_from_source() 구현"]
    D --> E["5. calculate_score() 구현"]
    E --> F["새 모델 동작 ✓"]
```

Five steps and a new model is added. Not a single line of common logic (save, delete, query, version management) needs to be touched.

---

## Reflections

### Template Method is the perfect pattern for "same flow, different details"
When the flow of "data collection, score calculation, save" is identical across all models and only the collection/calculation methods differ, Template Method is the right choice. The cost of adding a new model is reduced to "creating one class."

### Delegating preprocessing to the model simplifies the service
If the service layer handles preprocessing, the code branches for every model version. By splitting responsibilities into "gather raw data and send it" while "the model handles its own preprocessing," there's no need to touch the backend code when versions change.

### The value of a checklist emerges after the pattern is established
Creating a checklist without a pattern just yields a "list of things to do." A checklist after a pattern is established becomes a guarantee that "this is all you need to do." This difference is significant.
