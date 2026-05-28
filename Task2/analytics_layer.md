# Аналитический слой целевой архитектуры с Privacy by Design

Один из бизнес-целей «Медикаменте» — **BI, ML и AI на основе накопленных данных**.
По умолчанию это создаёт высокий риск: аналитик и data scientist соприкасаются с
сырыми ПДн и врачебной тайной. Целевой аналитический слой проектируется так,
чтобы получать всю пользу от данных, **не выдавая аналитикам и моделям прямой
доступ к L3/L4-данным**.

## Принципы построения слоя

1. **Privacy by default** — по умолчанию ни один человек и ни одна модель не
   получают доступ к raw-данным. Любой такой доступ — это исключение, оформленное
   через DPO с временным окном.
2. **Single source of truth — операционные домены**. Аналитика **не пишет** в
   операционные БД. Любые витрины — производные.
3. **Push, не pull**. Данные попадают в аналитический слой через событийный поток
   (CDC + Domain Events) — нет прямого подключения BI к operational-БД.
4. **Tag-aware**. Каждый атрибут несёт теги (см. `Task1/tagging.md`); политики
   маскирования и анонимизации применяются автоматически.
5. **Минимизация в источнике**. На этапе ETL поля с тегом `protect:anonymize`
   удаляются или агрегируются. Поля с `protect:pseudonymize` заменяются токенами.
6. **Дифференциальная приватность** для ML-датасетов с высокой чувствительностью.
7. **Локализация в РФ** — все витрины и Lake — в российском ЦОДе.

## Архитектура слоя (Mermaid)

```mermaid
flowchart LR
    subgraph OP[Operational Domain]
        DB1[(Patient DB)]
        DB2[(EMR DB)]
        DB3[(Sensitive EMR DB)]
        DB4[(Payment DB)]
        DB5[(Inventory DB)]
        DB6[(HR DB)]
        S3o[(Object Storage)]
    end

    BUS[/Kafka Event Bus<br/>mTLS, headers с тегами/]

    subgraph PBD[Privacy & Security Platform]
        CAT[Data Catalog<br/>Apache Atlas]
        KMS[KMS / Vault]
        TOK[PII Tokenization]
        OPA[Policy Decision Point]
        SIEM[Audit / SIEM]
    end

    subgraph A[Analytics Layer — PbD-Aware]
        CDC[Debezium CDC<br/>прозрачное чтение]
        ANON[Anonymization Pipeline<br/>Spark + ARX:<br/>- pseudonymize<br/>- k-anonymity<br/>- l-diversity<br/>- differential privacy]
        LAKE[(Privacy-aware<br/>Data Lake<br/>S3 + Iceberg<br/>доступ только сервисам PbD)]
        MART_BI[(BI Mart<br/>ClickHouse<br/>L1-L2 + анонимизированные L3)]
        MART_ML[(ML Feature Store<br/>Feast + S3<br/>псевдонимизированные L3,<br/>diff-privacy для L4)]
        BI[DataLens]
        NB[JupyterHub<br/>в изолированной среде]
        MLOPS[MLflow + Airflow]
    end

    Analyst((Аналитик))
    DS((Data Scientist))
    DPO((DPO))

    DB1 --> CDC
    DB2 --> CDC
    DB4 --> CDC
    DB5 --> CDC
    DB6 --> CDC
    S3o --> CDC
    DB3 -. "ТОЛЬКО агрегаты с diff-privacy" .-> ANON

    CDC --> BUS
    BUS --> ANON
    ANON <--> CAT
    ANON <--> TOK
    ANON --> LAKE
    ANON --> MART_BI
    ANON --> MART_ML

    BI --> MART_BI
    NB --> MART_BI
    NB --> MART_ML
    MLOPS --> MART_ML

    Analyst -->|HTTPS + MFA| BI
    DS -->|HTTPS + MFA + JIT-доступ| NB

    BI --> OPA
    NB --> OPA
    LAKE --> OPA
    SIEM <-- "события чтения витрин" --- BI
    SIEM <-- "запуск ноутбука, экспорт" --- NB
    SIEM <-- "JIT-разрешение DPO" --- DPO

    style PBD fill:#fee2e2,stroke:#b91c1c
    style A fill:#dbeafe,stroke:#1d4ed8
```

## Стек слоя

| Компонент | Технология | Роль |
|-----------|------------|------|
| **CDC** | Debezium | Прозрачно вычитывает изменения из operational-БД в Kafka |
| **Event Bus** | Apache Kafka + mTLS + ACL | Транспорт доменных событий с тегами в header |
| **Anonymization Pipeline** | Apache Spark + ARX Anonymization Tool | Применяет политики из Data Catalog: маски, токены, k-анонимизация, diff. приватность |
| **Privacy-aware Data Lake** | S3-совм. + Apache Iceberg | Хранилище сырых событий. Доступ — только сервисам PbD, не людям |
| **BI Mart** | ClickHouse | Витрины для дашбордов. Содержат L1-L2 без изменений, L3 — обезличенные, L4 — отсутствуют |
| **ML Feature Store** | Feast + S3 | Признаки для ML. L3 — псевдонимизированы (обратимо для аудита через Vault), L4 — добавлен шум diff-privacy |
| **BI** | Yandex DataLens / Apache Superset | Дашборды P&L, ABC, KPI |
| **Notebook** | JupyterHub в k8s, без выхода в интернет | ML / AI эксперименты |
| **MLOps** | MLflow + Apache Airflow | Регистрация моделей, расписания |
| **Audit** | Elastic + Wazuh | Что прочитано, кем, когда; экспорт = отдельное событие |

## Управляющие политики

### Политика 1. «Никакого L4+ в аналитике»

```rego
# OPA, упрощённый Rego
package medikamente.analytics.access

default allow := false

allow if {
  input.resource.tags["sensitivity:l1"]
}
allow if {
  input.resource.tags["sensitivity:l2"]
}
allow if {
  input.resource.tags["sensitivity:l3"]
  input.resource.tags["protect:anonymize"]  # значит уже обезличено
}

# L4 / L4+ запрещены всем, кроме явного JIT-разрешения от DPO
allow if {
  input.resource.tags["sensitivity:l4"]
  some grant in data.jit_grants[input.user.id]
  grant.expires_at > time.now_ns()
  grant.scope == input.resource.id
}
```

### Политика 2. JIT-доступ через DPO

- Аналитик не может «сразу» получить L3-данные с реальными идентификаторами.
- Если требуется (например, для разбора инцидента), он создаёт **тикет в DPO
  Console** с обоснованием.
- DPO выдаёт **временный grant** (от 1 часа до недели) на конкретный набор данных.
- Все действия аналитика в течение этого окна **дополнительно логируются** и
  алертятся при экспортах.

### Политика 3. Экспорт = особое событие

- Скачивание датасета из ноутбука или дашборда — отдельный аудируемый
  событийный канал.
- DLP проверяет содержимое экспорта на наличие тегов чувствительности.
- При попытке экспорта L3+ без активного JIT-grant — блок + алерт DPO.

## Дифференциальная приватность для ML на L4

Для моделей, которые обучаются на медицинских данных (например, прогноз потока
пациентов по диагнозам), используется **(ε, δ)-differential privacy**:

- В Anonymization Pipeline на признаки, попадающие в Feature Store, добавляется
  калиброванный шум (Laplace / Gaussian).
- ε и δ фиксируются на уровне политики и хранятся как метаданные модели в MLflow.
- Это обеспечивает математическую гарантию того, что вклад **отдельного
  пациента** в модель ограничен и его данные нельзя извлечь обратно.

## Что закрывает аналитический слой из проблем As-Is

| Проблема As-Is (см. `Task1/problems.md`) | Как закрывается |
|-------------------------------------------|-----------------|
| #22 — BI на «сырых» Excel | Аналитика только через Anonymization Pipeline |
| #4 — Избыточный сбор полей | Pipeline удаляет поля без явного назначения (по тегам) |
| #16 — ВИЧ вместе с обычными | Sensitive EMR DB вообще не попадает в Lake, только diff-privacy агрегаты |
| #8 — Нет аудита | Все чтения витрин, открытия ноутбуков, экспорты — в SIEM |
| #14 — «Право быть забытым» | Retention Engine публикует `subject.erased`, pipeline отбрасывает данные на ближайшем запуске |
| #13 — Бессрочное хранение | Lake/Mart имеют политику ретеншена по тегам, как и operational-БД |

## Итог

Аналитический слой построен так, чтобы **«знание» компании росло без накопления
рисков**. Архитектурно это достигается за счёт:

- односторонней связи operational → analytics (через CDC и шину событий);
- обязательного прохождения данных через Anonymization Pipeline;
- запрета прямого доступа человека к Data Lake;
- разделения витрин на BI (для людей) и Feature Store (для моделей);
- JIT-доступа и аудита через DPO Console;
- дифференциальной приватности для самых чувствительных моделей.
