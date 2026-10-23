# Implementation Plan: Confirmar no asistencia (Módulo 3)

**Date**: 2026-10-23
**Spec**: `spec-confirmar-no-asistencia.md` (reglas aplicables en `reglas-negocio-m3.md`: RN-SAN-01 a RN-SAN-14, RN-SCO-02, RN-SCO-12, RN-INT-04)
**Plan**: 3 de 6 del Módulo 3 — Control de Uso, Sanciones y Analítica (orden de implementación y fundamentos compartidos en `plan-indice.md`)

## Summary

Este plan implementa la confirmación de inasistencia: el personal administrativo constata en sitio que un estudiante no se presentó a la reserva de un espacio, pasados 30 minutos desde el inicio. M3 reporta la inasistencia a M2 y, solo cuando M2 la acepta, crea la sanción de 7 días y descuenta 5 puntos. También permite anular una inasistencia confirmada por error: la sanción se anula, los puntos se restituyen y M2 es informado.

## Technical Context

**Language/Version**: Java 21 (LTS), Spring Boot (versión con soporte vigente al iniciar el desarrollo)
**Primary Dependencies**: Spring Web, Spring Data JPA, Spring Validation, Spring Security (JWT), Spring for Apache Kafka, Flyway (migraciones)
**Storage**: MySQL 8
**Testing**: JUnit 5 + Mockito + Spring Boot Test; Testcontainers (MySQL y Kafka)
**Target Platform**: Linux server, contenedores Docker (backend, frontend, MySQL, Kafka)
**Project Type**: Web application (backend + frontend separados)
**Frontend Framework**: React + Vite (confirmado)
**Message Broker**: Apache Kafka (confirmado)
**Performance Goals**: registrar la confirmación en menos de 2 s (SC-005); listado de pendientes en menos de 3 s (SC-006); marcar sanciones cumplidas dentro de los 5 minutos siguientes a su fin
**Constraints**:
- Una reserva solo admite una confirmación de inasistencia (SC-002); no se sanciona sin acuse de aceptación de M2 (SC-004).
- La confirmación se rechaza antes de los 30 minutos desde el inicio de la reserva o si la reserva está cancelada (SC-003). No hay período máximo de confirmación.
- Registrar la confirmación no depende de que M2 esté disponible: el evento queda en el outbox (SC-007).
- Las sanciones corren en paralelo; los días se cuentan en la zona America/Bogota.
- Anular exige motivo obligatorio, queda en auditoría y no se puede repetir sobre una confirmación ya anulada.

## Module Communication

| Dirección | Módulo | Tópico | Evento | Estado del contrato |
|---|---|---|---|---|
| Publica | M2 | `module3.reservation.no-show.v1` | `NoShowReported`, `NoShowReportVoided` | Propuesto por M2 |
| Consume | M2 | `module2.reservation.no-show-ack.v1` | `NoShowReportAcknowledged` (aceptado, `duplicate`, `TOO_EARLY`, `RESERVATION_CANCELLED`, `ALREADY_RETURNED`) | Por acordar |
| Datos | M2 | (copia local) | Reservas recibidas por `plan-registrar-checkout` | Acordado |

M3 no publica eventos de sanción ni de bloqueo: M2 las conoce preguntando a M3 (`GET /api/v1/compliance/people/{code}`, en `plan-consultar-cumplimiento`).

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan-confirmar-no-asistencia.md                 # Este archivo
├── spec-confirmar-no-asistencia.md
└── reglas-negocio-m3.md
```

### Source Code (archivos de este caso de uso)

```text
backend/src/main/java/com/university/sanctions/
├── presentation/controller/AbsenceController.java
├── business/
│   ├── service/ AbsenceService               # SanctionService y TrustScoreService ya existen (plan 1)
│   └── event/ listeners de AbsenceAccepted y AbsenceAnnulled
├── domain/
│   ├── model/ AbsenceConfirmation, (auditoría de anulación)
│   ├── repository/ AbsenceConfirmationRepository
│   └── event/ AbsenceAccepted, AbsenceAnnulled
└── infrastructure/
    ├── persistence/jpa/ entity, repository, adapter, mapper
    ├── integration/module2/kafka/ NoShowAckConsumer  # publica vía ReservationModulePublisher (plan 1)
    └── scheduling/ SanctionExpiryJob

frontend/src/pages/AbsencePage.jsx
```

**Layered Architecture:** `presentation` recibe solicitudes; `business` coordina el caso de uso; `domain` contiene reglas, modelos y contratos de repositorio; `infrastructure` los implementa con MySQL/JPA y Kafka. Flujo: `presentation → business → domain`, e `infrastructure` implementa los contratos de `domain` y `business`. Los casos de uso se encadenan por eventos de dominio internos, nunca llamando al servicio de otro caso de uso. El detalle completo de capas y carpetas está en `plan-indice.md`.

## Phase 1: Setup

**Purpose**: Inicialización del proyecto

Cubierta por `plan-registrar-checkout` (T001–T005). No hay tareas nuevas en este plan.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: lo que este caso de uso necesita y que construyen otros planes

**⚠️ CRITICAL**: no empezar la Phase 3 hasta tener estos prerrequisitos

De `plan-registrar-checkout`:
- Fases 1 y 2 completas: copia local de reservas (T008, T021), `TrustScoreService` (T022), `SanctionService` y el modelo `Sanction` (T010, T023), bus de eventos (T016), outbox (T017), inbox (T018), Kafka (T019) y `ReservationModulePublisher` (T020).
- No necesita nada de la Phase 3 del check-out.

Este plan no agrega tareas fundacionales propias.

**Checkpoint**: prerrequisitos disponibles — se puede implementar la user story

---

## Phase 3: User Story — Confirmar no asistencia (Priority: P1)

**Goal**: el personal administrativo confirma que un estudiante no se presentó a la reserva de un espacio; M3 lo reporta a M2 y, cuando M2 lo acepta, crea la sanción y descuenta puntos; también se puede anular por error. Spec: `spec-confirmar-no-asistencia.md`.

**Independent Test**: confirmar una inasistencia sobre una reserva de la copia local, simular el acuse de aceptación de M2 y verificar la sanción y el descuento de score.

**Capas afectadas**: presentation (`AbsenceController`), business (`AbsenceService`, `SanctionService`, listeners), domain (`AbsenceConfirmation`), infrastructure (publicador y consumidor de acuse de M2, `SanctionExpiryJob`).

### Tests
- [ ] T001 [P] [INASISTENCIA] Test de contrato para `GET /api/v1/absences/pending`, `POST /api/v1/absences` y `POST /api/v1/absences/{id}/annul`
- [ ] T002 [P] [INASISTENCIA] Test de integración: confirmación aceptada (sanción y −5 puntos), antes de los 30 minutos, reserva cancelada, duplicada, rechazo de M2, sin acuse; anulación de una sanción vigente y de una ya cumplida, sin motivo y ya anulada; sanciones simultáneas; paso a Cumplida
- [ ] T003 [P] [INASISTENCIA] Test de contrato Kafka: `NoShowReported`, `NoShowReportVoided` y `NoShowReportAcknowledged` (aceptado, `duplicate`, `TOO_EARLY`, `RESERVATION_CANCELLED`)

### Implementation
- [ ] T004 [P] [INASISTENCIA] Modelar y migrar `AbsenceConfirmation` (estados Pendiente de acuse, Aceptada, Rechazada por M2, Anulada; instante de la constatación) y su auditoría de anulación
- [ ] T005 [INASISTENCIA] `AbsenceService`: validar con `ReservationCopy` (espacio, sancionable, no cancelada, inicio hace al menos 30 minutos), impedir duplicados y registrar la confirmación "Pendiente de acuse" (depende de T004)
- [ ] T006 [INASISTENCIA] Listado de pendientes: reservas de espacios elegibles sin confirmar y las "Rechazadas por M2" con su motivo
- [ ] T007 [INASISTENCIA] Endpoints `GET /api/v1/absences/pending` y `POST /api/v1/absences`
- [ ] T008 [INASISTENCIA] Escribir `NoShowReported` en el outbox hacia M2 (`verifiedAt` = instante de la constatación), sin bloquear la confirmación
- [ ] T009 [INASISTENCIA] Consumidor de `NoShowReportAcknowledged`: aceptado → evento de dominio `AbsenceAccepted`; rechazado → "Rechazada por M2" con el motivo (y `acceptedFrom` si es `TOO_EARLY`); `duplicate` no crea otra sanción
- [ ] T010 [INASISTENCIA] Listener de `AbsenceAccepted`: crear la sanción de 7 días desde la fecha de registro de la ausencia, descontar 5 puntos una sola vez y marcar la infracción (depende de plan-registrar-checkout T022 y T023)
- [ ] T011 [INASISTENCIA] Anulación (`POST /api/v1/absences/{id}/annul`): motivo obligatorio, confirmación y sanción pasan a Anulada, restitución de puntos (RESTITUCION_ANULACION), exclusión de `absenceCount` y de infracciones, auditoría y `NoShowReportVoided` en el outbox hacia M2
- [ ] T012 [INASISTENCIA] `SanctionExpiryJob` (cada 5 minutos): marcar como Cumplida toda sanción cuya fecha de fin ya pasó
- [ ] T013 [INASISTENCIA] Frontend: `AbsencePage.jsx` (pendientes, confirmar, rechazadas por M2, anular)

**Checkpoint**: Confirmar no asistencia funciona con acuses reales de M2 y la sanción afecta el score

---

## Phase N: Polish & Cross-Cutting Concerns

- [ ] T014 Documentación OpenAPI de los endpoints de inasistencia
- [ ] T015 Prueba de punta a punta con Kafka: confirmar → `NoShowReported` → acuse aceptado → sanción y −5 puntos → anular → `NoShowReportVoided` y restitución; acuse rechazado y reentrega
- [ ] T016 Ajuste de performance contra SC-005 y SC-006
- [ ] T017 Tests unitarios: umbral de 30 minutos, cálculo de fin de sanción en America/Bogota, restitución de puntos sin pasar del máximo, exclusión de `absenceCount`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: cubierta por `plan-registrar-checkout`
- **Foundational (Phase 2)**: prerrequisitos de `plan-registrar-checkout` (fases 1 y 2)
- **Confirmar no asistencia (Phase 3)**: depende de la Phase 2; puede desarrollarse en paralelo con `plan-registrar-checkout` Phase 3 y con `plan-reportar-novedad-tecnica`
- **Polish**: depende de la Phase 3

### Within Each User Story

- Modelos y contratos antes que adaptadores; adaptadores antes que servicios; servicios antes que controladores o listeners
- La sanción se crea solo al procesar el acuse de aceptación, nunca al confirmar
- Tests antes de la implementación

### Relación con otros planes

- `plan-desbloquear-usuario` levanta las sanciones que crea este plan.
- `plan-consultar-cumplimiento` reporta a M2 las sanciones vigentes y el conteo `absenceCount` (aceptadas y no anuladas).

## Notes

- **Pendientes de contrato con M2**: tópicos de acuse y de anulación "por acordar"; qué hace M2 al recibir `NoShowReportVoided`; la ventana y el texto de `reason` de las sanciones que M2 recibe.
- Riesgo abierto: si M2 no avisa el inicio de uso, M3 no sabe si el estudiante llegó; por eso la inasistencia la confirma el personal administrativo y no es automática.
- Verificar tests en verde y hacer commit después de cada tarea o grupo lógico; detenerse en cada Checkpoint.
