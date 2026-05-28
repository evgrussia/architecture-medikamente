# DFD 4 (To-Be) — Приём оплаты услуг + средства защиты

```mermaid
graph LR
    PAT["Пациент<br/>(касса / онлайн-оплата)"]:::ext
    KKM["ККМ<br/>🔒 TLS-туннель к 1С<br/>🔐 сертификат на устройстве"]:::ext
    BANK["Эквайер<br/>(банк)"]:::ext
    GW["Payment Gateway<br/>🔒 TLS 1.3<br/>🛡️ OPA<br/>📜 SIEM"]:::proc
    PAY["Payment Service"]:::proc
    TOK["Vault Transform 🎟️<br/>PAN → token"]:::sec
    DB[("PostgreSQL<br/>🔐 TDE<br/>хранит token, не PAN")]:::store
    ONEC[("1С: Бухгалтерия<br/>🔐 SQL-режим + TDE<br/>🛡️ роли")]:::store
    LOG["Vector → SIEM<br/>🧱 mask PAN<br/>📜 audit"]:::sec
    OFD["ОФД / ФНС"]:::ext
    BI[("BI-витрина<br/>🧮 анонимизированные<br/>агрегаты")]:::store
    ETL["ETL c анонимизацией<br/>🧮 удалить пациента"]:::proc

    PAT -->|"оплата"| KKM
    KKM -->|"авторизация PAN"| BANK
    BANK -->|"approve"| KKM
    KKM -->|"🔒 TLS<br/>чек, услуга, сумма"| GW
    GW --> PAY
    PAY -->|"PAN → токен"| TOK
    TOK --> PAY
    PAY -->|"🔐 запись (token, услуга, сумма)"| DB
    PAY -->|"проводка"| ONEC
    PAY -.->|"event: payment.confirmed"| LOG
    ONEC -->|"фискальные данные"| OFD
    DB --> ETL
    ETL -->|"без ФИО, без услуги-диагноза"| BI

    classDef ext fill:#fef3c7,stroke:#b45309,stroke-width:1px,color:#111;
    classDef proc fill:#dbeafe,stroke:#1d4ed8,stroke-width:1px,color:#111;
    classDef store fill:#dcfce7,stroke:#15803d,stroke-width:1px,color:#111;
    classDef sec fill:#fee2e2,stroke:#b91c1c,stroke-width:1px,color:#111;
```

## Что добавлено относительно As-Is

| Этап | Инструмент | Тег |
|------|------------|-----|
| ККМ ↔ 1С | TLS-туннель + сертификат на устройстве | `protect:encrypt-in-transit` |
| PAN | Токенизация Vault Transform — в системах хранится только токен | `protect:tokenize`, `class:payment-card` |
| 1С: Бухгалтерия | Перевод в клиент-серверный режим, TDE, роли пользователей | `protect:encrypt-at-rest` |
| Журнал оплат | Из Excel убирается; единая БД с TDE и аудитом | `class:financial`, `sensitivity:l3` |
| Логи | Маска PAN: 1234 **** **** 5678 | `protect:mask-in-logs` |
| BI / P&L | Анонимизация: удаляется ID пациента, услуга обобщается до категории | `protect:anonymize` |
| Аудит | SIEM-алерт на ручную правку платежей задним числом | — |
