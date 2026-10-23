# Modelo de base de datos — Módulo 3 (Control de Uso, Sanciones y Analítica)

El modelo se divide en una **vista general** (solo tablas y relaciones) y **cinco vistas por área** con las columnas, para que cada diagrama se lea completo en GitHub. Las notas entre comillas de cada columna indican valores permitidos, unicidad u opcionalidad.

## 1. Vista general

```mermaid
erDiagram
    STUDENT_REF ||--o| TRUST_SCORE : tiene
    STUDENT_REF ||--o{ SCORE_HISTORY : registra
    STUDENT_REF ||--o{ RESERVATION_COPY : titular
    STUDENT_REF ||--o{ SANCTION : recibe
    STUDENT_REF ||--o{ ABSENCE_CONFIRMATION : ausente
    STUDENT_REF ||--o{ OVERDUE_DEDUCTION : sufre
    STUDENT_REF ||--o{ UNBLOCK_AUDIT : desbloqueado
    STUDENT_REF ||--o{ CHARGE : responsable
    STUDENT_REF |o--o{ DAMAGE_REPORT : responsable

    RESERVATION_COPY ||--o| USAGE : origina
    RESERVATION_COPY ||--o{ OVERDUE_DEDUCTION : genera
    RESERVATION_COPY ||--o| LOST_LOAN : "puede perderse"
    RESERVATION_COPY ||--o| ABSENCE_CONFIRMATION : "se confirma"
    USAGE ||--o| CHECK_OUT : "se cierra con"
    USAGE |o--o| DAMAGE_REPORT : "se reporta"

    ABSENCE_CONFIRMATION |o--o| SANCTION : crea
    ABSENCE_CONFIRMATION ||--o{ ABSENCE_ANNULMENT_AUDIT : "se anula"
    UNBLOCK_AUDIT |o--o{ SANCTION : levanta

    DAMAGE_REPORT |o--o| CHARGE : genera
    LOST_LOAN |o--o| CHARGE : genera
    COST_CATALOG ||--o{ CHARGE : "fija monto"
    CHARGE ||--o{ CHARGE_AUDIT : audita

    SYSTEM_CONFIGURATION
    OUTBOX_MESSAGE
    PROCESSED_EVENT
```

## 2. Estudiante, score y desbloqueo

```mermaid
erDiagram
    STUDENT_REF ||--o| TRUST_SCORE : tiene
    STUDENT_REF ||--o{ SCORE_HISTORY : registra
    STUDENT_REF ||--o{ UNBLOCK_AUDIT : desbloqueado
    UNBLOCK_AUDIT |o--o{ SANCTION : levanta
    STUDENT_REF ||--o{ SANCTION : recibe

    STUDENT_REF {
        string student_code PK
        string name
    }
    TRUST_SCORE {
        string student_code PK "FK"
        int score "0 a 50; bloqueado si es 0"
        datetime last_infraction_at
        datetime last_recovery_at
        bigint version "bloqueo optimista"
    }
    SCORE_HISTORY {
        bigint id PK
        string student_code FK
        string type "ver tipos en RN-SCO"
        int points
        int score_before
        int score_after
        string origin_type "unico con origin_id y type"
        string origin_id
        datetime created_at
    }
    UNBLOCK_AUDIT {
        bigint id PK
        string student_code FK
        string unblocked_by
        datetime unblocked_at
        string reason
        int score_before
        int score_after
        int sanctions_lifted
    }
    SANCTION {
        bigint id PK
        string student_code FK
        string type "INASISTENCIA"
        string status "VIGENTE, CUMPLIDA, LEVANTADA, ANULADA"
        date starts_on
        date ends_on
        bigint absence_confirmation_id FK
        bigint unblock_audit_id FK
    }
    SYSTEM_CONFIGURATION {
        string config_key PK
        string value
        string description
    }
```

Tipos de `score_history.type`: `AUMENTO_CHECKOUT`, `REDUCCION_SANCION`, `REDUCCION_RETRASO`, `RECUPERACION`, `DESBLOQUEO_MANUAL`, `RESTITUCION_ANULACION`.

## 3. Reservas, check-out y descuento por retraso

```mermaid
erDiagram
    STUDENT_REF ||--o{ RESERVATION_COPY : titular
    STUDENT_REF ||--o{ OVERDUE_DEDUCTION : sufre
    RESERVATION_COPY ||--o| USAGE : origina
    RESERVATION_COPY ||--o{ OVERDUE_DEDUCTION : genera
    RESERVATION_COPY ||--o| LOST_LOAN : "puede perderse"
    USAGE ||--o| CHECK_OUT : "se cierra con"

    RESERVATION_COPY {
        string m2_reservation_id PK
        string origin
        boolean sanctionable
        string holder_code FK
        string resource_id
        string resource_name
        string resource_category "ACTIVO o ESPACIO"
        string resource_type
        string resource_location
        datetime starts_at
        datetime ends_at
        datetime loan_withdrawn_at
        datetime loan_due_at
        string status "ACTIVA o CANCELADA"
        datetime last_event_at
    }
    USAGE {
        bigint id PK
        string m2_reservation_id FK "unico"
        datetime due_at
        string status "ACTIVA o FINALIZADA"
    }
    CHECK_OUT {
        bigint id PK
        bigint usage_id FK "unico"
        string student_code FK
        datetime checked_out_at
        string result "A_TIEMPO o CON_RETRASO"
        int delay_minutes
        boolean has_damage
        string ack_status "PENDIENTE, ACEPTADO, RECHAZADO"
        string ack_reason
    }
    OVERDUE_DEDUCTION {
        bigint id PK
        string m2_reservation_id FK
        string student_code FK
        date deduction_date "unico por reserva y por estudiante"
        int points
    }
    LOST_LOAN {
        bigint id PK
        string m2_reservation_id FK "unico"
        datetime declared_at
    }
```

## 4. Inasistencia

```mermaid
erDiagram
    STUDENT_REF ||--o{ ABSENCE_CONFIRMATION : ausente
    RESERVATION_COPY ||--o| ABSENCE_CONFIRMATION : "se confirma"
    ABSENCE_CONFIRMATION |o--o| SANCTION : crea
    ABSENCE_CONFIRMATION ||--o{ ABSENCE_ANNULMENT_AUDIT : "se anula"

    ABSENCE_CONFIRMATION {
        bigint id PK
        string m2_reservation_id FK "unico"
        string student_code FK
        string space_id
        datetime verified_at
        string confirmed_by
        string status "PENDIENTE_ACUSE, ACEPTADA, RECHAZADA_M2, ANULADA"
        string reject_reason
        datetime accepted_from
    }
    ABSENCE_ANNULMENT_AUDIT {
        bigint id PK
        bigint absence_confirmation_id FK
        string annulled_by
        datetime annulled_at
        string reason
        int points_restored
    }
    SANCTION {
        bigint id PK
        string student_code FK
        string status "VIGENTE, CUMPLIDA, LEVANTADA, ANULADA"
        date starts_on
        date ends_on
        bigint absence_confirmation_id FK
    }
```

## 5. Novedad técnica y cobros

```mermaid
erDiagram
    USAGE |o--o| DAMAGE_REPORT : "se reporta"
    STUDENT_REF |o--o{ DAMAGE_REPORT : responsable
    DAMAGE_REPORT |o--o| CHARGE : genera
    LOST_LOAN |o--o| CHARGE : genera
    COST_CATALOG ||--o{ CHARGE : "fija monto"
    STUDENT_REF ||--o{ CHARGE : responsable
    CHARGE ||--o{ CHARGE_AUDIT : audita

    DAMAGE_REPORT {
        bigint id PK
        bigint usage_id FK "unico, opcional"
        string student_code FK "opcional"
        string resource_id
        string resource_snapshot
        string reported_by
        string description
        string evidence_path
        datetime reported_at
        string issue_status "PENDIENTE o RESUELTA"
        string charge_result "GENERADO, SIN_COSTO, SIN_RESPONSABLE, ERROR"
    }
    COST_CATALOG {
        bigint id PK
        string resource_type
        decimal amount
        boolean active
    }
    CHARGE {
        bigint id PK
        bigint damage_report_id FK "opcional"
        bigint lost_loan_id FK "opcional"
        bigint cost_catalog_id FK
        string student_code FK
        decimal amount "congelado"
        string status "PENDIENTE, PAGADO, ANULADO"
        datetime created_at
    }
    CHARGE_AUDIT {
        bigint id PK
        bigint charge_id FK
        string changed_by
        string previous_status
        string new_status
        string reason
        datetime changed_at
    }
```

## 6. Mensajería con Kafka (outbox e inbox)

```mermaid
erDiagram
    OUTBOX_MESSAGE {
        bigint id PK
        string topic
        string event_type
        string message_key
        text payload
        string status "PENDIENTE o PUBLICADO"
        int attempts
        datetime next_retry_at
        datetime created_at
    }
    PROCESSED_EVENT {
        string event_id PK
        string topic
        datetime processed_at
    }
```

Estas dos tablas no se relacionan con las demás: `outbox_message` guarda los eventos hacia M1 y M2 hasta publicarlos, y `processed_event` evita procesar dos veces un evento recibido.
