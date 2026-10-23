# Implementation Plan: Consultar analítica de cumplimiento (Módulo 3)

**Date**: 2026-10-23
**Spec**: `spec-consultar-analitica-cumplimiento.md` (reglas aplicables en `reglas-negocio-m3.md`: RN-COB-04 a RN-COB-08 (estados de cobro); sin reglas propias de analítica)
**Plan**: 6 de 6 del Módulo 3 — Control de Uso, Sanciones y Analítica (orden de implementación y fundamentos compartidos en `plan-indice.md`)

## Summary

Este plan implementa el panel de analítica del personal administrativo: indicadores generales del período (check-outs a tiempo y con retraso, descuentos por retraso, préstamos perdidos, sanciones por inasistencia, reportes de daño y cobros por estado), rankings de recursos con más daños y de espacios con más inasistencias, exportación a CSV o PDF, y el detalle nominal de un estudiante buscado por código institucional. Los indicadores generales no identifican estudiantes. Todo se construye solo con datos de M3.

## Technical Context

**Language/Version**: Java 21 (LTS), Spring Boot (versión con soporte vigente al iniciar el desarrollo)
**Primary Dependencies**: Spring Web, Spring Data JPA, Spring Validation, Spring Security (JWT), Spring for Apache Kafka, Flyway (migraciones)
**Storage**: MySQL 8
**Testing**: JUnit 5 + Mockito + Spring Boot Test; Testcontainers (MySQL y Kafka)
**Target Platform**: Linux server, contenedores Docker (backend, frontend, MySQL, Kafka)
**Project Type**: Web application (backend + frontend separados)
**Frontend Framework**: React + Vite (confirmado)
**Message Broker**: Apache Kafka (confirmado)
**Performance Goals**: panel de un semestre en menos de 5 s (SC-001); exportación en menos de 10 s (SC-005); detalle de un estudiante en menos de 3 s (SC-006)
**Constraints**:
- Los indicadores deben coincidir al 100% con los datos de origen para el mismo período y filtros (SC-002).
- Indicadores generales y rankings no identifican estudiantes (SC-003); el detalle nominal es solo para el personal administrativo y no registra quién consulta.
- Solo datos de M3: sin llamadas a M1 ni a M2; tipo y ubicación del recurso salen de las instantáneas guardadas al registrar cada evento.
- Los registros sin facultad o ubicación se agrupan en "Sin clasificar" y no se excluyen de los totales.
- Mostrar la fecha y hora de la última actualización de los datos; agregar y paginar para grandes volúmenes.

## Module Communication

Este caso de uso **no usa Kafka ni expone endpoints a otros módulos**. No hay integración directa con M1 ni M2: la analítica solo lee datos de M3.

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan-consultar-analitica-cumplimiento.md                 # Este archivo
├── spec-consultar-analitica-cumplimiento.md
└── reglas-negocio-m3.md
```

### Source Code (archivos de este caso de uso)

```text
backend/src/main/java/com/university/sanctions/
├── presentation/controller/AnalyticsController.java
├── business/service/ AnalyticsService
├── domain/ (solo lectura sobre CheckOut, DamageReport, Sanction, AbsenceConfirmation, Charge, ScoreHistory)
└── infrastructure/persistence/jpa/ consultas agregadas e índices

frontend/src/pages/ AnalyticsPage.jsx, StudentDetailPage.jsx
```

**Layered Architecture:** `presentation` recibe solicitudes; `business` coordina el caso de uso; `domain` contiene reglas, modelos y contratos de repositorio; `infrastructure` los implementa con MySQL/JPA y Kafka. Flujo: `presentation → business → domain`, e `infrastructure` implementa los contratos de `domain` y `business`. Los casos de uso se encadenan por eventos de dominio internos, nunca llamando al servicio de otro caso de uso. El detalle completo de capas y carpetas está en `plan-indice.md`.

## Phase 1: Setup

**Purpose**: Inicialización del proyecto

Cubierta por `plan-registrar-checkout` (T001–T005). No hay tareas nuevas en este plan.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: lo que este caso de uso necesita y que construyen otros planes

**⚠️ CRITICAL**: no empezar la Phase 3 hasta tener estos prerrequisitos

De los demás planes (lee sus datos):
- `plan-registrar-checkout`: `CheckOut`, `OverdueDeduction`, `LostLoan`, `TrustScore`, `ScoreHistory`.
- `plan-reportar-novedad-tecnica`: `DamageReport`, `Charge`.
- `plan-confirmar-no-asistencia`: `AbsenceConfirmation`, `Sanction`.
- `plan-desbloquear-usuario`: `UnblockAudit` (para el detalle por estudiante).

Puede empezarse antes con los modelos ya existentes y completar cada indicador cuando exista su fuente.

Este plan no agrega tareas fundacionales propias.

**Checkpoint**: prerrequisitos disponibles — se puede implementar la user story

---

## Phase 3: User Story — Consultar analítica de cumplimiento (Priority: P1)

**Goal**: el personal administrativo ve indicadores generales, rankings y el detalle de cada estudiante, y exporta el reporte. Spec: `spec-consultar-analitica-cumplimiento.md`.

**Independent Test**: con datos de un semestre, abrir el panel y verificar que los indicadores coinciden con los datos de origen; buscar un estudiante por código y verificar su detalle.

**Capas afectadas**: presentation (`AnalyticsController`), business (`AnalyticsService`), domain (solo lectura), infrastructure (consultas agregadas e índices).

### Tests
- [ ] T001 [P] [ANALITICA] Test de contrato para `GET /api/v1/analytics/overview`, `/rankings`, `/export` y `/students/{code}`
- [ ] T002 [P] [ANALITICA] Test de integración: indicadores del período por defecto (semestre vigente), filtros, período sin datos, rango inválido, rankings, "Sin clasificar", sin permisos, detalle de estudiante con y sin historial, exportación con los mismos indicadores

### Implementation
- [ ] T003 [ANALITICA] `AnalyticsService`: porcentaje de check-outs a tiempo y con retraso, préstamos con descuento por retraso y perdidos, sanciones por inasistencia, reportes de daño, cobros por estado (cantidad y monto), todo solo con datos de M3, con fecha de la última actualización
- [ ] T004 [ANALITICA] Filtros por rango de fechas, tipo de recurso o espacio y facultad (hasta que se defina el origen de la facultad, esos registros quedan en "Sin clasificar")
- [ ] T005 [ANALITICA] Rankings de recursos con más reportes de daño y de espacios con más inasistencias confirmadas, sin identificar estudiantes
- [ ] T006 [ANALITICA] Detalle por estudiante `GET /api/v1/analytics/students/{code}`: score e historial, bloqueo, sanciones, check-outs, préstamos vencidos, inasistencias, daños y cobros; sin registrar quién consulta
- [ ] T007 [ANALITICA] Exportación a CSV y PDF con los indicadores, el período y los filtros usados; una falla no afecta la consulta en pantalla
- [ ] T008 [ANALITICA] Restringir todo al personal administrativo e índices para cumplir <5 s en un semestre
- [ ] T009 [ANALITICA] Frontend: `AnalyticsPage.jsx` y `StudentDetailPage.jsx`

**Checkpoint**: los seis casos de uso están conectados de punta a punta

---

## Phase N: Polish & Cross-Cutting Concerns

- [ ] T010 Documentación OpenAPI de los endpoints de analítica
- [ ] T011 Ajuste de performance contra SC-001, SC-005 y SC-006 con un semestre de datos de prueba
- [ ] T012 Tests unitarios: cada indicador contra los datos de origen, período sin datos, "Sin clasificar" y rango de fechas inválido
- [ ] T013 Revisión de privacidad: confirmar que indicadores generales y rankings no exponen datos individuales

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: cubierta por `plan-registrar-checkout`
- **Foundational (Phase 2)**: modelos de los otros cinco planes, según el indicador
- **Analítica (Phase 3)**: es el último plan en implementarse completo
- **Polish**: depende de la Phase 3

### Within Each User Story

- Indicadores generales antes que filtros; filtros antes que rankings, detalle y exportación
- Índices y permisos antes de medir performance
- Tests antes de la implementación

## Notes

- **Pendiente con M1 y M2**: de dónde sale la facultad para el filtro; hoy ningún módulo la entrega a M3 y esos registros quedan en "Sin clasificar".
- No se registra quién consulta el detalle de cada estudiante (decisión del equipo).
- Verificar tests en verde y hacer commit después de cada tarea o grupo lógico; detenerse en cada Checkpoint.
