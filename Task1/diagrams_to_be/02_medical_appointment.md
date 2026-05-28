# DFD 2 (To-Be) — Приём специалистом и ведение медкарты + средства защиты

```mermaid
graph LR
    MD["Врач<br/>🛡️ роль doctor +<br/>ABAC: attending_doctor"]:::ext
    GW["API Gateway<br/>🔒 mTLS<br/>🛡️ OPA<br/>📜 SIEM"]:::proc
    EMR["EMR-сервис<br/>(электронная медкарта)<br/>field-level encrypt"]:::proc
    BTG["Break-the-glass<br/>обоснованный доступ<br/>📜 алерт в SIEM"]:::sec
    KMS["KMS / Vault<br/>🔐 ключи per домен<br/>🔐 доп. ключ для L4+"]:::sec
    EMR_DB[("PostgreSQL<br/>🔐 TDE + RLS<br/>schema: medical<br/>field-level encrypt<br/>(спец. ключ для L4+)")]:::store
    DOCS[("S3-совм.<br/>🔐 SSE-KMS<br/>заключения, направления")]:::store
    PRINT["Pull-print Server<br/>🛡️ аутентификация<br/>картой сотрудника<br/>📜 SIEM"]:::proc
    LOG["Vector → SIEM<br/>🧱 redact diagnoses<br/>📜 audit"]:::sec
    BUS["Kafka<br/>🔒 mTLS + ACL<br/>🧱 header с тегами"]:::store
    RET["Retention engine 🗑️<br/>75 лет для медкарты"]:::sec

    MD -->|"запрос медкарты<br/>(JWT с ролью)"| GW
    GW --> EMR
    EMR -->|"OPA: doctor && attending<br/>иначе → BTG"| BTG
    BTG -->|"при подтверждении"| EMR
    EMR -->|"data-key (envelope)"| KMS
    EMR -->|"🔐 чтение/запись"| EMR_DB
    EMR -->|"заключение PDF"| DOCS
    EMR -->|"печать"| PRINT
    PRINT -->|"бумажный экземпляр<br/>после идентификации"| MD
    EMR -.->|"событие: открыта медкарта"| BUS
    BUS --> LOG
    EMR_DB -.->|"audit log таблиц"| LOG
    RET -->|"архив/удаление"| EMR_DB
    RET -->|"архив/удаление"| DOCS

    classDef ext fill:#fef3c7,stroke:#b45309,stroke-width:1px,color:#111;
    classDef proc fill:#dbeafe,stroke:#1d4ed8,stroke-width:1px,color:#111;
    classDef store fill:#dcfce7,stroke:#15803d,stroke-width:1px,color:#111;
    classDef sec fill:#fee2e2,stroke:#b91c1c,stroke-width:1px,color:#111;
```

## Что добавлено относительно As-Is

| Этап | Инструмент | Тег |
|------|------------|-----|
| API доступа врача | OPA-политика `doctor && attending_doctor == current_user` | `domain:medical`, `sensitivity:l4` |
| Break-the-glass | Обоснованный доступ к чужому пациенту с обязательной фиксацией причины | `class:medical`, `sensitivity:l4` |
| Спец. категории (ВИЧ и пр.) | Отдельный ключ KMS, отдельная схема `medical_sensitive` | `class:medical-sensitive`, `sensitivity:l4-plus` |
| Печать | Pull-printing с картой сотрудника | — |
| Kafka-события | mTLS + ACL, header `x-data-tags` для downstream | все теги переносятся в lineage |
| SIEM | Алерт на BTG, на массовое чтение, на доступ к L4+ | — |
| Retention | 75 лет для медкарт (как первичная медицинская документация) | `legal:retention:75y` |
