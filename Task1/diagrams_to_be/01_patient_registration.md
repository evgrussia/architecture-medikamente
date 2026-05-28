# DFD 1 (To-Be) — Запись пациента на приём + средства защиты

```mermaid
graph LR
    P["Пациент<br/>(портал / моб. прил.)"]:::ext
    GW["API Gateway<br/>🔒 TLS 1.3<br/>🛡️ OPA по тегам<br/>📜 в SIEM"]:::proc
    CONSENT["Consent Service ✅<br/>проверка согласия"]:::sec
    REG["Сервис записи<br/>🛡️ RBAC + ABAC"]:::proc
    DB[("PostgreSQL<br/>🔐 TDE + RLS<br/>field-level encrypt<br/>(via KMS)")]:::store
    KMS["KMS / Vault<br/>🔐 управление ключами"]:::sec
    OBJ[("S3-совм. хранилище<br/>🔐 SSE-KMS<br/>тег sensitivity:l3")]:::store
    LOG["Vector → SIEM<br/>🧱 маскирование PII<br/>📜 аудит"]:::sec
    NOT["Notification Service<br/>🔒 TLS to SMS/Email GW<br/>🧱 маска номера"]:::proc
    R["Сотрудник ресепшен<br/>🛡️ роль reception<br/>🧱 маски ФИО в UI"]:::ext
    RET["Retention engine 🗑️<br/>legal:retention:*"]:::sec

    P -->|"[class:pii] анкета (TLS)"| GW
    GW --> CONSENT
    CONSENT -->|"согласие получено"| REG
    REG -->|"шифрование поля → ключ"| KMS
    KMS -->|"data-key"| REG
    REG -->|"🔐 зашифрованный INSERT"| DB
    REG -->|"скан паспорта"| OBJ
    REG -.->|"события доступа"| LOG
    DB -.->|"audit log"| LOG
    R -->|"чтение записи<br/>🛡️ OPA проверяет тег"| GW
    DB -->|"данные с масками"| R
    REG --> NOT
    NOT -->|"подтверждение SMS/email"| P
    RET -->|"удаление по таймеру"| DB
    RET -->|"удаление по таймеру"| OBJ

    classDef ext fill:#fef3c7,stroke:#b45309,stroke-width:1px,color:#111;
    classDef proc fill:#dbeafe,stroke:#1d4ed8,stroke-width:1px,color:#111;
    classDef store fill:#dcfce7,stroke:#15803d,stroke-width:1px,color:#111;
    classDef sec fill:#fee2e2,stroke:#b91c1c,stroke-width:1px,color:#111;
```

## Что добавлено относительно As-Is

| Этап | Инструмент | Какой тег срабатывает |
|------|------------|-----------------------|
| Канал клиент→портал | TLS 1.3 (Let's Encrypt / российский УЦ) | `protect:encrypt-in-transit` |
| API Gateway | OPA-плагин читает `sensitivity` + `domain` | `sensitivity:l3` → роль reception/patient |
| Consent Service | Проверка статуса согласия пациента | `legal:consent-required` |
| БД пациентов | PostgreSQL TDE + pgcrypto field-level | `protect:encrypt-at-rest`, `protect:field-level-encrypt` |
| Хранилище сканов | S3 SSE-KMS + object tags | `class:pii-special`, `sensitivity:l4` |
| Логи | Vector маскирует ФИО, телефон | `protect:mask-in-logs` |
| UI ресепшена | Маски в Web-формах | `protect:mask-in-ui` |
| SIEM | Алерт на чтение L4 без роли doctor | — |
| Retention Engine | Удаляет записи без активности 3 года | `legal:retention:3y` (анкета без записи) |
