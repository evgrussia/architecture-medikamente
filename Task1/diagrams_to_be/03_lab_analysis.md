# DFD 3 (To-Be) — Лабораторные анализы + средства защиты

```mermaid
graph LR
    MD["Врач"]:::ext
    EMR["EMR-сервис"]:::proc
    LAB_GW["Lab Integration<br/>Gateway<br/>🔒 mTLS + JWS подпись<br/>🛡️ OPA по контракту"]:::proc
    LAB["Внешняя лаборатория<br/>(контрагент по DPA)"]:::ext
    KMS["KMS / Vault 🔐"]:::sec
    EMR_DB[("PostgreSQL<br/>🔐 TDE<br/>field-level<br/>(результаты)")]:::store
    DOCS[("S3 SSE-KMS<br/>теги:<br/>class:medical,<br/>sensitivity:l4")]:::store
    PII_TOK["PII-Tokenization<br/>(Vault Transform) 🎟️"]:::sec
    LOG["Vector → SIEM<br/>🧱 redact результатов<br/>📜 audit"]:::sec
    RET["Retention engine 🗑️"]:::sec
    PAT["Пациент<br/>(моб. прил.)"]:::ext

    MD -->|"направление"| EMR
    EMR -->|"оригинальный ID пациента"| PII_TOK
    PII_TOK -->|"токен вместо ФИО/СНИЛС"| LAB_GW
    LAB_GW -->|"🔒 mTLS + подписанный JSON<br/>(только токен + материал)"| LAB
    LAB -->|"🔒 mTLS<br/>результат + токен"| LAB_GW
    LAB_GW -->|"токен → реальный ID"| PII_TOK
    PII_TOK --> EMR
    EMR -->|"🔐 шифрованная запись"| EMR_DB
    EMR -->|"PDF результата"| DOCS
    EMR -.->|"событие"| LOG
    PAT -->|"🔒 TLS + JWT<br/>🛡️ только свои анализы"| EMR
    EMR -->|"копия результата<br/>(маскированный шаблон)"| PAT
    RET --> EMR_DB
    RET --> DOCS

    classDef ext fill:#fef3c7,stroke:#b45309,stroke-width:1px,color:#111;
    classDef proc fill:#dbeafe,stroke:#1d4ed8,stroke-width:1px,color:#111;
    classDef store fill:#dcfce7,stroke:#15803d,stroke-width:1px,color:#111;
    classDef sec fill:#fee2e2,stroke:#b91c1c,stroke-width:1px,color:#111;
```

## Что добавлено относительно As-Is

| Этап | Инструмент | Тег |
|------|------------|-----|
| Канал с лабораторией | mTLS + JWS-подпись запросов, формализованный DPA | `protect:encrypt-in-transit` |
| Идентификатор пациента у внешнего контрагента | **Токенизация** через Vault Transform: лаборатория получает только токен | `protect:tokenize` |
| Хранение PDF результатов | S3 SSE-KMS + теги | `class:medical`, `sensitivity:l4` |
| Доступ пациента | Только к собственным анализам (`subject == current_user`) | `domain:patient` |
| Лог-сервис | Redact значений результатов в логах | `protect:mask-in-logs` |
| Retention | Анализы по правилам хранения медкарты | `legal:retention:25y` |
| Аудит | Алерт на массовую выгрузку анализов одним пользователем | — |
