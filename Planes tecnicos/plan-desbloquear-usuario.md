# Implementation Plan: Desbloquear usuario (Módulo 3)

**Date**: 2026-10-23
**Spec**: `spec-desbloquear-usuario.md` (reglas aplicables en `reglas-negocio-m3.md`: RN-BLO-01 a RN-BLO-07, RN-SCO-05, RN-SCO-07)
**Plan**: 4 de 6 del Módulo 3 — Control de Uso, Sanciones y Analítica (orden de implementación y fundamentos compartidos en `plan-indice.md`)

## Summary

Este plan implementa las dos salidas del bloqueo. El bloqueo no es un estado propio: un estudiante está bloqueado mientras su score sea 0. El personal administrativo puede desbloquearlo manualmente (sin límite de veces, con motivo obligatorio y auditoría): el score pasa a 5 y las sanciones vigentes se levantan. La otra salida es la recuperación automática de +5 puntos cada 30 días sin infracciones, que aplica a todos los estudiantes. M3 no publica eventos de desbloqueo: M2 lo ve en su siguiente consulta de cumplimiento.

## Technical Context

**Language/Version**: Java 21 (LTS), Spring Boot (versión con soporte vigente al iniciar el desarrollo)
**Primary Dependencies**: Spring Web, Spring Data JPA, Spring Validation, Spring Security (JWT), Spring for Apache Kafka, Flyway (migraciones)
**Storage**: MySQL 8
**Testing**: JUnit 5 + Mockito + Spring Boot Test; Testcontainers (MySQL y Kafka)
**Target Platform**: Linux server, contenedores Docker (backend, frontend, MySQL, Kafka)
**Project Type**: Web application (backend + frontend separados)
**Frontend Framework**: React + Vite (confirmado)
**Message Broker**: Apache Kafka (confirmado)
**Performance Goals**: completar el desbloqueo en menos de 2 s (SC-006)
**Constraints**:
- El desbloqueo y la recuperación automática no pueden sumar puntos dos veces si coinciden (SC-005): control de concurrencia sobre el score.
- Desbloquear es una sola transacción: score, sanciones levantadas, historial (DESBLOQUEO_MANUAL) y auditoría.
- Solo se desbloquea a quien tiene score 0; el motivo es obligatorio y solo lo hace el personal administrativo.
- Si el estudiante tiene un préstamo vencido, el descuento diario sigue aplicando tras el desbloqueo (se avisa al personal).
- El score tras el desbloqueo sale de `system_configuration`; si no está configurado, se rechaza.

## Module Communication

Este caso de uso **no publica ni consume eventos de Kafka** y no expone endpoints a otros módulos.

M2 conoce el resultado del desbloqueo preguntando a M3 en cada operación (`GET /api/v1/compliance/people/{code}`, en `plan-consultar-cumplimiento`): sin sanciones vigentes y con score mayor que 0, el estudiante vuelve a poder reservar.

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan-desbloquear-usuario.md                 # Este archivo
├── spec-desbloquear-usuario.md
└── reglas-negocio-m3.md
```

### Source Code (archivos de este caso de uso)

```text
backend/src/main/java/com/university/sanctions/
├── presentation/controller/UnblockController.java
├── business/service/ UnblockService, RecoveryService    # TrustScoreService y SanctionService ya existen (plan 1)
├── domain/
│   ├── model/ UnblockAudit
│   └── repository/ UnblockAuditRepository
└── infrastructure/
    ├── persistence/jpa/ entity, repository, adapter, mapper
    └── scheduling/ RecoveryJob

frontend/src/pages/BlockedStudentsPage.jsx
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
- Fases 1 y 2 completas: `TrustScoreService` con control de concurrencia y marca de infracción (T022), `SanctionService` y el modelo `Sanction` (T010, T023), parámetros de `SystemConfiguration` (T006), seguridad (T012).

Recomendado: tener `plan-confirmar-no-asistencia` y `plan-registrar-checkout` Phase 3 para probar el desbloqueo con sanciones reales y con el bloqueo del día 7; sin ellos se prueba con datos sembrados.

Este plan no agrega tareas fundacionales propias.

**Checkpoint**: prerrequisitos disponibles — se puede implementar la user story

---

## Phase 3: User Story — Desbloquear usuario (Priority: P1)

**Goal**: el personal administrativo desbloquea manualmente a un estudiante con score 0, y la recuperación automática de puntos es la otra salida del bloqueo. Spec: `spec-desbloquear-usuario.md`.

**Independent Test**: con un estudiante en score 0 y dos sanciones vigentes, desbloquearlo con motivo y verificar score 5, sanciones levantadas, historial y auditoría.

**Capas afectadas**: presentation (`UnblockController`), business (`UnblockService`, `RecoveryService`), domain (`UnblockAudit`), infrastructure (`RecoveryJob`).

### Tests
- [ ] T001 [P] [DESBLOQUEO] Test de contrato para `GET /api/v1/blocked-students` y `POST /api/v1/students/{code}/unblock`
- [ ] T002 [P] [DESBLOQUEO] Test de integración: desbloqueo con y sin sanciones vigentes, estudiante no bloqueado, sin motivo, sin permisos, desbloqueos repetidos, parámetro de score no configurado, nueva sanción tras el desbloqueo y recuperación automática simultánea sin duplicar puntos
- [ ] T003 [P] [DESBLOQUEO] Test de integración de la recuperación: +5 puntos cada 30 días sin infracciones, reinicio del conteo con una infracción, tope en el máximo, salida del bloqueo

### Implementation
- [ ] T004 [P] [DESBLOQUEO] Modelar y migrar `UnblockAudit` (estudiante, responsable, fecha, motivo, score antes y después, sanciones levantadas)
- [ ] T005 [DESBLOQUEO] `UnblockService`: en una sola transacción, fijar el score en el valor configurado (5), levantar las sanciones vigentes sin cambiar el score, registrar DESBLOQUEO_MANUAL y la auditoría; sin límite de desbloqueos (depende de T004)
- [ ] T006 [DESBLOQUEO] Endpoints `GET /api/v1/blocked-students` (fecha de bloqueo, última sanción, días sin infracciones, sanciones que se levantarían) y `POST /api/v1/students/{code}/unblock`; aviso al personal si el estudiante tiene un préstamo vencido
- [ ] T007 [DESBLOQUEO] `RecoveryService` y `RecoveryJob` (diario): +5 puntos a quien lleva 30 días sin infracciones, con control de concurrencia frente al desbloqueo manual (FR-011)
- [ ] T008 [DESBLOQUEO] Frontend: `BlockedStudentsPage.jsx` (listado, consecuencias y desbloqueo con motivo)

**Checkpoint**: las dos salidas del bloqueo (manual y automática) funcionan sin duplicar puntos

---

## Phase N: Polish & Cross-Cutting Concerns

- [ ] T009 Documentación OpenAPI de los endpoints de desbloqueo
- [ ] T010 Prueba de punta a punta: préstamo vencido hasta el día 7 → bloqueo → desbloqueo manual → consulta de M2 sin restricciones → nuevo descuento diario; desbloqueo concurrente con la recuperación automática
- [ ] T011 Ajuste de performance contra SC-006 (<2 s)
- [ ] T012 Tests unitarios: recuperación cada 30 días sin infracciones, reinicio del conteo, tope en el máximo, salida del bloqueo

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: cubierta por `plan-registrar-checkout`
- **Foundational (Phase 2)**: prerrequisitos de `plan-registrar-checkout` (fases 1 y 2)
- **Desbloquear usuario (Phase 3)**: depende de la Phase 2
- **Polish**: depende de la Phase 3

### Within Each User Story

- Modelo y contrato de auditoría antes que el adaptador; servicio antes que controlador y job
- El desbloqueo y la recuperación comparten el control de concurrencia sobre el score
- Tests antes de la implementación

### Relación con otros planes

- `RecoveryJob` usa `lastInfractionAt`, que marcan el descuento por retraso (`plan-registrar-checkout`), la inasistencia aceptada (`plan-confirmar-no-asistencia`) y el reporte de daño (`plan-reportar-novedad-tecnica`).
- `plan-consultar-cumplimiento` es lo que hace visible el desbloqueo a M2.

## Notes

- No existe la función "bloquear usuario" manual: el bloqueo solo ocurre cuando el score llega a 0.
- Los desbloqueos manuales no tienen límite; cada uno queda en la auditoría.
- Pendiente con M2: confirmar que no necesita un evento de desbloqueo y que no guarda copia de las restricciones.
- Verificar tests en verde y hacer commit después de cada tarea o grupo lógico; detenerse en cada Checkpoint.
