# Implementation Plan: Registrar check-out (Módulo 3)

**Date**: 2026-10-23
**Spec**: `spec-registrar-checkout.md` (reglas aplicables en `reglas-negocio-m3.md`: RN-SCO-03, RN-SCO-11, RN-BLO-02, RN-INT-01 a RN-INT-09)
**Plan**: 1 de 6 del Módulo 3 — Control de Uso, Sanciones y Analítica (orden de implementación y fundamentos compartidos en `plan-indice.md`)

## Summary

Este plan implementa el registro de la devolución de un recurso o espacio, sus consecuencias de score y el descuento diario por préstamos vencidos. Un check-out a tiempo y sin daño suma puntos; uno con daño dispara el caso de uso *Reportar novedad técnica* (`«extend»`); un préstamo de activo sin devolver recibe un descuento diario hasta llegar a score 0 el día 7. Cada check-out se informa a M1 (`reserva.finalizada`, solo si no hay daño) y a M2 (`CheckOutRegistered`), y M3 concilia el resultado con el acuse de M2.

**Este es el primer plan que se implementa: contiene el Setup y los fundamentos compartidos** (configuración, copia local de reservas, score, sanciones, seguridad, bus de eventos, outbox, inbox y Kafka) que usan los otros cinco planes.

## Technical Context

**Language/Version**: Java 21 (LTS), Spring Boot (versión con soporte vigente al iniciar el desarrollo)
**Primary Dependencies**: Spring Web, Spring Data JPA, Spring Validation, Spring Security (JWT), Spring for Apache Kafka, Flyway (migraciones)
**Storage**: MySQL 8
**Testing**: JUnit 5 + Mockito + Spring Boot Test; Testcontainers (MySQL y Kafka)
**Target Platform**: Linux server, contenedores Docker (backend, frontend, MySQL, Kafka)
**Project Type**: Web application (backend + frontend separados)
**Frontend Framework**: React + Vite (confirmado)
**Message Broker**: Apache Kafka (confirmado)
**Performance Goals**: registrar el check-out en menos de 2 s (SC-006), sin depender de que M1 o M2 estén disponibles; descuento diario ejecutado por un proceso por lote cada hora
**Constraints**:
- El score se mantiene siempre entre 0 y el máximo configurado (50); el bloqueo se deriva de score = 0 y nunca impide devolver.
- Una utilización solo admite un check-out exitoso, incluso con solicitudes simultáneas (SC-004).
- Publicar a Kafka nunca revierte un check-out ya guardado: el evento queda en el outbox y se reintenta (SC-005).
- La entrega de M2 es *at-least-once*: los eventos reentregados se procesan una sola vez (SC-009).
- El descuento diario es idempotente: máximo un descuento por estudiante y día, aunque el proceso se repita (SC-010).
- Ningún valor de negocio va fijo en el código: todo sale de `system_configuration`.

## Module Communication

**Regla de oro**: M3 se comunica con M1 y M2 únicamente por Kafka.

| Dirección | Módulo | Tópico | Evento | Estado del contrato |
|---|---|---|---|---|
| Publica | M1 | `unimag.m3.uso-novedades.v1` | `reserva.finalizada` (solo sin daño) | Tópico definido; payload en blanco |
| Publica | M2 | `module3.reservation.check-out.v1` | `CheckOutRegistered` (en espacios con `verdict`) | Propuesto por M2 |
| Consume | M2 | `module2.reservation.record.v1` | `ReservationRecordCreated`, `LoanDeclaredLost` | Acordado / por acordar |
| Consume | M2 | `module2.reservation.cancellation.v1` | `ReservationCancelled` | Acordado |
| Consume | M2 | `module2.reservation.check-out-ack.v1` | `CheckOutAcknowledged` | Por acordar |

Garantías (aplican a todo el módulo y se construyen aquí): **outbox** (el evento se guarda en la misma transacción del cambio de negocio y se publica después, con reintentos), **inbox** por `eventId` (se descarta lo ya procesado; el *offset* se confirma después de guardar), **DLT** `module3.dlt.<tópico>` para eventos mal formados, y dos estilos de sobre (M2: `eventId/type/version/occurredAt/data`, clave `reservation_id`; M1: cabeceras `X-Correlation-Id`, `X-Event-Type`, `X-Source-Module`, `X-Event-Version`, clave `recurso_id`).

M3 no llama por REST a M1 ni a M2. M1 cambia el estado del recurso escuchando los eventos de M3.

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan-registrar-checkout.md                 # Este archivo
├── spec-registrar-checkout.md
└── reglas-negocio-m3.md
```

### Source Code (archivos de este caso de uso)

```text
backend/src/main/java/com/university/sanctions/
├── presentation/controller/CheckOutController.java
├── business/
│   ├── service/ CheckOutService, OverdueLoanService, TrustScoreService, SanctionService,
│   │            ReservationMirrorService
│   ├── integration/ UsageEventsPublisher, ReservationModulePublisher
│   └── event/ listeners de CheckOutRecorded (score, M1, M2)
├── domain/
│   ├── model/ CheckOut, OverdueDeduction, LostLoan, ReservationCopy, TrustScore, ScoreHistory,
│   │          Sanction, StudentRef, SystemConfiguration
│   ├── repository/ (un contrato por modelo)
│   └── event/ CheckOutRecorded, LoanDeclaredLost, ScoreChanged
└── infrastructure/
    ├── persistence/jpa/ entity, repository, adapter, mapper
    ├── integration/module1/kafka/ UsageEventsKafkaPublisher
    ├── integration/module2/kafka/ ReservationModuleKafkaPublisher, ReservationRecordConsumer,
    │                              CancellationConsumer, CheckOutAckConsumer, LostLoanConsumer
    ├── messaging/ kafka, outbox, inbox
    ├── scheduling/ OverdueLoanJob, AckResendJob
    └── config/ seguridad (dos cadenas), configuración técnica

frontend/src/pages/CheckOutPage.jsx   (+ services/ y hooks/ compartidos)
```

**Layered Architecture:** `presentation` recibe solicitudes; `business` coordina el caso de uso; `domain` contiene reglas, modelos y contratos de repositorio; `infrastructure` los implementa con MySQL/JPA y Kafka. Flujo: `presentation → business → domain`, e `infrastructure` implementa los contratos de `domain` y `business`. Los casos de uso se encadenan por eventos de dominio internos, nunca llamando al servicio de otro caso de uso. El detalle completo de capas y carpetas está en `plan-indice.md`.

## Phase 1: Setup

**Purpose**: Inicialización del proyecto y estructura básica

- [ ] T001 Crear proyecto backend (Spring Boot con versión soportada vigente, Maven, Java 21) con la estructura de paquetes de `Project Structure`
- [ ] T002 Crear proyecto frontend (React + Vite) con la estructura de `Project Structure`
- [ ] T003 Configurar linting y formato: Spotless/Checkstyle (backend), ESLint + Prettier (frontend)
- [ ] T004 Configurar `docker-compose.yml` con MySQL, Kafka, backend y frontend para desarrollo local
- [ ] T005 Configurar Flyway y el primer script de migración vacío

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Infraestructura y entidades compartidas que TODOS los casos de uso necesitan

**⚠️ CRITICAL**: ningún caso de uso puede implementarse hasta que esta fase esté completa

**Capas afectadas**: las cuatro. **Prueba de punta a punta**: publicar en Kafka un `ReservationRecordCreated` de ejemplo y verificar que queda la copia local; reentregarlo y verificar que no se duplica.

- [ ] T006 Modelar y migrar `SystemConfiguration` con los parámetros: score inicial y máximo (50), puntos por entrega a tiempo (+2), descuento por sanción (−5), días de retraso hasta llegar a 0 (7), duración de la inasistencia (7 días), minutos para considerar la inasistencia (30), días y puntos de recuperación (30 y +5), score tras desbloqueo (5), porcentaje de cobro por préstamo perdido (50%)
- [ ] T007 [P] Modelar y migrar `StudentRef` (código institucional, nombre): referencia al estudiante, no ficha maestra
- [ ] T008 [P] Modelar y migrar `ReservationCopy`: identificador de M2, origen, si es sancionable, recurso (id, nombre, categoría ACTIVO/ESPACIO, tipo, ubicación), titular (código, nombre), ocupación (inicio, fin), préstamo (retiro, fecha límite), estado Activa/Cancelada y fechas del último evento aplicado
- [ ] T009 [P] Modelar y migrar `TrustScore` (con `lastInfractionAt`, `lastRecoveryAt` y versión para bloqueo optimista) y `ScoreHistory` (tipos AUMENTO_CHECKOUT, REDUCCION_SANCION, REDUCCION_RETRASO, RECUPERACION, DESBLOQUEO_MANUAL, RESTITUCION_ANULACION; restricción única por origen para idempotencia)
- [ ] T010 [P] Modelar y migrar `Sanction` (tipo INASISTENCIA; estados Vigente, Cumplida, Levantada, Anulada; fechas de inicio y fin; confirmación de origen)
- [ ] T011 Definir las interfaces de repositorio de dominio de las entidades anteriores y sus adaptadores JPA con mappers (`domain/repository`, `infrastructure/persistence/jpa`)
- [ ] T012 Implementar seguridad: Spring Security + JWT con roles ESTUDIANTE y PERSONAL_ADMINISTRATIVO para el frontend, y una cadena separada sin JWT para `GET /api/v1/compliance/**` y `GET /api/v1/novedades/**`
- [ ] T013 Configurar routing base de la API y manejador global de excepciones con un código de error por cada motivo de rechazo de los specs
- [ ] T014 Configurar logging estructurado con correlation-id, propagado en las cabeceras Kafka hacia M1 (`X-Correlation-Id`)
- [ ] T015 Configurar variables de entorno por perfil (dev/test/prod) y la zona horaria America/Bogota
- [ ] T016 Implementar el bus de eventos de dominio interno (`ApplicationEventPublisher` + `@TransactionalEventListener` después del commit, con transacción propia en cada listener y marca de inconsistencia para reintento si falla; RN-SAN-13, RN-SCO-09) — reemplaza las flechas `«extend»`/`«include»` entre casos de uso
- [ ] T017 Implementar el outbox: tabla `outbox_message` y publicador con reintentos y espera creciente, sin duplicados (`infrastructure/messaging/outbox`)
- [ ] T018 Implementar el inbox: tabla `processed_event` por `eventId`, base de consumidor idempotente con *offset* confirmado después de guardar, y DLT `module3.dlt.<tópico>` (`infrastructure/messaging/inbox`)
- [ ] T019 Configurar Kafka: tópicos de la sección *Module Communication*, serialización JSON, y los dos estilos de sobre (M2: `eventId/type/version/occurredAt/data` con clave `reservation_id`; M1: cabeceras estándar con clave `recurso_id`)
- [ ] T020 Definir en `business/integration` las interfaces `UsageEventsPublisher` (M1) y `ReservationModulePublisher` (M2), y sus implementaciones Kafka que escriben en el outbox
- [ ] T021 Implementar los consumidores de `ReservationRecordCreated` y `ReservationCancelled` (M2) y `ReservationMirrorService`: crean y actualizan la copia local, ordenan por `occurredAt`/`cancelledAt`, ignoran reservas académicas o no sancionables (RN-INT-01, RN-INT-02)
- [ ] T022 Implementar `TrustScoreService`: aplicar un cambio con límites 0 y máximo, historial con score antes y después, idempotencia por origen, control de concurrencia, marca de infracción y bloqueo derivado de score = 0 (RN-SCO-01 a RN-SCO-09, RN-BLO-01)
- [ ] T023 Implementar `SanctionService`: crear sanción de inasistencia (varias pueden estar vigentes a la vez, RN-SAN-07), anular, levantar y marcar como cumplida (RN-SAN-03 a RN-SAN-06)
- [ ] T024 [P] Pruebas de contrato de los consumidores con los JSON de ejemplo de M2, y prueba de idempotencia (reentrega) y de orden (cancelación que llega antes que la ficha)

**Checkpoint**: infraestructura lista — las user stories pueden implementarse en paralelo a partir de aquí

---

## Phase 3: User Story — Registrar check-out (Priority: P1)

**Goal**: registrar la devolución de un recurso o espacio, aplicar sus consecuencias de score, informar a M1 y a M2, y descontar puntos mientras un préstamo sigue sin devolverse. Spec: `spec-registrar-checkout.md`.

**Independent Test**: con una reserva en la copia local, registrar el check-out y verificar la fecha de devolución, la utilización finalizada y los eventos publicados en el outbox (sin necesidad de que existan los demás casos de uso).

**Capas afectadas**: presentation (`CheckOutController`), business (`CheckOutService`, `OverdueLoanService`, listeners), domain (`CheckOut`, `OverdueDeduction`, `LostLoan`), infrastructure (publicadores y consumidores Kafka, `OverdueLoanJob`).

### Tests
- [ ] T025 [P] [CHECKOUT] Test de contrato para `POST /api/v1/checkouts` y `GET /api/v1/me/usages/active`
- [ ] T026 [P] [CHECKOUT] Test de integración: check-out a tiempo, con retraso y con daño (activo y espacio); rechazos (utilización finalizada, sin utilización activa, reserva cancelada, reserva aún no recibida de M2); solicitudes simultáneas; devolución por un estudiante sancionado o bloqueado
- [ ] T027 [P] [CHECKOUT] Test de integración del descuento diario: día 1, día 7 (score 0 y bloqueo), devolución a mitad del retraso, retraso menor a un día, varios préstamos vencidos (un solo descuento por día) y repetición del proceso sin doble descuento
- [ ] T028 [P] [CHECKOUT] Test de contrato Kafka: `CheckOutRegistered` (activo sin campos de daño; espacio con `verdict`), `reserva.finalizada` solo sin daño, y acuses aceptado, `SLOT_NOT_FINISHED` y otros rechazos

### Implementation
- [ ] T029 [P] [CHECKOUT] Modelar y migrar `CheckOut` (resultado A_TIEMPO/CON_RETRASO, minutos de retraso, indicador de daño, estado del acuse; restricción única por reserva), `OverdueDeduction` (única por reserva y día, y por estudiante y fecha) y `LostLoan`
- [ ] T030 [CHECKOUT] `CheckOutService`: identificar la utilización activa en `ReservationCopy` (iniciada, no cancelada, sin check-out), tomar la fecha límite (vencimiento del préstamo en activos; fin de la ocupación en espacios), clasificar a tiempo o con retraso (solo activos) y finalizar la utilización (depende de T029)
- [ ] T031 [CHECKOUT] Endpoints `POST /api/v1/checkouts` (con el daño embebido en la solicitud como `«extend»`) y `GET /api/v1/me/usages/active`
- [ ] T032 [CHECKOUT] Manejo de concurrencia: restricción única sobre la reserva más bloqueo optimista para que solo un check-out simultáneo tenga éxito
- [ ] T033 [CHECKOUT] Publicar el evento de dominio `CheckOutRecorded` (con resultado, indicador de daño y, cuando lo hay, la descripción y la evidencia que el estudiante o el personal informó)
- [ ] T034 [CHECKOUT] Listener de score: +2 puntos si el check-out es a tiempo y sin daño (depende de T022)
- [ ] T035 [CHECKOUT] Listener hacia M1: escribir `reserva.finalizada` en el outbox solo cuando no hay daño (si hay daño, lo publica el plan `plan-reportar-novedad-tecnica` como `reporte.danos`)
- [ ] T036 [CHECKOUT] Listener hacia M2: escribir `CheckOutRegistered` en el outbox (en un espacio, `verdict` `REQUIERE_MANTENIMIENTO` si hubo daño o `SIN_NOVEDAD`)
- [ ] T037 [CHECKOUT] Consumidor de `CheckOutAcknowledged`: guardar el estado del acuse; ante `SLOT_NOT_FINISHED` programar el reenvío a la hora `acceptedFrom` (`AckResendJob`); ante otro rechazo marcar el check-out para revisión sin revertirlo
- [ ] T038 [CHECKOUT] `OverdueLoanService` y `OverdueLoanJob` (cada hora, idempotente): para cada estudiante con préstamos de activos vencidos y sin check-out, aplicar un solo descuento por día — el del préstamo más antiguo, una séptima parte del score que tenía al vencer, redondeada hacia arriba, y el día 7 lo que falte — con historial REDUCCION_RETRASO y marca de infracción en el primer descuento (RN-SCO-11)
- [ ] T039 [CHECKOUT] Consumidor de `LoanDeclaredLost` (M2): guardar `LostLoan` y publicar el evento de dominio `LoanDeclaredLost` (el cobro lo genera `plan-reportar-novedad-tecnica`)
- [ ] T040 [CHECKOUT] Respuesta al estudiante con el resultado (a tiempo o con retraso) y la consecuencia aplicada, y mensajes de rechazo por motivo (FR-014, FR-015)
- [ ] T041 [CHECKOUT] Frontend: `CheckOutPage.jsx` con indicación de daño y descripción obligatoria cuando se marca

**Checkpoint**: Registrar check-out funciona de forma aislada y deja en el outbox los eventos hacia M1 y M2

---

## Phase N: Polish & Cross-Cutting Concerns

- [ ] T042 Documentación OpenAPI de `POST /api/v1/checkouts` y `GET /api/v1/me/usages/active`
- [ ] T043 Prueba de punta a punta con Kafka (Testcontainers) simulando a M1 y M2: reserva → check-out → acuse; préstamo sin devolver → descuento diario → día 7 → pérdida declarada; reentregas y caída de Kafka
- [ ] T044 Ajuste de performance del check-out contra SC-006 (<2 s)
- [ ] T045 Hardening: límite de peticiones en check-out y validación de entrada
- [ ] T046 Observabilidad: métricas de mensajes pendientes en el outbox, eventos en DLT, check-outs marcados para revisión e inconsistencias de score; guía de operación del DLT
- [ ] T047 Documentar los contratos de eventos con ejemplos JSON (M1 y M2) para sentarse con ambos equipos
- [ ] T048 Tests unitarios de reglas: descuento proporcional con redondeo hacia arriba, descuento del día 7, límites del score

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias
- **Foundational (Phase 2)**: depende de Setup — bloquea este plan y los otros cinco
- **Registrar check-out (Phase 3)**: depende de Foundational
- **Polish**: depende de la Phase 3

### Qué desbloquea este plan

- `plan-reportar-novedad-tecnica` consume `CheckOutRecorded` y `LoanDeclaredLost`, y necesita `CheckOut` (T029).
- `plan-confirmar-no-asistencia`, `plan-desbloquear-usuario`, `plan-consultar-cumplimiento` y `plan-consultar-analitica-cumplimiento` solo necesitan la Phase 2 (y T021/T022/T023 en particular).

### Within Each User Story

- `domain/model/` y `domain/repository/` antes que los adaptadores de `infrastructure/persistence/`
- Adaptadores antes que `business/service/`; servicios antes que controladores o listeners
- Los eventos hacia M1 y M2 se escriben en el outbox al final de la transacción del servicio
- Tests antes de la implementación

## Notes

- Los eventos de dominio (`CheckOutRecorded`, `LoanDeclaredLost`) son internos al backend y encadenan este caso de uso con *Reportar novedad técnica* (la flecha `«extend»` del diagrama).
- **Pendientes de contrato con M1 y M2** que afectan este plan: payload de `reserva.finalizada` y si una devolución tardía sin daño también la usa; los tópicos "por acordar" de M2 (acuse del check-out y `LoanDeclaredLost`); si M2 avisará a M3 el inicio de uso de una reserva.
- Si M1 no recibe `reserva.finalizada` por falla de publicación, el evento queda en el outbox y se reenvía; el check-out del estudiante no se revierte.
- Verificar tests en verde y hacer commit después de cada tarea o grupo lógico; detenerse en cada Checkpoint.
