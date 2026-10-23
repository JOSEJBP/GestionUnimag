# Implementation Plan: Consultar mi cumplimiento (Módulo 3)

**Date**: 2026-10-23
**Spec**: `spec-consultar-cumplimiento.md` (reglas aplicables en `reglas-negocio-m3.md`: RN-SCO-01, RN-SCO-10, RN-BLO-04, RN-COB-04 a RN-COB-08, RN-INT-07, RN-INT-08)
**Plan**: 5 de 6 del Módulo 3 — Control de Uso, Sanciones y Analítica (orden de implementación y fundamentos compartidos en `plan-indice.md`)

## Summary

Este plan implementa dos consultas sobre los mismos datos. La primera es del estudiante: ve su score, historial, sanciones vigentes, cobros (Pendiente, Pagado, Anulado) y avisos de bloqueo o de préstamo vencido. La segunda es de M2: antes de aceptar una reserva consulta a M3 las restricciones activas del estudiante (sanciones vigentes y bloqueo), sin guardar copia. Ambas se calculan en cada consulta a partir de los datos de M3.

## Technical Context

**Language/Version**: Java 21 (LTS), Spring Boot (versión con soporte vigente al iniciar el desarrollo)
**Primary Dependencies**: Spring Web, Spring Data JPA, Spring Validation, Spring Security (JWT), Spring for Apache Kafka, Flyway (migraciones)
**Storage**: MySQL 8
**Testing**: JUnit 5 + Mockito + Spring Boot Test; Testcontainers (MySQL y Kafka)
**Target Platform**: Linux server, contenedores Docker (backend, frontend, MySQL, Kafka)
**Project Type**: Web application (backend + frontend separados)
**Frontend Framework**: React + Vite (confirmado)
**Message Broker**: Apache Kafka (confirmado)
**Performance Goals**: consulta del estudiante en menos de 3 s (SC-001); respuesta a M2 en menos de 1 s (SC-009); cambios de score, sanción o cobro visibles en menos de 5 s (SC-005)
**Constraints**:
- Cada estudiante solo ve sus propios datos (SC-002); una sanción cumplida nunca aparece como vigente (SC-003).
- Una falla en una sección no afecta a las demás (SC-006).
- La respuesta a M2 se calcula en cada consulta, sin caché, y debe funcionar aunque Kafka esté caído.
- El bloqueo (score 0) se informa a M2 como una sanción sin `endsOn` y con alcance `TODO`.
- Las llamadas de M2 van sin autenticación, en una cadena de seguridad aparte del JWT.

## Module Communication

| Dirección | Módulo | Medio | Detalle | Estado del contrato |
|---|---|---|---|---|
| Expone | M2 | REST `GET /api/v1/compliance/people/{code}` | `activeSanctions[{reason, startsOn, endsOn, scope}]`, `absenceCount`, `lateReturns`, `generatedAt`; estudiante sin historial → 200 vacío | Definido por M2 |

Este caso de uso no usa Kafka. M3 no publica eventos de sanción, bloqueo ni desbloqueo: M2 consulta en cada operación y no guarda copia.

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan-consultar-cumplimiento.md                 # Este archivo
├── spec-consultar-cumplimiento.md
└── reglas-negocio-m3.md
```

### Source Code (archivos de este caso de uso)

```text
backend/src/main/java/com/university/sanctions/
├── presentation/controller/
│   ├── ComplianceController.java                   # estudiante autenticado (JWT)
│   └── external/ModuleComplianceController.java    # M2 → M3, sin JWT
├── business/service/ ComplianceQueryService
├── domain/ (consultas sobre modelos existentes: TrustScore, ScoreHistory, Sanction, Charge, OverdueDeduction)
└── infrastructure/persistence/jpa/ consultas de lectura e índices

frontend/src/pages/CompliancePage.jsx
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
- Fases 1 y 2 completas: `TrustScore`, `ScoreHistory`, `Sanction` (T009, T010), seguridad con la cadena sin JWT (T012).

Secciones que dependen de otros planes (hasta que existan, devuelven vacío):
- Préstamo vencido y `lateReturns`: `plan-registrar-checkout` Phase 3 (`OverdueDeduction`, `CheckOut`).
- Cobros: `plan-reportar-novedad-tecnica` (`Charge`).
- `absenceCount` y sanciones por inasistencia: `plan-confirmar-no-asistencia`.

Este plan no agrega tareas fundacionales propias.

**Checkpoint**: prerrequisitos disponibles — se puede implementar la user story

---

## Phase 3: User Story — Consultar mi cumplimiento (Priority: P1)

**Goal**: el estudiante ve su score, historial, sanciones, cobros y avisos; y M2 consulta a M3 las restricciones activas de un estudiante. Spec: `spec-consultar-cumplimiento.md`.

**Independent Test**: con un estudiante con historial, sanción, cobro y préstamo vencido, consultar `GET /api/v1/me/compliance` y `GET /api/v1/compliance/people/{code}` y verificar que coinciden con los datos de origen.

**Capas afectadas**: presentation (`ComplianceController`, `ModuleComplianceController`), business (`ComplianceQueryService`), domain (consultas sobre entidades existentes), infrastructure (índices y adaptadores de lectura).

### Tests
- [ ] T001 [P] [CUMPLIMIENTO] Test de contrato para `GET /api/v1/me/compliance` y sus secciones, y para `GET /api/v1/compliance/people/{code}` con el JSON esperado por M2
- [ ] T002 [P] [CUMPLIMIENTO] Test de integración: score e historial paginado, sin historial (50/50), sanciones vigentes simultáneas, bloqueo, aviso de préstamo vencido, cobros, falla parcial en una sección, intento de ver datos de otro estudiante
- [ ] T003 [P] [CUMPLIMIENTO] Test de integración de la respuesta a M2: sanciones por inasistencia con alcance `ESPACIOS`, bloqueo como sanción sin `endsOn` con alcance `TODO`, estudiante sin historial (200 vacío), estudiante recién desbloqueado y respuesta sin Kafka disponible

### Implementation
- [ ] T004 [CUMPLIMIENTO] `ComplianceQueryService`: score actual y reglas vigentes, historial paginado, sanciones vigentes, aviso de bloqueo con fecha estimada de la próxima recuperación, aviso por préstamo vencido (días, puntos perdidos y descuento diario) y cobros con su estado
- [ ] T005 [CUMPLIMIENTO] `ComplianceController` para el estudiante autenticado, con secciones independientes para que una falla parcial no bloquee las demás
- [ ] T006 [CUMPLIMIENTO] `ModuleComplianceController`: `GET /api/v1/compliance/people/{code}` con `activeSanctions`, `absenceCount` (aceptadas y no anuladas), `lateReturns` y `generatedAt`; solo sanciones Vigente; calculada en cada consulta, sin caché
- [ ] T007 [CUMPLIMIENTO] Índices y consultas para cumplir <1 s en la respuesta a M2
- [ ] T008 [CUMPLIMIENTO] Frontend: `CompliancePage.jsx` con avisos destacados, tarjetas de sanción, historial y cobros

**Checkpoint**: M2 puede negar reservas según las sanciones y el bloqueo reales de M3, y el estudiante entiende su situación

---

## Phase N: Polish & Cross-Cutting Concerns

- [ ] T009 Documentación OpenAPI de `GET /api/v1/me/compliance` y de la ruta entrante de M2
- [ ] T010 Prueba de contrato con el JSON de ejemplo de M2 (fixture) y verificación de la respuesta sin Kafka disponible
- [ ] T011 Ajuste de performance contra SC-001 (<3 s) y SC-009 (<1 s)
- [ ] T012 Hardening: verificación de que el estudiante nunca accede a datos ajenos y de que la cadena sin JWT solo cubre `GET /api/v1/compliance/**`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: cubierta por `plan-registrar-checkout`
- **Foundational (Phase 2)**: prerrequisitos de `plan-registrar-checkout` (fases 1 y 2)
- **Consultar mi cumplimiento (Phase 3)**: depende de la Phase 2; la consulta de M2 (T006) conviene hacerla primero, con contadores en cero, porque sin ella M2 no puede aplicar sanciones ni bloqueos; se completa a medida que existan los otros planes
- **Polish**: depende de la Phase 3

### Within Each User Story

- Consultas y servicio antes que controladores; ruta de M2 antes que la pantalla del estudiante
- Tests antes de la implementación

### Relación con otros planes

- Es la vista de lo que producen los planes de check-out, novedad, inasistencia y desbloqueo.

## Notes

- **Pendientes de contrato con M2**: semántica de `absenceCount` y `lateReturns` (ventana y qué cuenta), texto de `reason`, y si M2 restringe reservas cuando hay un préstamo vencido (M3 solo lo avisa al estudiante).
- Verificar tests en verde y hacer commit después de cada tarea o grupo lógico; detenerse en cada Checkpoint.
