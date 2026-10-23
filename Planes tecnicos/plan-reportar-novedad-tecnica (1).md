# Implementation Plan: Reportar novedad técnica y gestionar cobros (Módulo 3)

**Date**: 2026-10-23
**Spec**: `spec-reportar-novedad-tecnica.md` (reglas aplicables en `reglas-negocio-m3.md`: RN-COB-01 a RN-COB-12, RN-SCO-05, RN-INT-05, RN-INT-06)
**Plan**: 2 de 6 del Módulo 3 — Control de Uso, Sanciones y Analítica (orden de implementación y fundamentos compartidos en `plan-indice.md`)

## Summary

Este plan implementa el registro de un daño sobre un recurso o espacio (durante el check-out o de forma independiente, a cargo del personal administrativo), el aviso a M1 para que lo pase a mantenimiento, la generación y gestión de cobros (por daño y por préstamo perdido) y la respuesta a M1 sobre si un recurso tiene novedades pendientes. El reporte se guarda como "Pendiente" y pasa a "Resuelta" cuando M1 informa que el recurso salió de mantenimiento; M3 nunca asume que el estado del recurso cambió.

## Technical Context

**Language/Version**: Java 21 (LTS), Spring Boot (versión con soporte vigente al iniciar el desarrollo)
**Primary Dependencies**: Spring Web, Spring Data JPA, Spring Validation, Spring Security (JWT), Spring for Apache Kafka, Flyway (migraciones)
**Storage**: MySQL 8
**Testing**: JUnit 5 + Mockito + Spring Boot Test; Testcontainers (MySQL y Kafka)
**Target Platform**: Linux server, contenedores Docker (backend, frontend, MySQL, Kafka)
**Project Type**: Web application (backend + frontend separados)
**Frontend Framework**: React + Vite (confirmado)
**Message Broker**: Apache Kafka (confirmado)
**Performance Goals**: registrar el reporte en menos de 2 s (SC-005); responder a M1 la consulta de novedades pendientes en menos de 2 s (SC-006)
**Constraints**:
- Máximo un reporte de daño por utilización (SC-002); la descripción es obligatoria (SC-004).
- No hay transacción que abarque MySQL y M1: el reporte y su evento en el outbox se guardan juntos; M1 aplica el estado por su lado.
- Un daño no se informa a M2: va solo de M3 a M1.
- Los cobros "Pagado" y "Anulado" no se pueden modificar; todo cambio de estado queda en la auditoría (SC-008).
- La consulta de M1 responde aunque Kafka esté caído; las llamadas entre módulos van sin autenticación, en una cadena de seguridad aparte del JWT.

## Module Communication

| Dirección | Módulo | Medio | Detalle | Estado del contrato |
|---|---|---|---|---|
| Publica | M1 | Kafka `unimag.m3.uso-novedades.v1` | `reporte.danos` (un evento por check-out con daño o por reporte independiente) | Tópico definido; payload en blanco |
| Consume | M1 | Kafka `unimag.m1.recursos-estados.v1` | `estado.actualizado` → resolver novedades al salir de mantenimiento | Definido por M1 (falta confirmar que se emite al salir de mantenimiento) |
| Expone | M1 | REST `GET /api/v1/novedades/pendientes?recursoId={id}` | `{ "recursoId", "tieneNovedadPendiente" }`; recurso desconocido → `false` | Definido por M1 |
| Interno | — | Eventos de dominio | Recibe `CheckOutRecorded` (con daño) y `LoanDeclaredLost` del plan `plan-registrar-checkout` | — |

M3 no avisa del daño a M2 ni cambia el estado del recurso por REST: el `PATCH` de estado de M1 es solo para personas.

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan-reportar-novedad-tecnica.md                 # Este archivo
├── spec-reportar-novedad-tecnica.md
└── reglas-negocio-m3.md
```

### Source Code (archivos de este caso de uso)

```text
backend/src/main/java/com/university/sanctions/
├── presentation/controller/
│   ├── DamageReportController.java, ChargeController.java, AdminConfigController.java
│   └── external/PendingIssuesController.java        # M1 → M3, sin JWT
├── business/
│   ├── service/ DamageReportService, ChargeService, CostCatalogService
│   ├── integration/ EvidenceStorage            # UsageEventsPublisher ya existe (plan 1)
│   └── event/ listeners de CheckOutRecorded, DamageReported, LoanDeclaredLost
├── domain/
│   ├── model/ DamageReport, Charge, ChargeAudit, CostCatalog
│   ├── repository/ (un contrato por modelo)
│   └── event/ DamageReported
└── infrastructure/
    ├── persistence/jpa/ entity, repository, adapter, mapper
    ├── integration/module1/kafka/ ResourceStateConsumer
    └── storage/ implementación de EvidenceStorage

frontend/src/pages/ DamageReportPage.jsx, ChargesPage.jsx
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
- Fases 1 y 2 completas: seguridad con la cadena sin JWT (T012), bus de eventos (T016), outbox (T017), inbox (T018), Kafka (T019), `UsageEventsPublisher` (T020), `TrustScoreService` (T022).
- Phase 3: `CheckOut` (T029), evento `CheckOutRecorded` (T033) y consumidor de `LoanDeclaredLost` (T039). Sin ellos el reporte independiente y el cobro por pérdida se pueden probar con datos sembrados, pero no el flujo completo.

Este plan no agrega tareas fundacionales propias.

**Checkpoint**: prerrequisitos disponibles — se puede implementar la user story

---

## Phase 3: User Story — Reportar novedad técnica y gestionar cobros (Priority: P1)

**Goal**: registrar un daño (en el check-out o de forma independiente), avisar a M1, generar y gestionar los cobros (por daño y por préstamo perdido) y responder a M1 si un recurso tiene novedades pendientes. Spec: `spec-reportar-novedad-tecnica.md`.

**Independent Test**: reportar un daño y verificar el reporte "Pendiente", el evento `reporte.danos` en el outbox y el cobro generado; consultar el endpoint de novedades pendientes.

**Capas afectadas**: presentation (`DamageReportController`, `ChargeController`, `PendingIssuesController`), business (`DamageReportService`, `ChargeService`, `CostCatalogService`), domain (`DamageReport`, `Charge`, `ChargeAudit`, `CostCatalog`), infrastructure (publicador hacia M1, consumidor de `estado.actualizado`, almacenamiento de evidencia).

### Tests
- [ ] T001 [P] [NOVEDAD] Test de contrato para `POST /api/v1/damage-reports`, `GET /api/v1/charges`, `POST /api/v1/charges/{id}/pay`, `POST /api/v1/charges/{id}/annul` y `GET /api/v1/novedades/pendientes`
- [ ] T002 [P] [NOVEDAD] Test de integración: daño durante el check-out, daño independiente, sin utilización identificable, duplicado, sin descripción, evidencia inválida; resultado del cobro (Generado, Sin costo, Sin responsable, Error); cobro por préstamo perdido; pago, anulación sin motivo y cambio sobre un cobro ya resuelto
- [ ] T003 [P] [NOVEDAD] Test de contrato Kafka: `reporte.danos` (un solo evento por check-out con daño) y `estado.actualizado` que resuelve la novedad

### Implementation
- [ ] T004 [P] [NOVEDAD] Modelar y migrar `DamageReport` (estado Pendiente/Resuelta, resultado del cobro, instantánea del recurso, evidencia), `Charge` (Pendiente, Pagado, Anulado), `ChargeAudit` y `CostCatalog`
- [ ] T005 [NOVEDAD] `DamageReportService`: registrar el reporte vinculado a la utilización (listener de `CheckOutRecorded` con daño, o endpoint independiente), impedir un segundo reporte por utilización y aceptar recurso sin utilización identificable (depende de T004)
- [ ] T006 [P] [NOVEDAD] `EvidenceStorage` (interfaz en `business/integration`) y su implementación en `infrastructure/storage`, con validación de tamaño y tipo de archivo sin perder el resto del reporte
- [ ] T007 [NOVEDAD] Endpoint `POST /api/v1/damage-reports` para el personal administrativo
- [ ] T008 [NOVEDAD] Listener de `DamageReported`: escribir `reporte.danos` en el outbox hacia M1, sin esperar su confirmación, y marcar la infracción del estudiante responsable (RN-SCO-05)
- [ ] T009 [NOVEDAD] `ChargeService`: generar el cobro por daño (monto del catálogo, responsable de la utilización, monto congelado, resultado guardado en el reporte) y el cobro por préstamo perdido al recibir `LoanDeclaredLost` (50% del valor del catálogo, uno por préstamo)
- [ ] T010 [NOVEDAD] Endpoints de cobros para el personal administrativo: listar, registrar pago y anular con motivo obligatorio, con auditoría y rechazo de cambios sobre cobros ya Pagado o Anulado
- [ ] T011 [P] [NOVEDAD] `CostCatalogService` y `AdminConfigController`: mantenimiento mínimo del catálogo de costos y del porcentaje de pérdida
- [ ] T012 [NOVEDAD] Consumidor de `estado.actualizado` (M1): marcar como Resueltas las novedades del recurso que salió de mantenimiento
- [ ] T013 [NOVEDAD] `PendingIssuesController`: `GET /api/v1/novedades/pendientes?recursoId=` sin JWT y sin depender de Kafka; recurso desconocido responde `false`
- [ ] T014 [NOVEDAD] Frontend: `DamageReportPage.jsx` y `ChargesPage.jsx` (listado, pago, anulación)

**Checkpoint**: Reportar novedad técnica y la gestión de cobros funcionan con el check-out real, y M1 puede consultar las novedades pendientes

---

## Phase N: Polish & Cross-Cutting Concerns

- [ ] T015 Documentación OpenAPI de los endpoints de reporte, cobros y de la ruta entrante de M1
- [ ] T016 Prueba de punta a punta con Kafka: check-out con daño → `reporte.danos` → cobro generado → `estado.actualizado` → novedad "Resuelta"; préstamo perdido → cobro del 50%
- [ ] T017 Ajuste de performance contra SC-005 y SC-006 (<2 s)
- [ ] T018 Hardening: revisar con M1 la cadena sin JWT de `/api/v1/novedades/**` y validación de archivos de evidencia
- [ ] T019 Tests unitarios: monto congelado del cobro, 50% del valor por pérdida, transiciones de estado del cobro

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: cubierta por `plan-registrar-checkout`
- **Foundational (Phase 2)**: prerrequisitos de `plan-registrar-checkout` (fases 1 y 2 y las tareas de check-out indicadas)
- **Reportar novedad técnica (Phase 3)**: depende de la Phase 2
- **Polish**: depende de la Phase 3

### Within Each User Story

- Modelos y contratos de repositorio antes que los adaptadores; adaptadores antes que servicios; servicios antes que controladores o listeners
- `DamageReported` se publica al final de la transacción del reporte; `reporte.danos` sale por el outbox
- Tests antes de la implementación

### Relación con otros planes

- `plan-consultar-cumplimiento` muestra los cobros de este plan; `plan-consultar-analitica-cumplimiento` usa reportes y cobros para indicadores y rankings.

## Notes

- Los cobros se gestionan dentro de este caso de uso para mantener los seis casos de uso del diagrama. Si prefieren un caso de uso aparte ("Gestionar cobros"), se separan las tareas de cobros (T009 a T011 de este plan) sin tocar el resto.
- **Pendientes de contrato con M1**: payload de `reporte.danos`; que `estado.actualizado` se emita también al salir de mantenimiento; si el contrato exige autenticación en `GET /api/v1/novedades/pendientes`.
- Riesgo abierto: el mantenimiento del catálogo de costos no tiene caso de uso propio; se cubre con endpoints de administración mínimos.
- Verificar tests en verde y hacer commit después de cada tarea o grupo lógico; detenerse en cada Checkpoint.
