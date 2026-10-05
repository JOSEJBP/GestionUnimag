# Implementation Plan: Bloquear usuario (UC3)

**Date**: 2026-10-05
**Spec**: [bloquear-usuario.md](../Especs/bloquear-usuario.md)
**Plan general**: [plan.md](../Especs/plan.md)
**Módulo**: Módulo 3 — Sanciones y Cumplimiento
**Planes previos**: [plan-uc1-actualizar-score.md](./plan-uc1-actualizar-score.md) — UC4 lo dispara cuando el score llega a 0 o sube de 0
**Planes relacionados**: [plan-uc-reportar-novedad-tecnica.md](./plan-uc-reportar-novedad-tecnica.md) — UC8 lo dispara cuando hay daño grave; [plan-uc2-calcular-penalizacion.md](./plan-uc2-calcular-penalizacion.md) — UC3 lo dispara indirectamente vía UC4

## Summary

UC5 es el **guardián del acceso** del módulo. Mantiene el estado de cumplimiento de cada estudiante (`Activo`, `Bloqueado por score`, `Bloqueado por daño`, `Bloqueado por ambos`) y decide si puede o no reservar. Sin él, un estudiante con score 0 seguiría reservando sin consecuencias.

Es un caso de uso **puramente interno**: no produce eventos Kafka, no expone endpoints nuevos a M2, no habla con M1. Lo que hace es **alimentar el endpoint que M2 ya consume** (`GET /api/v1/compliance/people/{code}`) para que M2 vea el estado actualizado sin cambiar nada de su lado.

**Enfoque técnico:**

1. UC5 mantiene el estado del estudiante en `user_compliance_profile`. Distingue **dos tipos de bloqueo independientes**: por score (cuando el score llega a 0) y por daño (cuando hay un daño grave sin resolver). Los dos pueden coexistir (FR-005).
2. Recibe disparos de **UC4** (`BlockUserPort.block` cuando el score llega a 0, `BlockUserPort.unblock` cuando sube de 0) y de **UC8** (`BlockForDamagePort.block` cuando hay daño grave).
3. Permite al **administrador** desbloquear manualmente a un estudiante (FR-004), registrando la intervención en el historial.
4. Cada bloqueo y desbloqueo se registra en `user_block_history` con fecha, motivo y usuario responsable (si es manual).
5. **Alimenta el endpoint `GET /api/v1/compliance/people/{code}`** (que ya existe en UC1 del M3) con el estado de bloqueo. Añade campos opcionales (`isBlocked`, `blockReason`) sin romper los que M2 ya conoce.
6. **No produce eventos Kafka.** M2 no consume eventos de bloqueo; consulta por REST cuando lo necesita.
7. **No expone endpoints nuevos a M2.** El endpoint que M2 consume ya existe.
8. El UC tiene **4 user stories**: bloqueo automático por score (US1), desbloqueo por recuperación o manual (US2), banner en el perfil (US3), y acceso al reglamento (US4).

## Technical Context

**Language/Version**: Java 21 (backend); JavaScript con React + Vite (frontend)
**Primary Dependencies**: Spring Boot 4.x (Web, Data JPA, Validation, Security OAuth2 Resource Server), Flyway, springdoc-openapi. **Ninguna nueva** respecto a `plan.md`. **No usa Spring for Apache Kafka**: UC5 no publica ni consume Kafka.
**Storage**: MySQL 8. UC5 **escribe** `user_compliance_profile` y `user_block_history`. Lee `trust_score` (para saber si el score es 0) y `penalty_history` (para el motivo del bloqueo).
**Testing**: JUnit 5 + Mockito (dominio y casos de uso), `@WebMvcTest` (controladores de consulta y desbloqueo manual), Testcontainers MySQL (persistencia, coexistencia de bloqueos y transacciones), ArchUnit (regla de dependencias entre capas).
**Target Platform**: Servidor Linux con JVM 21; navegador web para el frontend.
**Project Type**: Web: backend y frontend separados (React + Vite).
**Performance Goals**: El bloqueo se aplica en **menos de 1 segundo** desde que UC4 lo dispara (SC-004). Es una transacción local: un `UPDATE` y un `INSERT`.
**Constraints**:
- Dos bloqueos coexisten de forma independiente: por score y por daño (FR-005).
- Un bloqueo duplicado no genera un registro inconsistente (FR-006).
- El desbloqueo por score solo ocurre si no hay bloqueo por daño vigente (SC-003).
- La zona horaria es `America/Bogota`.
**Scale/Scope**: Mismo orden que M2 (miles de estudiantes). Una pantalla de perfil con el banner de bloqueo; una pantalla de administración para desbloqueo manual.

## Integración con otros módulos

### Lo que M3 consume de M1 y M2 (adoptado tal cual)

**UC5 no consume eventos de M1 ni de M2.** Sus disparadores son internos del M3:
- UC4 (`BlockUserPort.block` / `unblock`) por score.
- UC8 (`BlockForDamagePort.block`) por daño.

### Lo que M3 produce para M1 y M2 (creado por M3)

**UC5 no produce eventos Kafka.** M2 no consume eventos de bloqueo. M2 consulta el estado por REST cuando lo necesita (al reservar, al consultar).

### Endpoints que M3 expone para M2

**UC5 no expone endpoints nuevos a M2.** Lo que hace es **alimentar el endpoint que M2 ya consume**:

| Endpoint | Método | Quién lo consume | Qué cambia |
|---|---|---|---|
| `GET /api/v1/compliance/people/{code}` | GET | M2 | UC5 añade campos opcionales (`isBlocked`, `blockReason`) sin romper los que M2 ya conoce |

**El endpoint ya existe** (definido en UC1 del M3). UC5 solo enriquece su respuesta.

### Endpoints que M3 expone para su propio frontend

| Endpoint | Método | Para qué | Rol |
|---|---|---|---|
| `GET /api/v1/compliance/me` | GET | El estudiante consulta su estado de cumplimiento | `ESTUDIANTE` o `MONITOR` |
| `GET /api/v1/admin/compliance/{code}` | GET | El administrador consulta el estado de cualquier estudiante | `ADMIN` o `DIRECCION_PROGRAMA` |
| `POST /api/v1/admin/compliance/{code}/unblock` | POST | El administrador desbloquea manualmente a un estudiante | `ADMIN` |

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan.md                                    # Plan general del M3
├── bloquear-usuario.md                         # Spec de este plan
├── actualizar-score-confianza.md               # «extend» que lo dispara
├── reportar-novedad-tecnica.md                 # Disparador por daño
└── ...                                         # resto de specs
```

### Source Code (repository root)

Solo los archivos que crea o toca este plan. La organización completa está en `plan.md`.

```text
backend/
├── src/main/java/com/university/sanctions/
│   ├── presentation/
│   │   ├── controller/
│   │   │   ├── ComplianceController.java           # GET /api/v1/compliance/me (estudiante)
│   │   │   ├── AdminComplianceController.java      # GET y POST para admin
│   │   │   └── ComplianceQueryController.java      # GET /api/v1/compliance/people/{code} (para M2)
│   │   └── dto/
│   │       ├── ComplianceStatusResponse.java       # respuesta al estudiante/admin
│   │       ├── ComplianceQueryResponse.java        # respuesta a M2 (con campos opcionales)
│   │       └── ManualUnblockRequest.java
│   │
│   ├── business/
│   │   ├── service/
│   │   │   └── ComplianceService.java              # implementa BlockUserPort y BlockForDamagePort
│   │   └── event/
│   │       └── UserBlockStatusChangedEvent.java    # evento de dominio interno (opcional)
│   │
│   ├── domain/
│   │   ├── model/
│   │   │   ├── UserComplianceProfile.java          # estado del estudiante
│   │   │   ├── UserBlockHistory.java               # registro de cada bloqueo/desbloqueo
│   │   │   ├── BlockReason.java                    # enum: SCORE_ZERO, DAMAGE, MANUAL
│   │   │   ├── BlockType.java                      # enum: SCORE, DAMAGE
│   │   │   └── ComplianceStatus.java               # enum: ACTIVE, BLOCKED_BY_SCORE, BLOCKED_BY_DAMAGE, BLOCKED_BY_BOTH
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   ├── BlockUserPort.java              # ya definido en UC4
│   │   │   │   ├── BlockForDamagePort.java         # nuevo, para UC8
│   │   │   │   └── ManualUnblockPort.java          # para admin
│   │   │   └── out/
│   │   │       ├── UserComplianceProfileRepository.java
│   │   │       ├── UserBlockHistoryRepository.java
│   │   │       └── TrustScoreQueryPort.java        # para leer el score actual
│   │   └── error/
│   │       ├── UserComplianceProfileNotFound.java
│   │       ├── AlreadyBlockedException.java
│   │       └── NotBlockedException.java
│   │
│   └── infrastructure/
│       ├── persistence/
│       │   └── jpa/
│       │       ├── entity/
│       │       │   ├── UserComplianceProfileEntity.java
│       │       │   └── UserBlockHistoryEntity.java
│       │       ├── repository/
│       │       │   ├── JpaUserComplianceProfileRepository.java
│       │       │   └── JpaUserBlockHistoryRepository.java
│       │       ├── adapter/
│       │       │   ├── UserComplianceProfileRepositoryAdapter.java
│       │       │   └── UserBlockHistoryRepositoryAdapter.java
│       │       └── mapper/
│       │           ├── UserComplianceProfileMapper.java
│       │           └── UserBlockHistoryMapper.java
│       └── config/
│           ├── SecurityConfig.java
│           └── UseCasesConfig.java
│
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       ├── V9__user_compliance_profile.sql
│       └── V10__user_block_history.sql
│
└── src/test/java/com/university/sanctions/
    ├── contract/
    │   ├── ComplianceControllerTest.java
    │   ├── AdminComplianceControllerTest.java
    │   └── ComplianceQueryControllerTest.java      # el endpoint que consume M2
    ├── integration/
    │   ├── ComplianceServiceIT.java                # persistencia + coexistencia de bloqueos
    │   └── ManualUnblockIT.java                    # desbloqueo manual
    └── unit/
        ├── UserComplianceProfileTest.java
        └── ComplianceServiceTest.java

frontend/
└── src/
    ├── pages/
    │   ├── ProfilePage.jsx                          # perfil del estudiante con banner
    │   └── AdminCompliancePage.jsx                  # administración
    ├── components/
    │   ├── BlockBanner.jsx                          # banner rojo cuando está bloqueado
    │   ├── ActiveBlockBadge.jsx                     # badge con motivo y variación
    │   ├── ComplianceStatusCard.jsx                 # tarjeta con estado
    │   └── LoanRegulationLink.jsx                   # enlace al reglamento
    └── services/
        └── complianceApi.js                         # cliente HTTP
```

Structure Decision: se respeta la estructura de capas de plan.md (presentation / business / domain / infrastructure). El paquete raíz es com.university.sanctions. La lógica del bloqueo vive en business/service/ComplianceService.java (orquesta) y el modelo en domain/model/. Los puertos de entrada (BlockUserPort ya definido en UC4, BlockForDamagePort nuevo, ManualUnblockPort nuevo) viven en domain/port/in/. La persistencia vive en infrastructure. El frontend se organiza por páginas y componentes reutilizables.

Decisiones de diseño de este caso de uso
Dónde queda cada FR.

FR	Qué pide	Dónde se implementa
FR-001	Bloquear automáticamente cuando el score llegue a 0	ComplianceService.block(...) — lo llama UC4
FR-002	Rechazar cualquier intento de préstamo o check-out de un bloqueado	El endpoint GET /api/v1/compliance/people/{code} devuelve isBlocked: true; M2 y M3 lo consultan
FR-003	Restaurar el estado Activo cuando el score suba de 0	ComplianceService.unblock(...) — lo llama UC4
FR-004	Permitir al admin desbloquear manualmente	ComplianceService.manualUnblock(...) — endpoint POST /api/v1/admin/compliance/{code}/unblock
FR-005	Mantener el bloqueo por daño independiente del bloqueo por score	UserComplianceProfile tiene dos flags: blockedByScore, blockedByDamage. El estado combinado se calcula
FR-006	Impedir bloqueo duplicado	UserComplianceProfile.isAlreadyBlockedBy(type) → no-op con log
FR-007	Banner rojo cuando el score sea 0 o haya sanción que bloquee	BlockBanner.jsx
FR-008	Banner muestra estado, score sobre 50 y motivo	BlockBanner.jsx consume ComplianceStatusResponse
FR-009	Enlace "Ver reglamento de préstamos"	LoanRegulationLink.jsx
FR-010	Mostrar estado habilitado cuando el score > 0 y no hay bloqueos	ComplianceStatusCard.jsx
Decisiones de diseño justificadas.

1. UC5 no produce eventos Kafka. M2 no consume eventos de bloqueo. M2 consulta el estado por REST (GET /api/v1/compliance/people/{code}). Publicar eventos que nadie consume es ruido. UC5 solo actualiza su tabla y alimenta el endpoint existente.

2. UC5 no expone endpoints nuevos a M2. El endpoint que M2 consume ya existe (definido en UC1 del M3). UC5 solo enriquece la respuesta con campos opcionales (isBlocked, blockReason) sin romper los que M2 ya conoce. M2 sigue funcionando igual.

3. Dos tipos de bloqueo independientes. UserComplianceProfile tiene dos flags: blockedByScore y blockedByDamage. El estado combinado (ComplianceStatus) se calcula:

Ambos falsos → ACTIVE

Solo score → BLOCKED_BY_SCORE

Solo daño → BLOCKED_BY_DAMAGE

Ambos → BLOCKED_BY_BOTH

Esto cumple FR-005.

4. El desbloqueo por score no quita el bloqueo por daño. Si UC4 llama a unblock (score subió de 0) pero blockedByDamage = true, el estudiante sigue bloqueado. El estado pasa de BLOCKED_BY_BOTH a BLOCKED_BY_DAMAGE. Esto cumple SC-003.

5. Bloqueo duplicado es no-op. Si UC4 llama a block dos veces (porque el score era 0 y ya estaba bloqueado), la segunda llamada no genera un nuevo registro. ComplianceService.block(...) verifica isAlreadyBlockedBy(SCORE) antes de actuar. Esto cumple FR-006.

6. El historial registra cada cambio de estado, no cada llamada. user_block_history guarda una fila solo cuando el estado cambia (de ACTIVE a BLOCKED_BY_SCORE, de BLOCKED_BY_BOTH a BLOCKED_BY_DAMAGE, etc.). Un bloqueo duplicado no genera fila. Esto hace que el historial sea limpio y auditable.

7. UC5 alimenta el endpoint de M2 sin cambiar su forma. El endpoint GET /api/v1/compliance/people/{code} (definido en UC1 del M3) devuelve los campos que M2 ya conoce (isSanctioned, status, sanctions, noticeMessage). UC5 añade dos campos opcionales:

isBlocked: boolean

blockReason: string (solo si isBlocked)

M2 ignora estos campos si no los necesita. Si M2 los quiere usar en el futuro, los tiene disponibles.

8. El desbloqueo manual registra quién lo hizo. ManualUnblockPort.unblock(studentCode, adminCode, reason) registra en user_block_history el adminCode como responsable. Esto cumple FR-004.

9. La tabla user_compliance_profile se crea bajo demanda. Si un estudiante no tiene perfil y UC4 llama a block, se crea con ACTIVE y luego se bloquea. Esto evita un evento de creación de estudiante que no existe en M3.

10. El banner muestra el score actual y el motivo más reciente. BlockBanner.jsx consume GET /api/v1/compliance/me, que devuelve:

status: ACTIVE, BLOCKED_BY_SCORE, etc.

currentScore: 0-50 (de trust_score)

mostRecentBlockReason: el motivo del último bloqueo (SCORE_ZERO, DAMAGE, MANUAL)

## Contratos

Se aplican las convenciones comunes de M2 (camelCase, MAYUSCULA_CON_GUION_BAJO, nunca null, ISO-8601 con -05:00, RFC 9457). M3 las adopta para sus propios contratos.

### 1. Puerto de entrada — BlockUserPort (ya definido en UC4)

Lo llama Actualizar score (UC4) cuando el score llega a 0 o sube de 0.

```java
public interface BlockUserPort {
    void block(String studentCode, String reason, String sourceEventId);
    void unblock(String studentCode, String reason, String sourceEventId);
}
```
UC5 lo implementa en `ComplianceService`.

### 2. Puerto de entrada — BlockForDamagePort (nuevo)

Lo llama Reportar novedad técnica (UC8) cuando hay daño grave.

```java
public interface BlockForDamagePort {
    void block(String studentCode, String reason, String sourceEventId);
    void unblock(String studentCode, String reason, String sourceEventId);
}
```

Misma firma que `BlockUserPort`, pero con `BlockType.DAMAGE`. UC5 lo implementa en `ComplianceService`.

### 3. Puerto de entrada — ManualUnblockPort (nuevo)

Lo llama el administrador desde el endpoint `POST /api/v1/admin/compliance/{code}/unblock`.

```java
public interface ManualUnblockPort {
    void manualUnblock(String studentCode, String adminCode, String reason);
}
```

UC5 lo implementa en `ComplianceService`.

### 4. Endpoint para M2 — GET /api/v1/compliance/people/{code}

Ya existe (definido en UC1 del M3). UC5 enriquece la respuesta con dos campos opcionales:

```json
{
  "userCode": "2023123456",
  "isSanctioned": true,
  "status": "ACTIVE_SANCTION",
  "sanctions": [
    {
      "id": "sanc-9921",
      "reason": "Devolución con retraso de equipo audiovisual",
      "severity": "MEDIUM",
      "startDate": "2026-10-01T08:00:00-05:00",
      "endDate": "2026-10-08T23:59:59-05:00"
    }
  ],
  "noticeMessage": "Cuenta inhabilitada temporalmente para reservas hasta el 08/10/2026.",
  "isBlocked": true,
  "blockReason": "Score de confianza en 0"
}
```

Los campos `isSanctioned`, `status`, `sanctions`, `noticeMessage` NO cambian. M2 sigue funcionando igual. Los campos `isBlocked` y `blockReason` son nuevos y opcionales.

### 5. Endpoints REST — consulta del estudiante

`GET /api/v1/compliance/me`
Devuelve el estado de cumplimiento del estudiante autenticado. Rol `ESTUDIANTE` o `MONITOR`.

```json
{
  "studentCode": "2023123456",
  "status": "BLOCKED_BY_SCORE",
  "currentScore": 0,
  "maxScore": 50,
  "blockedByScore": true,
  "blockedByDamage": false,
  "mostRecentBlockReason": "Score de confianza en 0",
  "mostRecentBlockAt": "2026-10-05T11:15:00-05:00",
  "hasActiveBlocks": true,
  "message": "ESTADO: BLOQUEADO POR SANCIÓN COMPLETA. Su score actual es 0 / 50."
}
```

Cuando está activo:

```json
{
  "studentCode": "2023123456",
  "status": "ACTIVE",
  "currentScore": 45,
  "maxScore": 50,
  "blockedByScore": false,
  "blockedByDamage": false,
  "hasActiveBlocks": false,
  "message": "Tu cuenta está activa. Puedes reservar con normalidad."
}
```

| Campo | Tipo | Nota |
|---|---|---|
| `status` | enum | `ACTIVE`, `BLOCKED_BY_SCORE`, `BLOCKED_BY_DAMAGE`, `BLOCKED_BY_BOTH` |
| `currentScore` | int | `0-50` |
| `blockedByScore` | boolean |  |
| `blockedByDamage` | boolean |  |
| `mostRecentBlockReason` | string | Se omite si no hay bloqueos |
| `mostRecentBlockAt` | instante | Se omite si no hay bloqueos |
| `hasActiveBlocks` | boolean |  |
| `message` | string | Texto legible para el banner |

### 6. Endpoints REST — consulta del administrador

`GET /api/v1/admin/compliance/{code}`
Igual que el del estudiante, pero para cualquier code. Rol `ADMIN` o `DIRECCION_PROGRAMA`.

Incluye `studentName` y `studentEmail`.

### 7. Endpoints REST — desbloqueo manual

`POST /api/v1/admin/compliance/{code}/unblock`
Rol `ADMIN`.

```json
{
  "reason": "Revisión administrativa: el daño fue causado por desgaste normal."
}
```

Respuesta `200 OK`:

```json
{
  "studentCode": "2023123456",
  "status": "ACTIVE",
  "unblockedAt": "2026-10-06T09:00:00-05:00",
  "unblockedBy": "admin@unimagdalena.edu.co",
  "reason": "Revisión administrativa: el daño fue causado por desgaste normal."
}
```

Nota: si el estudiante tiene ambos bloqueos (score y daño), el desbloqueo manual solo quita el bloqueo por daño. El bloqueo por score sigue vigente hasta que el score suba de 0. Esto cumple SC-003.

### 8. Errores

| Código HTTP | `code` | Cuándo |
|---|---|---|
| 400 | `MISSING_REASON` | El desbloqueo manual no tiene reason |
| 404 | `PROFILE_NOT_FOUND` | El estudiante no tiene perfil de cumplimiento |
| 409 | `ALREADY_BLOCKED` | Se intenta bloquear a alguien ya bloqueado por el mismo tipo |
| 409 | `NOT_BLOCKED` | Se intenta desbloquear a alguien que no está bloqueado |
| 500 | `PERSISTENCE_ERROR` | Fallo al persistir |

`ProblemDetail` de ejemplo:

```json
{
  "type": "https://sanctions.unimagdalena.edu.co/errors/already-blocked",
  "title": "El usuario ya está bloqueado",
  "status": 409,
  "detail": "El estudiante 2023123456 ya tiene un bloqueo por score vigente.",
  "instance": "/api/v1/admin/compliance/2023123456/unblock",
  "code": "ALREADY_BLOCKED",
  "blockType": "SCORE"
}
```

### 9. Tablas

#### `user_compliance_profile`

| Columna | Tipo | Nota |
|---|---|---|
| `student_code` | `varchar(20) PK` | Código institucional |
| `blocked_by_score` | `boolean NOT NULL DEFAULT false` |  |
| `blocked_by_damage` | `boolean NOT NULL DEFAULT false` |  |
| `most_recent_block_reason` | `varchar(255), nulo` |  |
| `most_recent_block_at` | `timestamp, nulo` |  |
| `created_at` | `timestamp NOT NULL` |  |
| `updated_at` | `timestamp NOT NULL` |  |

El estado combinado (`ACTIVE`, `BLOCKED_BY_SCORE`, etc.) se calcula a partir de los dos flags. No se guarda como columna para evitar inconsistencias.

#### `user_block_history`

| Columna | Tipo | Nota |
|---|---|---|
| `id` | `varchar(36) PK` |  |
| `student_code` | `varchar(20) FK` |  |
| `block_type` | `varchar(20) NOT NULL` | `SCORE`, `DAMAGE` |
| `action` | `varchar(20) NOT NULL` | `BLOCK`, `UNBLOCK` |
| `reason` | `varchar(255) NOT NULL` |  |
| `source_event_id` | `varchar(36), nulo` | El evento que originó el cambio |
| `performed_by` | `varchar(50), nulo` | Nulo si es automático, el `adminCode` si es manual |
| `occurred_at` | `timestamp NOT NULL` |  |

```sql
CREATE INDEX user_block_history_student ON user_block_history (student_code, occurred_at DESC);
```

La tabla guarda solo cambios de estado reales. Un bloqueo duplicado no genera fila.

### 10. Tipos del frontend

```js
export const ComplianceStatus = {
  ACTIVE: "ACTIVE",
  BLOCKED_BY_SCORE: "BLOCKED_BY_SCORE",
  BLOCKED_BY_DAMAGE: "BLOCKED_BY_DAMAGE",
  BLOCKED_BY_BOTH: "BLOCKED_BY_BOTH",
};

export const getMyCompliance = async () => {
  const response = await fetch("/api/v1/compliance/me");
  if (!response.ok) throw await response.json();
  return response.json();
};

export const getStudentCompliance = async (studentCode) => {
  const response = await fetch(`/api/v1/admin/compliance/${studentCode}`);
  if (!response.ok) throw await response.json();
  return response.json();
};

export const manualUnblock = async (studentCode, reason) => {
  const response = await fetch(`/api/v1/admin/compliance/${studentCode}/unblock`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ reason }),
  });
  if (!response.ok) throw await response.json();
  return response.json();
};
```

### 11. Fixtures compartidos

```text
backend/src/test/resources/contracts/
├── api-compliance-me-active-200.json
├── api-compliance-me-blocked-score-200.json
├── api-compliance-me-blocked-both-200.json
├── api-compliance-admin-200.json
├── api-compliance-unblock-200.json
├── api-compliance-people-for-m2-200.json       # el endpoint que M2 consume
└── api-compliance-409-already-blocked.json
```
Phase 1: Setup
□ T001 Añadir a application.yml los parámetros: sanctions.compliance.regulation-url=https://... (para el enlace al reglamento)
□ T002 [P] Extender SanctionsProperties con ese parámetro
Phase 2: Foundational (Blocking Prerequisites)
□ T003 Escribir V9__user_compliance_profile.sql con la tabla user_compliance_profile de Contratos §9
□ T004 Escribir V10__user_block_history.sql con la tabla user_block_history y su índice
□ T005 [P] Crear el modelo de dominio: UserComplianceProfile, UserBlockHistory, BlockReason, BlockType, ComplianceStatus en domain/model/
□ T006 [P] Definir los repositorios UserComplianceProfileRepository y UserBlockHistoryRepository en domain/port/out/
□ T007 [P] Definir BlockForDamagePort y ManualUnblockPort en domain/port/in/ (verificar que BlockUserPort ya existe de UC4)
□ T008 [P] Definir TrustScoreQueryPort en domain/port/out/ (para leer el score actual)
□ T009 [P] Crear UserComplianceProfileNotFound, AlreadyBlockedException, NotBlockedException en domain/error/
□ T010 [P] Implementar las entidades JPA UserComplianceProfileEntity y UserBlockHistoryEntity y sus repositorios Spring Data
□ T011 [P] Implementar los adaptadores UserComplianceProfileRepositoryAdapter y UserBlockHistoryRepositoryAdapter con sus mappers
□ T012 Enriquecer el ComplianceQueryController (de UC1) para que devuelva los campos isBlocked y blockReason
Checkpoint: Existe dónde guardar el estado del usuario y el historial de bloqueos

Phase 3: User Story 1 — Bloqueo automático por score de confianza en 0 (Priority: P1)
Goal: Cuando UC4 detecte que el score llegó a 0, UC5 bloquea al estudiante, registra el bloqueo, y M2 lo ve por REST.

Tests for User Story 1
□ T013 [P] [US1] Pruebas en ComplianceServiceTest.java (unit):
block(studentCode, reason, sourceEventId) marca blockedByScore = true y registra en el historial

Un bloqueo sobre alguien ya bloqueado por score es no-op (FR-006)

Un bloqueo sobre alguien con blockedByDamage = true pasa a BLOCKED_BY_BOTH

El estado calculado es correcto (ACTIVE, BLOCKED_BY_SCORE, BLOCKED_BY_DAMAGE, BLOCKED_BY_BOTH)

□ T014 [P] [US1] Prueba ComplianceServiceIT.java con Testcontainers:
El UPDATE de user_compliance_profile y el INSERT de user_block_history ocurren en la misma transacción

Un fallo a mitad → rollback completo

Un bloqueo duplicado no genera dos filas en el historial

□ T015 [P] [US1] Prueba ComplianceQueryControllerTest.java con @WebMvcTest:
GET /api/v1/compliance/people/{code} con blockedByScore = true devuelve isBlocked: true y blockReason

GET /api/v1/compliance/people/{code} sin bloqueo devuelve isBlocked: false y omite blockReason

Los campos que M2 ya conoce (isSanctioned, status, sanctions, noticeMessage) no cambian

Implementation for User Story 1
□ T016 [US1] Implementar ComplianceService.block(...):
Buscar o crear UserComplianceProfile
Si blockedByScore = true, no-op con log (FR-006)
Marcar blockedByScore = true
Actualizar most_recent_block_reason y most_recent_block_at
Persistir UserComplianceProfile
Insertar UserBlockHistory con action = BLOCK, blockType = SCORE
(depende de T005 a T011)
□ T017 [US1] Implementar el @Transactional en block(...)
□ T018 [US1] Enriquecer ComplianceQueryController (de UC1) para consultar user_compliance_profile y añadir isBlocked y blockReason (depende de T012)
Checkpoint: El bloqueo por score funciona y M2 lo ve sin cambiar nada

Phase 4: User Story 2 — Desbloqueo por recuperación o intervención administrativa (Priority: P2)
Goal: Cuando UC4 detecte que el score subió de 0, UC5 quita el bloqueo por score. Si el estudiante también tiene bloqueo por daño, sigue bloqueado. El administrador puede desbloquear manualmente.

Tests for User Story 2
□ T019 [P] [US2] Pruebas en ComplianceServiceTest.java:
unblock(studentCode, reason, sourceEventId) quita blockedByScore = true

Si blockedByDamage = true, el estado pasa de BLOCKED_BY_BOTH a BLOCKED_BY_DAMAGE (no a ACTIVE)

manualUnblock(studentCode, adminCode, reason) quita el bloqueo por daño (SC-003)

manualUnblock sobre alguien sin bloqueo lanza NotBlockedException

manualUnblock registra performed_by = adminCode en el historial

□ T020 [P] [US2] Prueba ComplianceServiceIT.java (extensión):
El desbloqueo por score no quita el bloqueo por daño

El desbloqueo manual no quita el bloqueo por score

□ T021 [P] [US2] Prueba ManualUnblockIT.java:
Un desbloqueo manual se registra en el historial con performed_by correcto

Un desbloqueo manual sobre alguien con ambos bloqueos deja solo el de score

Implementation for User Story 2
□ T022 [US2] Implementar ComplianceService.unblock(...) (depende de T016)
□ T023 [US2] Implementar ComplianceService.manualUnblock(...):
Buscar UserComplianceProfile
Si no tiene bloqueos, lanzar NotBlockedException
Quitar blockedByDamage (el bloqueo por score solo se quita si el score sube de 0)
Actualizar most_recent_block_reason y most_recent_block_at
Persistir
Insertar UserBlockHistory con action = UNBLOCK, blockType = DAMAGE, performed_by = adminCode
(depende de T016)
□ T024 [US2] Implementar BlockForDamagePort en ComplianceService (misma firma que BlockUserPort, pero blockType = DAMAGE)
Checkpoint: Los desbloqueos funcionan y respetan la coexistencia de bloqueos

Phase 5: User Story 3 — Visualización del estado bloqueado en el perfil (Priority: P1)
Goal: El estudiante ve un banner rojo cuando está bloqueado, con el motivo y el score actual.

Tests for User Story 3
□ T025 [P] [US3] Prueba ComplianceControllerTest.java con @WebMvcTest, contra los fixtures:
GET /api/v1/compliance/me con BLOCKED_BY_SCORE → 200 con el message correcto

GET /api/v1/compliance/me con ACTIVE → 200 con el message correcto

GET /api/v1/compliance/me con BLOCKED_BY_BOTH → 200 con el message correcto

401 sin sesión

□ T026 [P] [US3] Prueba AdminComplianceControllerTest.java:
GET /api/v1/admin/compliance/{code} con rol ADMIN → 200

GET /api/v1/admin/compliance/{code} con rol ESTUDIANTE → 403

Implementation for User Story 3
□ T027 [US3] Implementar ComplianceController con GET /api/v1/compliance/me
□ T028 [US3] Implementar AdminComplianceController con GET /api/v1/admin/compliance/{code} y POST /api/v1/admin/compliance/{code}/unblock
□ T029 [US3] Implementar los DTOs ComplianceStatusResponse, ComplianceQueryResponse, ManualUnblockRequest
□ T030 [P] [US3] Frontend: complianceApi.js con las llamadas
□ T031 [P] [US3] Frontend: BlockBanner.jsx (banner rojo con estado, score y motivo)
□ T032 [P] [US3] Frontend: ActiveBlockBadge.jsx (badge con motivo y variación)
□ T033 [P] [US3] Frontend: ComplianceStatusCard.jsx (tarjeta con estado)
□ T034 [US3] Frontend: ProfilePage.jsx que integra el banner y los componentes
Checkpoint: El estudiante ve su estado y el motivo

Phase 6: User Story 4 — Acceso al reglamento de préstamos (Priority: P2)
Goal: El estudiante accede al reglamento desde su perfil.

Tests for User Story 4
□ T035 [P] [US4] Prueba del enlace en el frontend (con React Testing Library o similar)
Implementation for User Story 4
□ T036 [P] [US4] Frontend: LoanRegulationLink.jsx que abre el reglamento en un modal o vista
Checkpoint: El enlace al reglamento está disponible

Phase 7: Polish & Cross-Cutting Concerns
□ T037 [P] Verificar SC-001 (100% de estudiantes con score 0 quedan bloqueados antes de poder reservar)
□ T038 [P] Verificar SC-002 (100% de intentos de préstamo de un bloqueado son rechazados por M2)
□ T039 [P] Verificar SC-003 (100% de recuperaciones por encima de 0 restauran el estado, salvo bloqueo por daño)
□ T040 [P] Verificar SC-004 (bloqueo en < 1s)
□ T041 [P] Registrar en logs cada bloqueo/desbloqueo con su motivo y su origen
□ T042 [P] Documentar en el README cómo gestionar bloqueos en el perfil local
□ T043 Llevar a pendientes-clarificacion.md los NEEDS CLARIFICATION abiertos en este plan
Dependencies & Execution Order
Phase Dependencies
Setup (Phase 1): depende de plan.md

Foundational (Phase 2): depende de Setup — BLOCKS las user stories

User Story 1 (Phase 3): depende de Foundational

User Story 2 (Phase 4): depende de US1 (comparte ComplianceService)

User Story 3 (Phase 5): depende de Foundational

User Story 4 (Phase 6): depende de US3

Polish (Phase 7): depende de todas las US

Dependencias con otros casos de uso del M3
Actualizar score (UC4): lo dispara cuando el score llega a 0 (block) o sube de 0 (unblock).

Reportar novedad técnica (UC8): lo dispara cuando hay daño grave (BlockForDamagePort.block).

Calcular penalización (UC3): lo dispara indirectamente vía UC4 (la penalización reduce el score, y si llega a 0, UC4 llama a UC5).

Dependencias con otros módulos
Módulo 2: no hay integración directa. M2 consulta el estado por GET /api/v1/compliance/people/{code} (el endpoint que ya existe de UC1 del M3). UC5 enriquece la respuesta con campos opcionales sin romper los que M2 ya conoce.

Módulo 1: sin integración.

Parallel Opportunities
En Foundational: T005 a T011

En US1: T013, T014, T015 (tests)

En US2: T019, T020, T021 (tests)

En US3: T025, T026 (tests) y T030, T031, T032, T033 (frontend)

En Polish: T037 a T042

Notes
La numeración T0XX es propia de este plan

[P] tasks = different files, no dependencies

[US1] a [US4] = trazabilidad a la user story

La sección Contratos es la única fuente del JSON de UC5

Decisiones adoptadas de M2: convenciones (camelCase, MAYUSCULA_CON_GUION_BAJO, nunca null, ISO-8601 con -05:00, RFC 9457), nombres de tablas en snake_case

Decisiones propias de M3: UC5 no produce eventos Kafka; UC5 no expone endpoints nuevos a M2; solo enriquece el endpoint existente; dos tipos de bloqueo independientes; el desbloqueo por score no quita el bloqueo por daño; bloqueo duplicado es no-op; el historial registra solo cambios reales.
