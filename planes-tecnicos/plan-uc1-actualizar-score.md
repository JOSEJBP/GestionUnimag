# Implementation Plan: Actualizar score de confianza (UC1)

**Date**: 2026-10-05
**Spec**: [actualizar-score-confianza.md](../Especs/actualizar-score-confianza.md)
**Plan general**: [plan.md](../Especs/plan.md)
**Módulo**: Módulo 3 — Sanciones y Cumplimiento
**Planes previos**: plan-uc-realizar-check-out.md — UC1 lo dispara cuando el check-out es `ON_TIME`
**Planes relacionados**: [plan-uc2-calcular-penalizacion.md](./plan-uc2-calcular-penalizacion.md) — lo dispara cuando `afecta_score_confianza = TRUE`; [plan-uc3-bloquear-usuario.md](./plan-uc3-bloquear-usuario.md) — lo recibe cuando el score llega a 0 o sube por encima; notificar-sancion.md — lo recibe en cada cambio

## Summary

UC4 es el **corazón del módulo de sanciones**. Es el caso de uso que mantiene el score de confianza de cada estudiante: lo reduce cuando comete una infracción, lo aumenta cuando cumple, y lo recupera cuando pasa un período limpio. Sin él, los demás casos de uso no tienen efecto: `Calcular penalización` calcularía días de suspensión sin tocar la reputación; `Realizar check-out` registraría devoluciones sin premiar al que cumple; `Bloquear usuario` no tendría disparador.

**Enfoque técnico:**

1. `Actualizar score` **no decide** si una infracción merece sanción. Eso es de `Calcular penalización`. Tampoco decide si el estudiante debe ser bloqueado. Eso es de `Bloquear usuario`. **Solo ejecuta el cambio aritmético del score**, dentro de los límites [0, 50], y lo registra.
2. Expone un **puerto de entrada** (`UpdateTrustScorePort`) con tres operaciones: `reduce`, `increaseForOnTimeCheckOut` y `recoverForCleanPeriod`. Lo llaman directamente (en memoria) `Calcular penalización`, `Realizar check-out` y un `@Scheduled` diario. **No hay Kafka interno**: los tres UCs viven en el mismo JVM y el cambio debe ser atómico con el hecho que lo origina.
3. El score se guarda como **columna** (`trust_score.current_value`) y **cada cambio se registra** en `trust_score_history` (FR-SC-009). La columna se actualiza en la misma transacción que el historial.
4. **Bloquear usuario** se dispara **síncronamente** cuando el score llega a 0 o sube de 0 estando bloqueado (FR-SC-006, FR-SC-007). Es una llamada al puerto de UC5 dentro de la misma transacción.
5. **Notificar sanción** se dispara **asíncronamente** en cada actualización (FR-SC-008). Se publica un evento a la outbox; el `OutboxPublisher` lo lleva a Kafka cuando M2 esté disponible. El cambio de score **no se bloquea** si M2 o Kafka están caídos.
6. **M2 necesita saber** cuándo un usuario es bloqueado/desbloqueado y cuándo cambia su score. M3 **crea los topics** correspondientes y los documenta para que M2 los consuma.
7. El UC tiene **5 user stories**: reducción (US1), aumento por check-out (US2), recuperación por período (US3), disparo de bloqueo/desbloqueo (US4) y consulta (US5). US1–US4 son el núcleo; US5 es la cara visible para el estudiante y el administrador.

## Technical Context

**Language/Version**: Java 21 (backend); JavaScript con React + Vite (frontend)
**Primary Dependencies**: Spring Boot 4.x (Web, Data JPA, Validation, Security OAuth2 Resource Server, Spring for Apache Kafka, Spring Scheduling), Flyway, springdoc-openapi. **Ninguna nueva** respecto a `plan.md`.
**Storage**: MySQL 8. UC4 **escribe** `trust_score`, `trust_score_history`, `outbox_message`. Lee `user_compliance_profile` (si UC5 lo creó) y `sanction` (para el motivo que se notifica).
**Testing**: JUnit 5 + Mockito (dominio y casos de uso), `@WebMvcTest` (controladores de consulta), Testcontainers MySQL (persistencia, transacciones y outbox), Testcontainers Kafka (publicación del evento de notificación), ArchUnit (regla de dependencias entre capas).
**Target Platform**: Servidor Linux con JVM 21; navegador web para el frontend.
**Project Type**: Web: backend y frontend separados (React + Vite).
**Performance Goals**: El score se actualiza en **menos de 1 segundo** desde que se recibe la solicitud (SC-001). Es una transacción local: dos `INSERT`/`UPDATE` en MySQL y una inserción en outbox. El presupuesto sobra.
**Constraints**:
- El score **nunca** es negativo (FR-SC-004) ni supera 50 (FR-SC-005).
- El score **inicia en 50** para todo estudiante nuevo (FR-SC-002).
- Cada actualización dispara **Bloquear usuario** si el score resultante es 0, o si sube de 0 estando bloqueado (FR-SC-006, FR-SC-007).
- Cada actualización dispara **Notificar sanción** (FR-SC-008), de forma asíncrona.
- La zona horaria es `America/Bogota`.
- Los parámetros (período de recuperación, puntos por check-out) son configurables (FR-SC-012, FR-SC-013).
**Scale/Scope**: Mismo orden que M2 (miles de estudiantes). Una pantalla de perfil con el score y su historial; una pantalla de administración para consultar cualquier estudiante.

## Integración con otros módulos

### Lo que M3 consume de M2 y M1 (adoptado tal cual)

**De M2:**

| Evento | Topic | Para qué lo consume M3 | Dónde lo definió M2 |
|---|---|---|---|
| `ReservationRecordCreated` | `module2.reservation.record.v1` | UC1 lo usa para crear la `Usage` local; UC4 no lo consume directamente | `plan-uc2` §6, `plan-uc10` §1 |
| `ReservationCancelled` | `module2.reservation.cancellation.v1` | UC11 del M3 lo consume para saber si una reserva se canceló | `plan-uc3` §7, `plan-uc4` §3 |
| `CheckOutAcknowledged` | `module2.reservation.check-out-ack.v1` | UC1 lo consume para confirmar que M2 cerró el préstamo | `plan-uc12` §2 |
| `NoShowReportAcknowledged` | `module2.reservation.no-show-ack.v1` | UC2 del M3 lo consume para confirmar el reporte | `plan-uc9` §2 |
| `LoanDeclaredLost` | (M2 lo produce, topic pendiente de definir) | UC3 lo consumirá para penalizar la pérdida | `plan-uc12` §2 |

**De M1:** M1 no produce eventos que UC4 consuma. No aplica.

**Estos contratos se adoptan tal cual.** M3 no los redefine.

### Lo que M3 produce para M2 (creado por M3, obligatorio para M2)

M2 necesita saber cuándo un usuario es bloqueado, desbloqueado, o cuándo cambia su score, para:
- Rechazar reservas de usuarios bloqueados sin consultar REST en cada petición.
- Mostrar el score actualizado al estudiante cuando consulta.

**M3 crea estos topics y los documenta para que M2 los consuma:**

| Evento | Topic | Cuándo se emite | Por qué M2 lo necesita | FR |
|---|---|---|---|---|
| `TrustScoreChanged` | `module3.sanctions.trust-score-changed.v1` | Cada vez que el score cambia | M2 muestra el score actualizado al estudiante | FR-SC-008 |
| `UserBlocked` | `module3.sanctions.user-blocked.v1` | Cuando el score llega a 0 | M2 debe rechazar nuevas reservas de ese usuario | FR-SC-006 |
| `UserUnblocked` | `module3.sanctions.user-unblocked.v1` | Cuando el score sube de 0 estando bloqueado | M2 debe permitir reservas de nuevo | FR-SC-007 |

**Estos topics son creados por M3.** M2 los consume cuando esté listo. M3 los publica igual.

### Endpoints que M3 expone para M2

| Endpoint | Método | Para qué | Consumidor |
|---|---|---|---|
| `GET /api/v1/compliance/people/{code}/trust-score` | GET | M2 consulta el score actual de un usuario cuando lo necesita | M2 |

**M3 lo crea. M2 lo consume cuando quiera.** Es idempotente y de solo lectura.

### Endpoints que M3 expone para su propio frontend

| Endpoint | Método | Para qué |
|---|---|---|
| `GET /api/v1/trust-score/me` | GET | El estudiante consulta su score |
| `GET /api/v1/trust-score/me/history` | GET | El estudiante consulta su historial |
| `GET /api/v1/admin/trust-score/{code}` | GET | El administrador consulta el score de cualquier estudiante |
| `GET /api/v1/admin/trust-score/{code}/history` | GET | El administrador consulta el historial de cualquier estudiante |

**Estos endpoints son para el frontend de M3**, no son interfaces con M2. El endpoint que M2 consume es `GET /api/v1/compliance/people/{code}/trust-score`.

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan.md                                    # Plan general del M3
├── actualizar-score-confianza.md               # Spec de este plan
├── bloquear-usuario.md                         # «extend» que dispara
├── notificar-sancion.md                        # «include» que dispara
├── calcular-penalizacion.md                    # Origen de las reducciones
├── realizar-checkout.md                       # Origen del aumento
└── ...                                         # resto de specs
```

### Source Code (repository root)

Solo los archivos que crea o toca este plan. La organización completa está en plan.md.

```text
backend/
├── src/main/java/com/university/sanctions/
│   ├── presentation/
│   │   ├── controller/
│   │   │   ├── TrustScoreController.java           # consulta del estudiante (GET /api/v1/trust-score/me)
│   │   │   ├── AdminTrustScoreController.java      # consulta por administrador
│   │   │   └── ComplianceTrustScoreController.java # endpoint para M2 (GET /api/v1/compliance/people/{code}/trust-score)
│   │   └── dto/
│   │       ├── TrustScoreResponse.java
│   │       ├── TrustScoreHistoryResponse.java
│   │       └── ScoreChangeResponse.java
│   │
│   ├── business/
│   │   ├── service/
│   │   │   └── TrustScoreService.java              # implementa los 3 métodos del puerto
│   │   └── event/
│   │       └── TrustScoreChangedEvent.java         # evento de dominio interno
│   │
│   ├── domain/
│   │   ├── model/
│   │   │   ├── TrustScore.java                     # value object: valor + límites
│   │   │   ├── TrustScoreHistory.java              # registro de un cambio
│   │   │   ├── ScoreChangeType.java                # enum: REDUCTION, CHECKOUT_REWARD, RECOVERY
│   │   │   └── ScoreChangeReason.java              # motivo legible
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   └── UpdateTrustScorePort.java       # puerto de entrada (3 métodos)
│   │   │   └── out/
│   │   │       ├── TrustScoreRepository.java
│   │   │       ├── TrustScoreHistoryRepository.java
│   │   │       ├── BlockUserPort.java              # salida hacia UC5
│   │   │       └── NotifyScoreChangePort.java      # salida hacia UC6 (Notificar sanción)
│   │   └── error/
│   │       ├── InvalidScoreChangeException.java
│   │       └── TrustScoreNotFoundException.java
│   │
│   └── infrastructure/
│       ├── persistence/
│       │   └── jpa/
│       │       ├── entity/
│       │       │   ├── TrustScoreEntity.java
│       │       │   └── TrustScoreHistoryEntity.java
│       │       ├── repository/
│       │       │   ├── JpaTrustScoreRepository.java
│       │       │   └── JpaTrustScoreHistoryRepository.java
│       │       ├── adapter/
│       │       │   ├── TrustScoreRepositoryAdapter.java
│       │       │   └── TrustScoreHistoryRepositoryAdapter.java
│       │       └── mapper/
│       │           ├── TrustScoreMapper.java
│       │           └── TrustScoreHistoryMapper.java
│       ├── messaging/
│       │   ├── kafka/
│       │   │   ├── TrustScoreEventPublisher.java    # publica TrustScoreChanged, UserBlocked, UserUnblocked
│       │   │   └── evento/
│       │   │       ├── TrustScoreChangedEvent.java  # payload Kafka
│       │   │       ├── UserBlockedEvent.java
│       │   │       └── UserUnblockedEvent.java
│       │   └── outbox/
│       │       ├── OutboxMessage.java              # (existe, de UC1)
│       │       ├── OutboxRepository.java           # (existe)
│       │       └── OutboxPublisher.java            # (existe) — se reusa
│       ├── scheduler/
│       │   └── CleanPeriodRecoveryScheduler.java   # @Scheduled diario (US3)
│       └── config/
│           ├── SecurityConfig.java
│           ├── SchedulingConfig.java               # habilita @Scheduled
│           └── UseCasesConfig.java
│
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       ├── V4__trust_score.sql                     # trust_score + trust_score_history
│       └── V5__trust_score_indexes.sql
│
└── src/test/java/com/university/sanctions/
    ├── contract/
    │   ├── TrustScoreControllerTest.java
    │   ├── AdminTrustScoreControllerTest.java
    │   └── ComplianceTrustScoreControllerTest.java  # el endpoint que consume M2
    ├── integration/
    │   ├── TrustScoreIT.java                       # persistencia + transacciones + outbox
    │   ├── TrustScoreRecoveryIT.java               # el batch diario
    │   └── TrustScoreNotificationIT.java           # publicación a Kafka
    └── unit/
        ├── TrustScoreTest.java                     # límites [0, 50]
        ├── TrustScoreServiceTest.java              # los 3 métodos del puerto
        └── CleanPeriodRecoverySchedulerTest.java   # con reloj inyectado

frontend/
└── src/
    ├── pages/
    │   ├── TrustScorePage.jsx                      # perfil del estudiante
    │   └── AdminTrustScorePage.jsx                 # consulta de cualquier estudiante
    ├── components/
    │   ├── TrustScoreRing.jsx                      # anillo circular sobre 50
    │   ├── TrustScoreHistory.jsx                   # lista cronológica
    │   ├── RecoveryProgress.jsx                    # barra de días restantes
    │   └── TrustScoreConsequence.jsx               # texto de consecuencia vigente
    └── services/
        └── trustScoreApi.js                        # cliente HTTP
```

Structure Decision: se respeta la estructura de capas de plan.md (presentation / business / domain / infrastructure). El paquete raíz es com.university.sanctions. La lógica del score vive en domain/model/TrustScore.java (value object con límites) y business/service/TrustScoreService.java (orquesta). Los puertos de salida hacia UC5 y UC6 viven en domain/port/out/ para que el dominio no dependa de las implementaciones. La persistencia, el scheduler y la mensajería viven en infrastructure. El frontend se organiza por páginas y componentes reutilizables.

Decisiones de diseño de este caso de uso
Dónde queda cada FR.

FR	Qué pide	Dónde se implementa
FR-SC-001	Reducir 5 puntos si Calcular penalización y afecta_score_confianza = TRUE	TrustScoreService.reduce(...)
FR-SC-002	Aumentar 2 puntos si check-out a tiempo	TrustScoreService.increaseForOnTimeCheckOut(...)
FR-SC-003	Aumentar 5 puntos si 30 días sin infracciones	TrustScoreService.recoverForCleanPeriod(...) + CleanPeriodRecoveryScheduler
FR-SC-004	Score nunca negativo (mínimo 0)	TrustScore value object (clamp a 0)
FR-SC-005	Score nunca supera 50	TrustScore value object (clamp a 50)
FR-SC-006	Disparar Bloquear usuario si el score queda en 0	TrustScoreService llama a BlockUserPort.block(...) en la misma transacción
FR-SC-007	Disparar Bloquear usuario si sube de 0 estando bloqueado	TrustScoreService llama a BlockUserPort.unblock(...) en la misma transacción
FR-SC-008	Disparar Notificar sanción en cada actualización	TrustScoreService llama a NotifyScoreChangePort.notify(...) → outbox
FR-SC-009	Registrar cada cambio en el historial	TrustScoreHistoryRepository.save(...) en la misma transacción
FR-SC-010	Estudiante consulta su score e historial	TrustScoreController
FR-SC-011	Administrador consulta cualquier estudiante	AdminTrustScoreController
FR-SC-012	Configurar período de recuperación	application.yml + SanctionsProperties.cleanPeriodDays
FR-SC-013	Configurar puntos por check-out	application.yml + SanctionsProperties.checkOutRewardPoints
FR-SC-014	«extend» condicional desde Calcular penalización	PenaltyService llama a UpdateTrustScorePort.reduce(...) solo si afecta_score_confianza = TRUE
FR-SC-015	Componente visual circular sobre 50	TrustScoreRing.jsx
FR-SC-016	Mostrar consecuencia vigente y regla de recuperación	TrustScoreConsequence.jsx
FR-SC-017	Mostrar contador de días restantes para recuperación	RecoveryProgress.jsx
FR-SC-018	Historial visible con evento, fecha, motivo, variación	TrustScoreHistory.jsx
Decisiones de diseño justificadas.

1. Actualizar score no decide, solo ejecuta. Es la decisión central del plan. Calcular penalización decide si hay infracción y cuántos días de suspensión; Actualizar score solo aplica el delta (−5) al score. Esta separación permite cambiar las reglas de penalización sin tocar el score, y cambiar el score sin tocar las reglas. Es coherente con la separación del M2 (cada UC hace una cosa).

2. Comunicación interna por llamada directa en memoria, no por Kafka. Los tres disparadores (Calcular penalización, Realizar check-out, batch diario) viven en el mismo JVM. Una llamada directa al puerto UpdateTrustScorePort es atómica, rápida y simple. Kafka se reserva para comunicación entre módulos (M3 → M2), no dentro de M3.

3. El score es columna + historial, no agregado. Guardar el valor actual en trust_score.current_value es rápido de consultar (SC-001) y el historial se mantiene aparte para auditoría. Si fuera agregado (sumar el historial en cada consulta), la consulta sería O(n) y crecería con el tiempo.

4. Bloqueo/desbloqueo es síncrono, notificación es asíncrona.

Bloquear usuario debe ocurrir en la misma transacción que el cambio de score: si el score llega a 0, el usuario debe quedar bloqueado inmediatamente.

Notificar sanción puede ser asíncrono: al estudiante no le urge saber en el mismo instante; y si M2 está caído, no queremos bloquear el cambio de score.

5. M3 crea los topics que M2 necesita consumir. M2 necesita saber cuándo un usuario es bloqueado, desbloqueado, o cuándo cambia su score. M2 no tiene esos topics definidos. M3 los crea (module3.sanctions.trust-score-changed.v1, module3.sanctions.user-blocked.v1, module3.sanctions.user-unblocked.v1) y los documenta para que M2 los consuma cuando esté listo. M3 los publica igual, aunque M2 todavía no los consuma.

6. M3 crea el endpoint de consulta para M2. M2 necesita saber el score actual de un usuario cuando lo consulta. M3 expone GET /api/v1/compliance/people/{code}/trust-score para que M2 lo consuma. Es de solo lectura e idempotente.

7. Idempotencia por eventId + inbox_message. Los tres disparadores pueden entregar el mismo evento dos veces (Kafka at-least-once, o un reintento). Cada evento trae un eventId. TrustScoreService inserta el eventId en inbox_message en la misma transacción que el cambio de score. Si ya existe, el evento se descarta.

8. La transacción abarca todo. El cambio de score, el registro en el historial, la llamada a BlockUserPort, la inserción en outbox_message y la inserción en inbox_message ocurren en la misma transacción MySQL. Si algo falla, todo se revierte.

9. Outbox para la notificación a M2. Los eventos TrustScoreChanged, UserBlocked y UserUnblocked se insertan en outbox_message en la misma transacción. El OutboxPublisher (@Scheduled, reusado de UC1) los publica a Kafka. Si Kafka o M2 están caídos, el evento espera y se reintenta. El cambio de score no se bloquea.

10. El período de recuperación se cuenta desde la última infracción. El CleanPeriodRecoveryScheduler busca estudiantes cuya última infracción (penalty_history.occurred_at) sea anterior a now - cleanPeriodDays. Si nunca han tenido infracción, no aplica (ya están en 50). Si su score ya está en 50, no aplica (FR-SC-005). Si aplica, +5.

11. El score se inicializa en 50 bajo demanda. M3 no crea estudiantes (los crea M2 o el sistema de identidad). Cuando llega el primer cambio de score para un estudiante sin trust_score, se crea con 50 y se aplica el cambio. Esto cubre FR-SC-002 sin necesidad de un evento de creación de estudiante.

## Contratos

Se aplican las convenciones comunes de M2 (camelCase, MAYUSCULA_CON_GUION_BAJO, nunca null, ISO-8601 con -05:00, RFC 9457). M3 las adopta para sus propios contratos.

### 1. Interfaces con M2 — lo que M3 consume (adoptado tal cual)

Actualizar score no consume eventos directamente. Los consumen otros UCs de M3 (UC1, UC2, UC3). Este UC solo usa datos que ya están en la base de M3 (por ejemplo, penalty_history para el batch de recuperación).

No hay contrato de entrada con M2 en este UC.

### 2. Interfaces con M2 — lo que M3 produce (creado por M3)

M3 crea tres topics para que M2 los consuma. Los payloads siguen el envoltorio de M2 (eventId, type, version, occurredAt, source, data).

Topic 1 — module3.sanctions.trust-score-changed.v1
Evento: TrustScoreChanged

Cuándo se emite: cada vez que el score de un estudiante cambia (reducción, aumento o recuperación).

Payload:

```json
{
  "eventId": "tsc-4b6e82c1-84ef-4b2a-9e01-2c7d8b5f3a64",
  "type": "TrustScoreChanged",
  "version": 1,
  "occurredAt": "2026-10-05T11:15:00-05:00",
  "source": "MODULO_3",
  "data": {
    "studentCode": "2023123456",
    "scoreBefore": 50,
    "scoreAfter": 45,
    "delta": -5,
    "changeType": "REDUCTION",
    "reason": "Infracción por retraso",
    "sourceEventId": "pen-88391"
  }
}
```
Campo	Tipo	Nota
studentCode	string	Código institucional (el mismo code que M2 usa en sus endpoints)
scoreBefore / scoreAfter	int	0–50
delta	int	Negativo para reducción, positivo para aumento
changeType	enum	REDUCTION, CHECKOUT_REWARD, RECOVERY
reason	string	Motivo legible
sourceEventId	string	El eventId del evento que originó el cambio (para trazabilidad)
Clave de partición: studentCode. Así todos los eventos de un mismo estudiante llegan en orden.

Topic 2 — module3.sanctions.user-blocked.v1
Evento: UserBlocked

Cuándo se emite: cuando el score de un estudiante llega a 0.

Payload:

```json
{
  "eventId": "ubl-8c2e1d7a-5b42-4a19-8c0d-2f7e6a1b3c45",
  "type": "UserBlocked",
  "version": 1,
  "occurredAt": "2026-10-05T11:15:00-05:00",
  "source": "MODULO_3",
  "data": {
    "studentCode": "2023123456",
    "reason": "Score de confianza en 0",
    "blockedAt": "2026-10-05T11:15:00-05:00",
    "currentScore": 0
  }
}
```
Clave de partición: studentCode.

Topic 3 — module3.sanctions.user-unblocked.v1
Evento: UserUnblocked

Cuándo se emite: cuando el score de un estudiante sube de 0 estando bloqueado.

Payload:

```json
{
  "eventId": "uub-3f8b6d20-9a14-4e75-b0c8-5d1e7f2a9c46",
  "type": "UserUnblocked",
  "version": 1,
  "occurredAt": "2026-10-06T09:00:00-05:00",
  "source": "MODULO_3",
  "data": {
    "studentCode": "2023123456",
    "reason": "Score recuperado por encima de 0",
    "unblockedAt": "2026-10-06T09:00:00-05:00",
    "currentScore": 2
  }
}
```
Clave de partición: studentCode.

Estos tres topics son creados por M3. M2 los consume cuando esté listo. M3 los publica igual.

### 3. Endpoints que M3 expone para M2

GET /api/v1/compliance/people/{code}/trust-score
M2 consulta el score actual de un usuario cuando lo necesita.

```http
```
GET /api/v1/compliance/people/2023123456/trust-score
```json
{
  "userCode": "2023123456",
  "currentScore": 45,
  "maxScore": 50,
  "minScore": 0,
  "status": "ACTIVE",
  "isBlocked": false,
  "lastUpdatedAt": "2026-10-05T11:15:00-05:00"
}
```
Errores: 404 si el estudiante no existe, 401/403 si no hay token de servicio válido.

### 4. Endpoints que M3 expone para su propio frontend

GET /api/v1/trust-score/me
El estudiante consulta su score actual. Rol ESTUDIANTE o MONITOR. El studentCode sale del JWT.

```json
{
  "studentCode": "2023123456",
  "currentScore": 45,
  "maxScore": 50,
  "minScore": 0,
  "status": "ACTIVE",
  "lastUpdatedAt": "2026-10-05T11:15:00-05:00",
  "consequence": "Puedes reservar con normalidad.",
  "recovery": {
    "ruleActive": true,
    "cleanPeriodDays": 30,
    "daysWithoutInfractions": 12,
    "daysRemaining": 18,
    "nextRecoveryPoints": 5
  }
}
```
GET /api/v1/trust-score/me/history
El estudiante consulta su historial de cambios.

```json
{
  "studentCode": "2023123456",
  "history": [
    {
      "id": "hst-001",
      "changeType": "REDUCTION",
      "points": -5,
      "reason": "Infracción por retraso",
      "scoreBefore": 50,
      "scoreAfter": 45,
      "occurredAt": "2026-10-05T11:15:00-05:00"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "totalPages": 1, "total": 1 }
}
```
GET /api/v1/admin/trust-score/{code} y /history
Igual que los del estudiante, pero para cualquier code. Rol ADMIN o DIRECCION_PROGRAMA.

### 5. Errores

Código HTTP	code	Cuándo
400	INVALID_SCORE_CHANGE	El delta intenta salir de [0, 50] y no es un clamp válido
400	MISSING_EVENT_ID	El evento de entrada no trae eventId
404	TRUST_SCORE_NOT_FOUND	El estudiante no existe
409	EVENT_ALREADY_PROCESSED	El eventId ya está en inbox_message
500	PERSISTENCE_ERROR	Fallo al persistir el cambio
### 6. Puerto de entrada — UpdateTrustScorePort

El contrato que usan los otros UCs del M3. No es REST ni Kafka: es una interfaz Java.

```java
public interface UpdateTrustScorePort {

    TrustScoreChangeResult reduce(String studentCode, int points, String reason, String sourceEventId);

    TrustScoreChangeResult increaseForOnTimeCheckOut(String studentCode, String sourceEventId);

    TrustScoreChangeResult recoverForCleanPeriod(String studentCode, String sourceEventId);
}
TrustScoreChangeResult:

```java
public record TrustScoreChangeResult(
    String studentCode,
    int scoreBefore,
    int scoreAfter,
    int delta,
    ScoreChangeType changeType,
    boolean triggeredBlock,
    boolean triggeredUnblock,
    boolean notificationQueued
) {}
### 7. Puerto de salida — BlockUserPort

Lo llama TrustScoreService para disparar Bloquear usuario (UC5).

```java
public interface BlockUserPort {
    void block(String studentCode, String reason, String sourceEventId);
    void unblock(String studentCode, String reason, String sourceEventId);
}
Se implementa en UC5.

```
### 8. Puerto de salida — NotifyScoreChangePort

Lo llama TrustScoreService para disparar Notificar sanción (UC6).

```java
public interface NotifyScoreChangePort {
    void notify(TrustScoreChangeResult result);
}
Se implementa en UC6, que inserta el evento en outbox_message.

```
### 9. Tablas

trust_score
Columna	Tipo	Nota
student_code	varchar(20) PK	Código institucional
current_value	int NOT NULL	0–50
last_updated_at	timestamp NOT NULL	
created_at	timestamp NOT NULL	
```sql
ALTER TABLE trust_score ADD CONSTRAINT trust_score_range CHECK (current_value BETWEEN 0 AND 50);
trust_score_history
```
Columna	Tipo	Nota
id	varchar(36) PK	
student_code	varchar(20) FK	
change_type	varchar(20) NOT NULL	REDUCTION, CHECKOUT_REWARD, RECOVERY
points	int NOT NULL	
reason	varchar(255) NOT NULL	
score_before	int NOT NULL	
score_after	int NOT NULL	
source_event_id	varchar(36) NOT NULL	Para idempotencia
occurred_at	timestamp NOT NULL	
```sql
CREATE UNIQUE INDEX trust_score_history_source_event ON trust_score_history (source_event_id);
CREATE INDEX trust_score_history_student ON trust_score_history (student_code, occurred_at DESC);
inbox_message y outbox_message
Ya existen de UC1. UC4 los reusa.

```
### 10. Tipos del frontend

```js
export const ScoreChangeType = {
  REDUCTION: "REDUCTION",
  CHECKOUT_REWARD: "CHECKOUT_REWARD",
  RECOVERY: "RECOVERY",
};

export const getMyTrustScore = async () => {
  const response = await fetch("/api/v1/trust-score/me");
  if (!response.ok) throw await response.json();
  return response.json();
};

export const getMyTrustScoreHistory = async (page = 1) => {
  const response = await fetch(`/api/v1/trust-score/me/history?page=${page}`);
  if (!response.ok) throw await response.json();
  return response.json();
};

export const getStudentTrustScore = async (studentCode) => {
  const response = await fetch(`/api/v1/admin/trust-score/${studentCode}`);
  if (!response.ok) throw await response.json();
  return response.json();
};
```
### 11. Fixtures compartidos

```text
backend/src/test/resources/contracts/
├── api-trust-score-me-200.json
├── api-trust-score-me-history-200.json
├── api-trust-score-admin-200.json
├── api-trust-score-404.json
├── api-compliance-trust-score-200.json
├── event-penalty-calculated.json
├── event-check-out-on-time.json
├── event-trust-score-changed.json
├── event-user-blocked.json
└── event-user-unblocked.json
Phase 1: Setup
□ T001 Añadir a application.yml los parámetros: sanctions.trust-score.clean-period-days=30, sanctions.trust-score.checkout-reward-points=2, sanctions.trust-score.reduction-points=5, sanctions.trust-score.recovery-points=5, sanctions.trust-score.max=50, sanctions.trust-score.min=0, y los topics: sanctions.kafka.topic.trust-score-changed=module3.sanctions.trust-score-changed.v1, sanctions.kafka.topic.user-blocked=module3.sanctions.user-blocked.v1, sanctions.kafka.topic.user-unblocked=module3.sanctions.user-unblocked.v1
□ T002 [P] Extender SanctionsProperties con esos parámetros y validarlos al arrancar
□ T003 Habilitar @Scheduled en SchedulingConfig y configurar el cron del job de recuperación (diario a las 03:00 America/Bogota)
Phase 2: Foundational (Blocking Prerequisites)
□ T004 Escribir V4__trust_score.sql con la tabla trust_score y su CHECK de rango
□ T005 Escribir V5__trust_score_indexes.sql con la tabla trust_score_history y sus índices
□ T006 [P] Crear el value object TrustScore con el clamp a [0, 50]
□ T007 [P] Crear TrustScoreHistory, ScoreChangeType y ScoreChangeReason
□ T008 [P] Definir TrustScoreRepository y TrustScoreHistoryRepository en domain/port/out/
□ T009 [P] Definir BlockUserPort y NotifyScoreChangePort en domain/port/out/
□ T010 [P] Crear InvalidScoreChangeException y TrustScoreNotFoundException
□ T011 [P] Crear TrustScoreChangedEvent en business/event/
□ T012 [P] Implementar entidades JPA TrustScoreEntity y TrustScoreHistoryEntity y sus repositorios Spring Data
□ T013 [P] Implementar adaptadores TrustScoreRepositoryAdapter y TrustScoreHistoryRepositoryAdapter con mappers
□ T014 [P] Implementar TrustScoreEventPublisher y los payloads Kafka TrustScoreChangedEvent, UserBlockedEvent, UserUnblockedEvent
□ T015 Verificar que inbox_message y outbox_message existen (de UC1) y son accesibles
Checkpoint: Existe dónde guardar el score, el historial, y los puertos hacia UC5 y UC6

Phase 3: User Story 1 — Reducción automática del score por infracción (Priority: P1)
Goal: Cuando Calcular penalización determine que una infracción afecta el score, Actualizar score reduce 5 puntos, respeta el mínimo de 0, registra el cambio, dispara Bloquear usuario si llega a 0, y encola la notificación a M2.

Tests for User Story 1
□ T016 [P] [US1] Pruebas en TrustScoreTest.java: clamp a 0, clamp a 50, delta 0
□ T017 [P] [US1] Pruebas en TrustScoreServiceTest.java: reducción estándar, reducción a 0 (dispara block), reducción sobre bloqueado, eventId duplicado, estudiante sin score
□ T018 [P] [US1] Prueba TrustScoreIT.java con Testcontainers: transacción atómica, rollback, clave única en source_event_id, CHECK de rango
Implementation for User Story 1
□ T019 [US1] Implementar TrustScoreService.reduce(...) con los 10 pasos de la decisión 8
□ T020 [US1] Implementar el @Transactional en reduce(...)
Checkpoint: La reducción funciona de forma aislada

Phase 4: User Story 2 — Aumento automático del score por check-out a tiempo (Priority: P1)
Tests for User Story 2
□ T021 [P] [US2] Pruebas en TrustScoreServiceTest.java: aumento estándar, aumento en 50 (clamp), aumento desde 0 (dispara unblock), eventId duplicado
□ T022 [P] [US2] Prueba TrustScoreIT.java (extensión): mismas garantías transaccionales
Implementation for User Story 2
□ T023 [US2] Implementar TrustScoreService.increaseForOnTimeCheckOut(...) reusando la lógica
Checkpoint: El aumento funciona de forma aislada

Phase 5: User Story 3 — Recuperación por período sin infracciones (Priority: P2)
Tests for User Story 3
□ T024 [P] [US3] Pruebas en CleanPeriodRecoverySchedulerTest.java: 30 días exactos, 29 días, score en 50, sin infracciones nunca, infracción reciente
□ T025 [P] [US3] Prueba TrustScoreRecoveryIT.java: identificación correcta, registro en historial y outbox, transacción por estudiante, eventId único por día
Implementation for User Story 3
□ T026 [US3] Implementar CleanPeriodRecoveryScheduler con @Scheduled
□ T027 [US3] Implementar TrustScoreService.recoverForCleanPeriod(...) con eventId = recovery-{studentCode}-{date}
Checkpoint: La recuperación batch funciona

Phase 6: User Story 4 — Disparo del bloqueo/desbloqueo (Priority: P1)
Tests for User Story 4
□ T028 [P] [US4] Pruebas en TrustScoreServiceTest.java: reducción a 0 dispara block, aumento desde 0 dispara unblock, recuperación desde 0 dispara unblock
□ T029 [P] [US4] Prueba TrustScoreIT.java (extensión): BlockUserPort se llama en la misma transacción
Implementation for User Story 4
□ T030 [US4] Verificar que T019, T023 y T027 disparan correctamente BlockUserPort. No añade código nuevo.
Checkpoint: Los dos eventos de bloqueo se disparan correctamente

Phase 7: User Story 5 — Consulta y visualización del score (Priority: P3)
Tests for User Story 5
□ T031 [P] [US5] Prueba TrustScoreControllerTest.java con @WebMvcTest, contra los fixtures
□ T032 [P] [US5] Prueba AdminTrustScoreControllerTest.java con @WebMvcTest
□ T033 [P] [US5] Prueba ComplianceTrustScoreControllerTest.java con @WebMvcTest — el endpoint que M2 consume
Implementation for User Story 5
□ T034 [US5] Implementar TrustScoreController con GET /api/v1/trust-score/me y /history
□ T035 [US5] Implementar AdminTrustScoreController con GET /api/v1/admin/trust-score/{code} y /history
□ T036 [US5] Implementar ComplianceTrustScoreController con GET /api/v1/compliance/people/{code}/trust-score — el endpoint para M2
□ T037 [US5] Implementar los DTOs TrustScoreResponse, TrustScoreHistoryResponse, ScoreChangeResponse
□ T038 [P] [US5] Frontend: trustScoreApi.js
□ T039 [P] [US5] Frontend: TrustScoreRing.jsx
□ T040 [P] [US5] Frontend: TrustScoreHistory.jsx
□ T041 [P] [US5] Frontend: RecoveryProgress.jsx
□ T042 [P] [US5] Frontend: TrustScoreConsequence.jsx
□ T043 [US5] Frontend: TrustScorePage.jsx
□ T044 [US5] Frontend: AdminTrustScorePage.jsx
Checkpoint: La consulta está disponible para estudiante, administrador y M2

Phase 8: Polish & Cross-Cutting Concerns
□ T045 [P] Verificar SC-001 (actualización < 1s)
□ T046 [P] Verificar SC-003 (score entre 0 y 50 en el 100% de los casos)
□ T047 [P] Verificar SC-007 (100% de los cambios quedan en el historial)
□ T048 [P] Registrar en logs cada cambio de score
□ T049 [P] Documentar en el README cómo inicializar el score de un estudiante y cómo ver el historial
□ T050 Llevar a pendientes-clarificacion.md los NEEDS CLARIFICATION abiertos en este plan
Dependencies & Execution Order
Phase Dependencies
Setup (Phase 1): depende de plan.md

Foundational (Phase 2): depende de Setup — BLOCKS las user stories

User Story 1 (Phase 3): depende de Foundational

User Story 2 (Phase 4): depende de US1

User Story 3 (Phase 5): depende de US1

User Story 4 (Phase 6): depende de US1, US2 y US3

User Story 5 (Phase 7): depende de Foundational

Polish (Phase 8): depende de todas las US

Dependencias con otros casos de uso del M3
Calcular penalización (UC3): lo dispara cuando afecta_score_confianza = TRUE

Realizar check-out (UC1): lo dispara cuando el check-out es ON_TIME

Bloquear usuario (UC5): lo recibe cuando el score llega a 0 o sube de 0

Notificar sanción (UC6): lo recibe en cada actualización

Dependencias con otros módulos
M2: consume los topics module3.sanctions.trust-score-changed.v1, module3.sanctions.user-blocked.v1 y module3.sanctions.user-unblocked.v1 (creados por M3). Consume el endpoint GET /api/v1/compliance/people/{code}/trust-score (creado por M3).

M1: sin integración en este UC.

Parallel Opportunities
En Foundational: T006 a T015

En cada US: los tests son paralelizables

En US5: los componentes del frontend

Notes
La numeración T0XX es propia de este plan

[P] tasks = different files, no dependencies

[US1] a [US5] = trazabilidad a la user story

La sección Contratos es la única fuente del JSON de UC4

Lo que M3 consume de M1/M2: se adopta tal cual (no se redefine)

Lo que M3 produce para M1/M2: M3 lo crea y lo documenta. M2 lo consume cuando esté listo.

Decisiones propias de M3: Actualizar score no decide, solo ejecuta; comunicación interna por llamada directa; score como columna + historial; bloqueo síncrono, notificación asíncrona; idempotencia por eventId; transacción atómica; outbox para M2; M3 crea los topics que M2 necesita.

```
