# DFD 6 (To-Be) — Учёт ТМЦ и закупки + средства защиты

```mermaid
graph LR
    WH["Кладовщик<br/>🛡️ роль warehouse"]:::ext
    MD["Врач<br/>(заявка)"]:::ext
    INV["Inventory Service<br/>🔒 TLS"]:::proc
    SUPGW["Supplier Gateway<br/>🔒 mTLS + ЭП<br/>🛡️ DPA контракты"]:::proc
    SUP["Поставщик"]:::ext
    DB[("PostgreSQL<br/>🔐 TDE<br/>теги: class:commercial-secret")]:::store
    ONEC[("1С: Торговля и склад<br/>🔐 SQL + TDE<br/>🛡️ роли")]:::store
    BUS["Kafka<br/>🔒 mTLS"]:::store
    BI[("BI-витрина<br/>🧮 без привязки<br/>к пациенту/процедуре")]:::store
    LOG["SIEM 📜"]:::sec
    DLP["DLP<br/>(InfoWatch / KSPL)<br/>🛡️ контроль исходящего"]:::sec

    MD -->|"заявка с категорией"| INV
    WH -->|"приёмка"| INV
    INV -->|"🔐 запись"| DB
    INV -->|"проводка"| ONEC
    INV -->|"🔒 заказ поставщику"| SUPGW
    SUPGW --> DLP
    DLP -->|"одобрено"| SUP
    SUP -->|"🔒 прайс, накладные"| SUPGW
    SUPGW --> INV
    INV -.->|"audit"| LOG
    INV -->|"события списания (без пациента)"| BUS
    BUS --> BI

    classDef ext fill:#fef3c7,stroke:#b45309,stroke-width:1px,color:#111;
    classDef proc fill:#dbeafe,stroke:#1d4ed8,stroke-width:1px,color:#111;
    classDef store fill:#dcfce7,stroke:#15803d,stroke-width:1px,color:#111;
    classDef sec fill:#fee2e2,stroke:#b91c1c,stroke-width:1px,color:#111;
```

## Что добавлено относительно As-Is

| Этап | Инструмент | Тег |
|------|------------|-----|
| Канал с поставщиками | mTLS + ЭП, защищённый EDI-формат | `protect:encrypt-in-transit` |
| Документы | S3 SSE-KMS + теги | `class:commercial-secret` |
| 1С: Торговля и склад | Клиент-серверный режим + роли + TDE | `protect:encrypt-at-rest` |
| Витрина BI | Списания идут как агрегаты без привязки к пациенту | `protect:anonymize` |
| DLP | Контроль исходящих писем поставщикам на утечку прайсов и контактов | — |
| Аудит | Алерт на массовую выгрузку прайсов поставщиков | — |
