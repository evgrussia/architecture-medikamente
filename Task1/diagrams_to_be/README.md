# DFD To-Be — c инструментами защиты на каждом этапе

В этой директории — те же шесть DFD, что и в `../diagrams/`, но с указанием:

- какие **теги** (см. `../tagging.md`) проставляются полям и сообщениям;
- какое **шифрование** действует на канале или в хранилище;
- какие **средства** обработки/защиты включены в поток (KMS, OPA, Vault, SIEM,
  Consent Service, Retention Engine, DLP и т.д.).

Чтобы не дублировать схему процесса, на To-Be диаграммах сохраняется такая же
структура акторов и хранилищ, но рядом подписан используемый инструмент.

## Условные обозначения

| Иконка | Назначение |
|--------|------------|
| 🔐 | Шифрование at-rest (TDE / KMS) |
| 🔒 | Шифрование канала (TLS / mTLS) |
| 🎟️ | Токенизация / псевдонимизация (Vault Transform) |
| 🧱 | Маскирование (UI / logs) |
| 🧮 | Обезличивание (BI / ML) |
| 🛡️ | Политика доступа OPA / ABAC по тегам |
| 📜 | Запись в SIEM / журнал доступа |
| 🗑️ | Retention engine (срок хранения) |
| ✅ | Consent service (проверка согласия) |

## Файлы

- [`01_patient_registration.md`](01_patient_registration.md)
- [`02_medical_appointment.md`](02_medical_appointment.md)
- [`03_lab_analysis.md`](03_lab_analysis.md)
- [`04_payment.md`](04_payment.md)
- [`05_accounting_hr.md`](05_accounting_hr.md)
- [`06_inventory.md`](06_inventory.md)
