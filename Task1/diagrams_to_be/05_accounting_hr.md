# DFD 5 (To-Be) — Бухгалтерия, зарплаты, кадровый учёт + средства защиты

```mermaid
graph LR
    HR["HR / бухгалтер<br/>🛡️ роль hr"]:::ext
    EMP["Сотрудник<br/>(self-service)"]:::ext
    PORTAL["HR-портал<br/>🔒 TLS + MFA<br/>🛡️ OPA"]:::proc
    HRSVC["HR-сервис"]:::proc
    KMS["KMS / Vault 🔐"]:::sec
    DOCS[("S3 SSE-KMS<br/>сканы паспортов,<br/>мед.книжек<br/>теги: class:pii-special,<br/>sensitivity:l4")]:::store
    HRDB[("PostgreSQL<br/>🔐 TDE<br/>field-level encrypt<br/>(СНИЛС, паспорт)")]:::store
    PAY[("Payroll-сервис<br/>🔐 TDE")]:::store
    BANK_GW["Bank Gateway<br/>🔒 mTLS + ЭП<br/>🎟️ токен реестра"]:::proc
    BANK["Банк"]:::ext
    FNS_GW["Контур / 1С-Отчётность<br/>🔒 TLS + ЭП"]:::proc
    FNS["ФНС / СФР"]:::ext
    LOG["Vector → SIEM<br/>🧱 mask СНИЛС/паспорт"]:::sec
    RET["Retention engine 🗑️<br/>после увольнения + срок"]:::sec

    EMP -->|"🔒 self-service<br/>(анкета)"| PORTAL
    HR -->|"🔒 + MFA"| PORTAL
    PORTAL --> HRSVC
    HRSVC -->|"🔐 шифр-запись"| HRDB
    HRSVC -->|"сканы"| DOCS
    HRSVC -->|"оклад/премии"| PAY
    HRSVC -->|"data-key"| KMS
    PAY -->|"реестр на выплату"| BANK_GW
    BANK_GW -->|"🔒 ЭП + mTLS"| BANK
    PAY -->|"проводка налогов"| FNS_GW
    FNS_GW --> FNS
    HRSVC -.->|"audit"| LOG
    PAY -.->|"audit"| LOG
    RET --> HRDB
    RET --> DOCS

    classDef ext fill:#fef3c7,stroke:#b45309,stroke-width:1px,color:#111;
    classDef proc fill:#dbeafe,stroke:#1d4ed8,stroke-width:1px,color:#111;
    classDef store fill:#dcfce7,stroke:#15803d,stroke-width:1px,color:#111;
    classDef sec fill:#fee2e2,stroke:#b91c1c,stroke-width:1px,color:#111;
```

## Что добавлено относительно As-Is

| Этап | Инструмент | Тег |
|------|------------|-----|
| Доступ HR | MFA + OPA-политика | `class:hr`, `sensitivity:l3` |
| Сканы документов | S3 SSE-KMS, теги | `class:pii-special`, `sensitivity:l4` |
| СНИЛС, паспорт, ИНН | Vault Transform — токенизация | `protect:tokenize` |
| Реестры в банк | mTLS + усиленная ЭП, токены вместо ПДн там, где можно | `protect:encrypt-in-transit` |
| Логи | Маскирование номеров документов | `protect:mask-in-logs` |
| Retention | Срок хранения после увольнения (75 лет — личные дела, 5 лет — ЗП) | `legal:retention:75y` / `5y` |
| Аудит | Алерт на чтение кадровых документов вне рабочих часов | — |
