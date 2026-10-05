# Implementation Plan: Calcular penalización (UC2)

**Date**: 2026-10-05
**Spec**: [calcular-penalizacion.md](../Especs/calcular-penalizacion.md)
**Plan general**: [plan.md](../Especs/plan.md)
**Módulo**: Módulo 3 — Sanciones y Cumplimiento
**Planes previos**: plan-uc-realizar-check-out.md — UC1 lo dispara cuando el check-out es `LATE`
**Planes relacionados**: [plan-uc1-actualizar-score.md](./plan-uc1-actualizar-score.md) — UC4 lo recibe cuando `afecta_score_confianza = TRUE`; [plan-uc-reportar-no-asistencia.md](./plan-uc-reportar-no-asistencia.md) — UC2 lo dispara obligatoriamente; notificar-sancion.md — UC6 lo recibe siempre

## Summary

UC3 es el **motor de penalizaciones** del módulo. Decide si una infracción merece sanción, cuántos días de suspensión aplicar según la escala progresiva, y dispara las consecuencias: la reducción del score (si la regla lo permite) y la notificación al estudiante.

Es un caso de uso **puramente interno**: no habla con M1 ni con M2. Lo disparan otros UCs del M3 (`Realizar check-out` cuando hay retraso, `Reportar no asistencia` cuando hay inasistencia), y dispara a otros UCs del M3 (`Actualizar score` y `Notificar sanción`). Toda la comunicación es por **puertos directos en memoria**, dentro del mismo JVM y la misma transacción.

**Enfoque técnico:**

1. `Calcular penalización` recibe una **solicitud de penalización** con el estudiante, el recurso, el tipo de infracción (retraso o inasistencia) y el evento de origen. La solicitud viene de `Realizar check-out` (UC1) o de `Reportar no asistencia` (UC2).
2. Determina la **regla aplicable** (`sanction_rule`) según el tipo de infracción. Si no hay regla activa, rechaza el cálculo.
3. Cuenta las **infracciones previas** del estudiante en el período configurado (30 días por defecto), sumando retrasos e inasistencias (FR-013, FR-014).
4. Determina los **días de suspensión** consultando la **escala progresiva** (`progressive_scale`). Si las infracciones superan el último rango, aplica el rango más alto (FR-011). Si no hay escala configurada, rechaza el cálculo.
5. Registra la **penalización** en `penalty_history` con la regla, la escala, el estudiante, la fecha, los días y el motivo (FR-004, FR-005).
6. **Dispara a `Actualizar score`** (UC4) solo si `sanction_rule.affects_trust_score = TRUE` (FR-006). Es una llamada directa al puerto `UpdateTrustScorePort.reduce(...)` en la misma transacción.
7. **Dispara a `Notificar sanción`** (UC6) siempre, tras registrar la penalización (FR-007). Es una llamada directa al puerto `NotifyPenaltyPort.notify(...)`, que UC6 implementa y que se encarga de publicar a M2 por Kafka.
8. El UC tiene **4 user stories**: sanción por retraso (US1), sanción por inasistencia (US2), configuración de la escala (US3), y consulta de penalizaciones vigentes (US4).

## Technical Context

**Language/Version**: Java 21 (backend); JavaScript con React + Vite (frontend)
**Primary Dependencies**: Spring Boot 4.x (Web, Data JPA, Validation, Security OAuth2 Resource Server), Flyway, springdoc-openapi. **Ninguna nueva** respecto a `plan.md`. **No usa Spring for Apache Kafka**: UC3 no publica ni consume Kafka.
**Storage**: MySQL 8. UC3 **escribe** `penalty_history`. Lee `sanction_rule`, `progressive_scale`, `system_config`, `penalty_history` (para contar infracciones previas) y `trust_score` (vía el puerto de UC4).
**Testing**: JUnit 5 + Mockito (dominio y casos de uso), `@WebMvcTest` (controladores de consulta y administración), Testcontainers MySQL (persistencia, transacciones y conteo de infracciones), ArchUnit (regla de dependencias entre capas).
**Target Platform**: Servidor Linux con JVM 21; navegador web para el frontend.
**Project Type**: Web: backend y frontend separados (React + Vite).
**Performance Goals**: El cálculo se completa en **menos de 2 segundos** desde que se recibe la solicitud (SC-001), incluyendo el conteo de infracciones y la búsqueda en la escala. Es una transacción local: dos consultas y un `INSERT`.
**Constraints**:
- No existe regla activa para el tipo de infracción → rechazar (FR-010).
- No existe rango aplicable en la escala → rechazar (FR-010).
- Penalización duplicada para el mismo evento → rechazar (FR-008).
- El conteo de infracciones considera solo el período configurado (FR-013).
- Se acumulan retrasos e inasistencias en el mismo contador (FR-014).
- La zona horaria es `America/Bogota`.
**Scale/Scope**: Mismo orden que M2 (miles de estudiantes). Una pantalla de perfil con las penalizaciones vigentes; una pantalla de administración para gestionar la escala progresiva.

## Integración con otros módulos

### Lo que M3 consume de M1 y M2 (adoptado tal cual)

**UC3 no consume eventos de M1 ni de M2.** Sus disparadores son internos del M3:
- `Realizar check-out` (UC1) cuando el check-out es `LATE`.
- `Reportar no asistencia` (UC2) siempre.

**`LoanDeclaredLost`**: M2 produce este evento (`plan-uc12` §2) cuando un préstamo se declara perdido a los 7 días. **Tu spec de `Calcular penalización` no lo menciona como disparador.** Se marca como `NEEDS CLARIFICATION NC-01` y **no se implementa** hasta que se decida.

### Lo que M3 produce para M1 y M2 (creado por M3)

**UC3 no produce eventos para M1 ni M2.** La notificación al estudiante la hace UC6 (`Notificar sanción`), que publica a M2 por Kafka. UC3 solo llama al puerto `NotifyPenaltyPort`, y UC6 se encarga.

### Endpoints que M3 expone para M2

**UC3 no expone endpoints a M2.** Los endpoints de consulta de penalizaciones son para el frontend de M3.

### Endpoints que M3 expone para su propio frontend

| Endpoint | Método | Para qué | Rol |
|---|---|---|---|
| `GET /api/v1/penalties/me` | GET | El estudiante consulta sus penalizaciones vigentes | `ESTUDIANTE` o `MONITOR` |
| `GET /api/v1/admin/penalties/{code}` | GET | El administrador consulta las penalizaciones de cualquier estudiante | `ADMIN` o `DIRECCION_PROGRAMA` |
| `GET /api/v1/admin/progressive-scales` | GET | El administrador lista la escala progresiva | `ADMIN` |
| `PUT /api/v1/admin/progressive-scales/{id}` | PUT | El administrador modifica un rango de la escala | `ADMIN` |
| `POST /api/v1/admin/progressive-scales` | POST | El administrador añade un rango a la escala | `ADMIN` |
| `DELETE /api/v1/admin/progressive-scales/{id}` | DELETE | El administrador elimina un rango | `ADMIN` |

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan.md                                    # Plan general del M3
├── calcular-penalizacion.md                    # Spec de este plan
├── actualizar-score-confianza.md               # «include» que dispara
├── notificar-sancion.md                        # «include» que dispara
├── realizar-checkout.md                       # «extend» que lo dispara
├── reportar-no-asistencia.md                   # «include» que lo dispara
└── ...                                         # resto de specs
```

### Source Code (repository root)

Solo los archivos que crea o toca este plan. La organización completa está en plan.md.

```text
backend/
├── src/main/java/com/university/sanctions/
│   ├── presentation/
│   │   ├── controller/
│   │   │   ├── PenaltyController.java              # consulta del estudiante (GET /api/v1/penalties/me)
│   │   │   ├── AdminPenaltyController.java         # consulta por administrador
│   │   │   └── ProgressiveScaleController.java     # administración de la escala
│   │   └── dto/
│   │       ├── PenaltyResponse.java
│   │       ├── ProgressiveScaleResponse.java
│   │       ├── ProgressiveScaleRequest.java
│   │       └── PenaltyCalculationResult.java
│   │
│   ├── business/
│   │   ├── service/
│   │   │   └── PenaltyService.java                 # implementa CalculatePenaltyPort
│   │   └── event/
│   │       └── PenaltyCalculatedEvent.java         # evento de dominio interno (opcional)
│   │
│   ├── domain/
│   │   ├── model/
│   │   │   ├── PenaltyRequest.java                 # comando: estudiante, recurso, tipo, origen
│   │   │   ├── PenaltyResult.java                  # resultado: días, regla, escala
│   │   │   ├── PenaltyRule.java                    # regla de penalización
│   │   │   ├── ProgressiveScale.java               # escala progresiva
│   │   │   ├── PenaltyHistory.java                 # registro de una penalización
│   │   │   ├── PenaltyType.java                    # enum: LATE_RETURN, NO_SHOW, ...
│   │   │   └── PenaltyStatus.java                  # enum: PENDING, FULFILLED, APPEALED, EXPIRED
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   └── CalculatePenaltyPort.java       # puerto de entrada
│   │   │   └── out/
│   │   │       ├── PenaltyRuleRepository.java
│   │   │       ├── ProgressiveScaleRepository.java
│   │   │       ├── PenaltyHistoryRepository.java
│   │   │       ├── UpdateTrustScorePort.java       # salida hacia UC4 (ya definido)
│   │   │       └── NotifyPenaltyPort.java          # salida hacia UC6
│   │   └── error/
│   │       ├── NoActiveRuleException.java
│   │       ├── NoApplicableScaleException.java
│   │       └── DuplicatePenaltyException.java
│   │
│   └── infrastructure/
│       ├── persistence/
│       │   └── jpa/
│       │       ├── entity/
│       │       │   ├── PenaltyRuleEntity.java
│       │       │   ├── ProgressiveScaleEntity.java
│       │       │   └── PenaltyHistoryEntity.java
│       │       ├── repository/
│       │       │   ├── JpaPenaltyRuleRepository.java
│       │       │   ├── JpaProgressiveScaleRepository.java
│       │       │   └── JpaPenaltyHistoryRepository.java
│       │       ├── adapter/
│       │       │   ├── PenaltyRuleRepositoryAdapter.java
│       │       │   ├── ProgressiveScaleRepositoryAdapter.java
│       │       │   └── PenaltyHistoryRepositoryAdapter.java
│       │       └── mapper/
│       │           ├── PenaltyRuleMapper.java
│       │           ├── ProgressiveScaleMapper.java
│       │           └── PenaltyHistoryMapper.java
│       └── config/
│           ├── SecurityConfig.java
│           └── UseCasesConfig.java
│
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       ├── V6__penalty_rule_and_scale.sql          # sanction_rule + progressive_scale
│       ├── V7__penalty_history.sql                 # penalty_history
│       └── V8__seed_penalty_data.sql               # datos iniciales de reglas y escala
│
└── src/test/java/com/university/sanctions/
    ├── contract/
    │   ├── PenaltyControllerTest.java
    │   ├── AdminPenaltyControllerTest.java
    │   └── ProgressiveScaleControllerTest.java
    ├── integration/
    │   ├── PenaltyServiceIT.java                   # persistencia + transacciones
    │   ├── PenaltyCalculationIT.java               # conteo de infracciones
    │   └── ProgressiveScaleIT.java                 # administración de la escala
    └── unit/
        ├── PenaltyRuleTest.java
        ├── ProgressiveScaleTest.java
        └── PenaltyServiceTest.java

frontend/
└── src/
    ├── pages/
    │   ├── PenaltiesPage.jsx                        # perfil del estudiante
    │   └── AdminProgressiveScalesPage.jsx           # administración
    ├── components/
    │   ├── PenaltyCard.jsx                          # tarjeta por sanción
    │   ├── PenaltyEmptyState.jsx                    # "No tienes penalizaciones activas"
    │   └── ProgressiveScaleTable.jsx                # tabla editable
    └── services/
        └── penaltyApi.js                            # cliente HTTP
```

Structure Decision: se respeta la estructura de capas de plan.md (presentation / business / domain / infrastructure). El paquete raíz es com.university.sanctions. La lógica del cálculo vive en business/service/PenaltyService.java (orquesta) y el modelo en domain/model/. Los puertos de salida hacia UC4 y UC6 viven en domain/port/out/ para que el dominio no dependa de las implementaciones. La persistencia vive en infrastructure. El frontend se organiza por páginas y componentes reutilizables.

Decisiones de diseño de este caso de uso
Dónde queda cada FR.

FR	Qué pide	Dónde se implementa
FR-001	Calcular penalización cuando Realizar check-out indique retraso	PenaltyService.calculate(...) — lo llama UC1
FR-002	Calcular obligatoriamente cuando Reportar no asistencia	PenaltyService.calculate(...) — lo llama UC2
FR-003	Determinar días de suspensión consultando la escala progresiva	ProgressiveScaleRepository.findByInfractionCount(...)
FR-004	Registrar la infracción en penalty_history	PenaltyHistoryRepository.save(...)
FR-005	Registrar el motivo exacto tomado de sanction_rule	PenaltyRule.reason
FR-006	Disparar Actualizar score si affects_trust_score = TRUE	PenaltyService llama a UpdateTrustScorePort.reduce(...)
FR-007	Disparar Notificar sanción tras registrar	PenaltyService llama a NotifyPenaltyPort.notify(...)
FR-008	Impedir cálculo duplicado para el mismo evento	Clave única en penalty_history.source_event_id
FR-009	Informar cuando un check-out no supere el umbral	PenaltyService devuelve un resultado NO_PENALTY (lo maneja UC1)
FR-010	Rechazar cuando no haya regla o escala	NoActiveRuleException y NoApplicableScaleException
FR-011	Aplicar el rango más alto si se supera el último	ProgressiveScaleRepository.findHighest(...)
FR-012	Permitir gestionar la escala (CRUD)	ProgressiveScaleController
FR-013	Contar infracciones solo del período configurado	PenaltyHistoryRepository.countByStudentAndPeriod(...)
FR-014	Acumular retrasos e inasistencias en el mismo contador	La consulta no filtra por tipo
FR-015	Exponer penalizaciones vigentes	GET /api/v1/penalties/me y GET /api/v1/admin/penalties/{code}
FR-016	UI: tarjeta por sanción con fecha límite	PenaltyCard.jsx
FR-017	UI: mensaje cuando no hay penalizaciones	PenaltyEmptyState.jsx
Decisiones de diseño justificadas.

1. UC3 no habla con M1 ni con M2. Es un caso de uso puramente interno. Sus disparadores son otros UCs del M3, y sus consecuencias también. La comunicación con M2 la hace UC6 (Notificar sanción), que publica a Kafka. Esto mantiene la separación de responsabilidades: UC3 decide la penalización, UC6 la comunica.

2. Comunicación interna por puertos directos en memoria. Los tres UCs que participan (Realizar check-out, Reportar no asistencia como disparadores; Actualizar score y Notificar sanción como disparados) viven en el mismo JVM. Una llamada directa a un puerto Java es:

Atómica: si falla el score, falla la penalización.

Rápida: sin serialización ni broker.

Simple: sin topics internos que mantener.

Kafka se reserva para comunicación entre módulos (M3 → M2), no dentro de M3.

3. La escala progresiva se consulta por número de infracciones. ProgressiveScale mapea infractionCount → suspensionDays. Si el estudiante tiene 4 infracciones previas y hay un rango para 5, se aplica el rango de 5 (FR-011). Si no hay rango para su número exacto, se busca el rango más alto configurado.

4. El conteo de infracciones acumula retrasos e inasistencias. La consulta a penalty_history no filtra por penalty_type; cuenta todas las infracciones del estudiante en el período. Esto es lo que dice FR-014.

5. El período de conteo es configurable. system_config guarda penalty.infraction-count-days = 30. Si cambia, el conteo cambia sin tocar código.

6. Idempotencia por source_event_id. Cada solicitud de penalización trae un sourceEventId (el eventId del evento que la originó: el check-out o el reporte de no asistencia). penalty_history tiene una clave única en source_event_id. Si el mismo evento intenta generar dos penalizaciones, el segundo INSERT falla y se lanza DuplicatePenaltyException.

7. UC3 no decide si el estudiante se bloquea. Eso es de UC4 → UC5. UC3 solo dispara la reducción del score; si eso lleva a 0, UC4 llama a UC5.

8. UC3 no publica a Kafka. La notificación al estudiante la hace UC6 (Notificar sanción), que implementa NotifyPenaltyPort. UC3 solo llama al puerto.

9. Los endpoints de consulta de penalizaciones vigentes son para el frontend de M3. No son para M2. M2 no consulta penalizaciones; consulta sanciones (a través de GET /api/v1/compliance/people/{code}), que es un concepto distinto (la sanción es la penalización vista desde fuera).

## Contratos

Se aplican las convenciones comunes de M2 (camelCase, MAYUSCULA_CON_GUION_BAJO, nunca null, ISO-8601 con -05:00, RFC 9457). M3 las adopta para sus propios contratos.

### 1. Puerto de entrada — CalculatePenaltyPort

El contrato que usan UC1 y UC2. No es REST ni Kafka: es una interfaz Java.

```java
public interface CalculatePenaltyPort {

    /**
     * Calcula la penalización correspondiente a una infracción.
     * Lo llama Realizar check-out (UC1) cuando el check-out es LATE.
     * Lo llama Reportar no asistencia (UC2) siempre.
     *
     * @param request la solicitud con estudiante, recurso, tipo y origen
     * @return el resultado del cálculo
     * @throws NoActiveRuleException si no hay regla activa para el tipo
     * @throws NoApplicableScaleException si no hay escala aplicable
     * @throws DuplicatePenaltyException si ya existe una penalización para el mismo evento
     */
    PenaltyResult calculate(PenaltyRequest request);
}
```

PenaltyRequest:

```java
public record PenaltyRequest(
    String studentCode,
    String resourceId,
    PenaltyType penaltyType,      // LATE_RETURN, NO_SHOW
    String sourceEventId,          // eventId del check-out o del reporte de no asistencia
    String sourceEventType,        // "CHECK_OUT" o "NO_SHOW_REPORT"
    Instant occurredAt             // cuándo ocurrió la infracción
) {}
```

PenaltyResult:

```java
public record PenaltyResult(
    String penaltyId,
    String studentCode,
    PenaltyType penaltyType,
    String reason,                 // motivo tomado de sanction_rule
    int suspensionDays,            // días de suspensión según la escala
    int previousInfractions,       // infracciones previas en el período
    int appliedScaleFrom,          // el rango de la escala que se aplicó
    boolean trustScoreAffected,    // si se disparó a UC4
    boolean notificationTriggered, // si se disparó a UC6
    Instant occurredAt
) {}
```

### 2. Puerto de salida — UpdateTrustScorePort (ya definido en UC4)

Lo llama PenaltyService cuando sanction_rule.affects_trust_score = TRUE.

```java
public interface UpdateTrustScorePort {
    TrustScoreChangeResult reduce(String studentCode, int points, String reason, String sourceEventId);
    // los otros métodos son de UC4, no de UC3
}
```

Ya está definido en UC4. UC3 solo lo usa.

### 3. Puerto de salida — NotifyPenaltyPort

Lo llama PenaltyService siempre, tras registrar la penalización.

```java
public interface NotifyPenaltyPort {
    void notify(PenaltyResult result);
}
```

Se implementa en UC6 (Notificar sanción), que se encarga de publicar a M2 por Kafka.

### 4. Endpoints REST — consulta del estudiante

GET /api/v1/penalties/me
Devuelve las penalizaciones vigentes del estudiante autenticado. Rol ESTUDIANTE o MONITOR.

```json
{
  "studentCode": "2023123456",
  "activePenalties": [
    {
      "id": "pen-88391",
      "penaltyType": "LATE_RETURN",
      "reason": "Retraso en entrega",
      "suspensionDays": 3,
      "startDate": "2026-10-05T10:00:00-05:00",
      "endDate": "2026-10-08T22:00:00-05:00",
      "status": "PENDING",
      "resourceId": "ACT-004512",
      "resourceName": "Libro de Cálculo I"
    }
  ],
  "message": null
}
```

Cuando no hay penalizaciones vigentes:

```json
{
  "studentCode": "2023123456",
  "activePenalties": [],
  "message": "No tienes penalizaciones activas. ¡Sigue así!"
}
```

| Campo | Tipo | Nota |
|---|---|---|
| `activePenalties` | array | Solo las que están `PENDING` y su `endDate` es futuro |
| `suspensionDays` | int | Días de suspensión aplicados según la escala |
| `startDate` / `endDate` | instante ISO-8601 | El rango de la suspensión |
| `status` | enum | `PENDING`, `FULFILLED`, `APPEALED`, `EXPIRED` |
| `message` | string | Se omite si hay penalizaciones; se incluye si no hay |

### 5. Endpoints REST — consulta del administrador

GET /api/v1/admin/penalties/{code}
Igual que el del estudiante, pero para cualquier code. Rol ADMIN o DIRECCION_PROGRAMA.

Incluye penalizaciones PENDING, FULFILLED, APPEALED y EXPIRED (todas), no solo las vigentes.

### 6. Endpoints REST — administración de la escala progresiva

GET /api/v1/admin/progressive-scales
Rol ADMIN.

```json
{
  "scales": [
    { "id": 1, "infractionCount": 1, "suspensionDays": 1, "active": true },
    { "id": 2, "infractionCount": 2, "suspensionDays": 2, "active": true },
    { "id": 3, "infractionCount": 3, "suspensionDays": 3, "active": true },
    { "id": 4, "infractionCount": 4, "suspensionDays": 5, "active": true },
    { "id": 5, "infractionCount": 5, "suspensionDays": 7, "active": true }
  ]
}
```

PUT /api/v1/admin/progressive-scales/{id}

```json
{
  "infractionCount": 2,
  "suspensionDays": 5,
  "active": true
}
```

Respuesta: `200 OK` con el recurso actualizado.

POST /api/v1/admin/progressive-scales

```json
{
  "infractionCount": 6,
  "suspensionDays": 45,
  "active": true
}
```

Respuesta: `201 Created` con el recurso creado.

DELETE /api/v1/admin/progressive-scales/{id}

Respuesta: `204 No Content`.

### 7. Errores

Código HTTP	code	Cuándo
400	INVALID_PENALTY_REQUEST	El PenaltyRequest no tiene los campos obligatorios
404	NO_ACTIVE_RULE	No hay regla activa para el tipo de infracción (FR-010)
404	NO_APPLICABLE_SCALE	No hay rango aplicable en la escala (FR-010)
409	DUPLICATE_PENALTY	Ya existe una penalización para el mismo sourceEventId (FR-008)
500	PERSISTENCE_ERROR	Fallo al persistir la penalización
ProblemDetail de ejemplo:

```json
{
  "type": "https://sanctions.unimagdalena.edu.co/errors/no-active-rule",
  "title": "No hay regla activa",
  "status": 404,
  "detail": "No se encontró una regla activa para el tipo de infracción LATE_RETURN.",
  "instance": "/api/v1/penalties/calculate",
  "code": "NO_ACTIVE_RULE",
  "penaltyType": "LATE_RETURN"
}
```
### 8. Tablas

sanction_rule
Columna	Tipo	Nota
id	bigint PK auto	
penalty_type	varchar(30) NOT NULL	LATE_RETURN, NO_SHOW, etc.
reason	varchar(255) NOT NULL	Motivo legible
min_threshold_minutes	int, nulo	Umbral mínimo (solo para retraso)
affects_trust_score	boolean NOT NULL	Si dispara a UC4
active	boolean NOT NULL DEFAULT true	
created_at	timestamp NOT NULL	
updated_at	timestamp NOT NULL	
```sql
CREATE UNIQUE INDEX sanction_rule_type_active ON sanction_rule (penalty_type) WHERE active = true;
```
El índice único parcial garantiza que solo haya una regla activa por tipo.

progressive_scale
Columna	Tipo	Nota
id	bigint PK auto	
infraction_count	int NOT NULL	Número de infracciones previas
suspension_days	int NOT NULL	Días de suspensión a aplicar
active	boolean NOT NULL DEFAULT true	
created_at	timestamp NOT NULL	
updated_at	timestamp NOT NULL	
```sql
CREATE UNIQUE INDEX progressive_scale_count_active ON progressive_scale (infraction_count) WHERE active = true;
penalty_history
```
Columna	Tipo	Nota
id	varchar(36) PK	pen-XXXXX
student_code	varchar(20) NOT NULL	
resource_id	varchar(50) NOT NULL	
penalty_type	varchar(30) NOT NULL	LATE_RETURN, NO_SHOW
rule_id	bigint FK NOT NULL	La regla aplicada
scale_id	bigint FK NOT NULL	El rango aplicado
reason	varchar(255) NOT NULL	Copiado de la regla
suspension_days	int NOT NULL	Copiado de la escala
previous_infractions	int NOT NULL	Infracciones previas al momento del cálculo
source_event_id	varchar(36) NOT NULL	Para idempotencia
source_event_type	varchar(30) NOT NULL	CHECK_OUT o NO_SHOW_REPORT
occurred_at	timestamp NOT NULL	Cuándo ocurrió la infracción
start_date	timestamp NOT NULL	Inicio de la suspensión
end_date	timestamp NOT NULL	Fin de la suspensión
status	varchar(20) NOT NULL	PENDING, FULFILLED, APPEALED, EXPIRED
created_at	timestamp NOT NULL	
```sql
CREATE UNIQUE INDEX penalty_history_source_event ON penalty_history (source_event_id);
CREATE INDEX penalty_history_student_period ON penalty_history (student_code, occurred_at DESC);
CREATE INDEX penalty_history_status ON penalty_history (status, end_date);
```
La clave única en source_event_id es lo que hace cumplir FR-008 a nivel de base de datos.

system_config
Columna	Tipo	Nota
key	varchar(50) PK	
value	varchar(255) NOT NULL	
description	varchar(255)	
updated_at	timestamp NOT NULL	
Configuraciones relevantes:

penalty.infraction-count-days = 30 (FR-013)

No hay tabla de relación con M1 ni M2 en este UC. UC3 es puramente interno.

### 9. Tipos del frontend

```js
export const PenaltyType = {
  LATE_RETURN: "LATE_RETURN",
  NO_SHOW: "NO_SHOW",
};

export const PenaltyStatus = {
  PENDING: "PENDING",
  FULFILLED: "FULFILLED",
  APPEALED: "APPEALED",
  EXPIRED: "EXPIRED",
};

export const getMyPenalties = async () => {
  const response = await fetch("/api/v1/penalties/me");
  if (!response.ok) throw await response.json();
  return response.json();
};

export const getStudentPenalties = async (studentCode) => {
  const response = await fetch(`/api/v1/admin/penalties/${studentCode}`);
  if (!response.ok) throw await response.json();
  return response.json();
};

export const getProgressiveScales = async () => {
  const response = await fetch("/api/v1/admin/progressive-scales");
  if (!response.ok) throw await response.json();
  return response.json();
};

export const updateProgressiveScale = async (id, data) => {
  const response = await fetch(`/api/v1/admin/progressive-scales/${id}`, {
    method: "PUT",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw await response.json();
  return response.json();
};
```
### 10. Fixtures compartidos

```text
backend/src/test/resources/contracts/
├── api-penalties-me-200.json
├── api-penalties-me-empty-200.json
├── api-penalties-admin-200.json
├── api-progressive-scales-200.json
├── api-progressive-scale-201.json
├── api-penalty-404-no-active-rule.json
├── api-penalty-404-no-applicable-scale.json
└── api-penalty-409-duplicate.json
Phase 1: Setup
□ T001 Añadir a application.yml los parámetros: sanctions.penalty.infraction-count-days=30, y configurar las propiedades de las reglas si aplica
□ T002 [P] Extender SanctionsProperties con esos parámetros y validarlos al arrancar
Phase 2: Foundational (Blocking Prerequisites)
□ T003 Escribir V6__penalty_rule_and_scale.sql con las tablas sanction_rule y progressive_scale de Contratos §8
□ T004 Escribir V7__penalty_history.sql con la tabla penalty_history y sus índices (incluida la clave única en source_event_id)
□ T005 Escribir V8__seed_penalty_data.sql con datos iniciales: una regla para LATE_RETURN, una para NO_SHOW, y la escala progresiva por defecto (1→1, 2→2, 3→3, 4→5, 5→7)
□ T006 [P] Crear el modelo de dominio: PenaltyRequest, PenaltyResult, PenaltyRule, ProgressiveScale, PenaltyHistory, PenaltyType, PenaltyStatus en domain/model/
□ T007 [P] Definir los repositorios PenaltyRuleRepository, ProgressiveScaleRepository y PenaltyHistoryRepository en domain/port/out/
□ T008 [P] Definir CalculatePenaltyPort en domain/port/in/, y verificar que UpdateTrustScorePort (de UC4) y NotifyPenaltyPort (nuevo) están en domain/port/out/
□ T009 [P] Crear NoActiveRuleException, NoApplicableScaleException y DuplicatePenaltyException en domain/error/
□ T010 [P] Implementar las entidades JPA PenaltyRuleEntity, ProgressiveScaleEntity y PenaltyHistoryEntity y sus repositorios Spring Data
□ T011 [P] Implementar los adaptadores PenaltyRuleRepositoryAdapter, ProgressiveScaleRepositoryAdapter y PenaltyHistoryRepositoryAdapter con sus mappers
Checkpoint: Existe dónde guardar la penalización, y los puertos hacia UC4 y UC6

Phase 3: User Story 1 — Sanción progresiva por retraso en check-out (Priority: P1)
Goal: Cuando Realizar check-out indique que el tiempo de uso superó el umbral, Calcular penalización cuenta las infracciones previas en el período, determina los días de suspensión según la escala, registra la penalización, dispara Actualizar score si la regla lo permite, y dispara Notificar sanción siempre.

Tests for User Story 1
□ T012 [P] [US1] Pruebas en PenaltyServiceTest.java (unit, con repositorios y puertos falsos):
Primera infracción por retraso: 0 previas → 1 día de suspensión, affects_trust_score = TRUE, dispara UC4 y UC6

Quinta infracción por retraso: 4 previas → 7 días de suspensión, dispara UC4 y UC6

Infracción que no supera el umbral: no calcula penalización, devuelve NO_PENALTY

Sin regla activa para LATE_RETURN: lanza NoActiveRuleException

Sin escala aplicable: lanza NoApplicableScaleException

sourceEventId duplicado: lanza DuplicatePenaltyException

Regla con affects_trust_score = FALSE: no dispara UC4, pero sí UC6

□ T013 [P] [US1] Prueba PenaltyServiceIT.java con Testcontainers MySQL:
La penalización, la actualización del score (vía puerto), y la notificación encolada ocurren en la misma transacción

Un fallo a mitad → rollback completo

```
La clave única en source_event_id rechaza el segundo intento

□ T014 [P] [US1] Prueba PenaltyCalculationIT.java:
El conteo de infracciones previas considera solo el período configurado

Retrasos e inasistencias se acumulan en el mismo contador (FR-014)

El conteo es correcto con datos de prueba (0, 1, 4, 5, 10 infracciones)

Implementation for User Story 1
□ T015 [US1] Implementar PenaltyService.calculate(...):
Validar el PenaltyRequest
Buscar la regla activa por penaltyType; si no existe → NoActiveRuleException
Contar infracciones previas en el período (30 días)
Buscar el rango de escala aplicable; si no existe → NoApplicableScaleException
Si el número de infracciones supera el último rango, aplicar el rango más alto (FR-011)
Calcular startDate (ahora) y endDate (ahora + suspensionDays días)
Persistir PenaltyHistory
Si affects_trust_score = TRUE, llamar a UpdateTrustScorePort.reduce(...)
Llamar a NotifyPenaltyPort.notify(...)
Devolver PenaltyResult
(depende de T006 a T011)
□ T016 [US1] Implementar el @Transactional en PenaltyService.calculate(...)
□ T017 [US1] Implementar el método de conteo de infracciones en PenaltyHistoryRepositoryAdapter
Checkpoint: La sanción por retraso funciona de forma aislada

Phase 4: User Story 2 — Sanción progresiva por inasistencia (Priority: P1)
Goal: Cuando Reportar no asistencia (UC2) indique una inasistencia, Calcular penalización aplica obligatoriamente la sanción progresiva correspondiente.

Independent Test: Invocar CalculatePenaltyPort.calculate(...) con penaltyType = NO_SHOW sobre un estudiante con 2 infracciones previas, y comprobar que:

Se aplican 3 días de suspensión.

Se registra en penalty_history.

Se dispara Actualizar score (si la regla lo permite).

Se dispara Notificar sanción.

Tests for User Story 2
□ T018 [P] [US2] Pruebas en PenaltyServiceTest.java:
Primera inasistencia: 0 previas → 1 día

Sexta inasistencia: 5 previas → 30 días (rango más alto)

affects_trust_score de NO_SHOW es TRUE → dispara UC4 y UC6

□ T019 [P] [US2] Prueba PenaltyCalculationIT.java (extensión): inasistencias y retrasos se acumulan en el mismo contador
Implementation for User Story 2
□ T020 [US2] Verificar que PenaltyService.calculate(...) funciona con penaltyType = NO_SHOW. No añade código nuevo: es la misma lógica con otro tipo.
Checkpoint: La sanción por inasistencia funciona

Phase 5: User Story 3 — Configuración de la escala progresiva (Priority: P3)
Goal: El administrador puede gestionar (crear, modificar, eliminar) los rangos de la escala progresiva a través de la API.

Tests for User Story 3
□ T021 [P] [US3] Pruebas en ProgressiveScaleControllerTest.java con @WebMvcTest:
GET /api/v1/admin/progressive-scales → 200 con la lista

PUT /api/v1/admin/progressive-scales/{id} → 200 con el recurso actualizado

POST /api/v1/admin/progressive-scales → 201 con el recurso creado

DELETE /api/v1/admin/progressive-scales/{id} → 204

403 si el rol no es ADMIN

□ T022 [P] [US3] Prueba ProgressiveScaleIT.java:
Modificar un rango → las siguientes sanciones usan el nuevo valor

Añadir un rango → se aplica cuando el estudiante alcanza ese número

El índice único parcial impide dos rangos activos con el mismo infraction_count

Implementation for User Story 3
□ T023 [US3] Implementar ProgressiveScaleController con los endpoints de Contratos §6
□ T024 [US3] Implementar los DTOs ProgressiveScaleResponse y ProgressiveScaleRequest
□ T025 [US3] Implementar los métodos de gestión en ProgressiveScaleRepositoryAdapter
Checkpoint: La escala es configurable

Phase 6: User Story 4 — Consulta de penalizaciones vigentes (Priority: P1)
Goal: El estudiante consulta sus penalizaciones vigentes; el administrador consulta las de cualquier estudiante. La UI muestra una tarjeta por sanción con su fecha límite, o un mensaje cuando no hay.

Tests for User Story 4
□ T026 [P] [US4] Prueba PenaltyControllerTest.java con @WebMvcTest, contra los fixtures:
GET /api/v1/penalties/me con penalizaciones → 200 con la lista

GET /api/v1/penalties/me sin penalizaciones → 200 con message

401 sin sesión

□ T027 [P] [US4] Prueba AdminPenaltyControllerTest.java:
GET /api/v1/admin/penalties/{code} con rol ADMIN → 200

GET /api/v1/admin/penalties/{code} con rol ESTUDIANTE → 403

404 si el estudiante no existe

Implementation for User Story 4
□ T028 [US4] Implementar PenaltyController con GET /api/v1/penalties/me
□ T029 [US4] Implementar AdminPenaltyController con GET /api/v1/admin/penalties/{code}
□ T030 [US4] Implementar el método de consulta de penalizaciones vigentes en PenaltyHistoryRepositoryAdapter (filtra status = PENDING y end_date > now)
□ T031 [P] [US4] Frontend: penaltyApi.js con las llamadas
□ T032 [P] [US4] Frontend: PenaltyCard.jsx (tarjeta por sanción con fecha límite e ícono de calendario)
□ T033 [P] [US4] Frontend: PenaltyEmptyState.jsx ("No tienes penalizaciones activas. ¡Sigue así!")
□ T034 [US4] Frontend: PenaltiesPage.jsx que integra los componentes
Checkpoint: La consulta está disponible

Phase 7: Polish & Cross-Cutting Concerns
□ T035 [P] Verificar SC-001 (cálculo < 2s)
□ T036 [P] Verificar SC-002 (100% de sanciones aplican la escala correcta)
□ T037 [P] Verificar SC-003 (100% de sanciones disparan UC4 y UC6)
□ T038 [P] Verificar SC-004 (0% de sanciones duplicadas)
□ T039 [P] Verificar SC-005 (100% de intentos sin regla o escala son rechazados)
□ T040 [P] Registrar en logs cada cálculo con su estudiante, tipo, días y resultado
□ T041 [P] Documentar en el README cómo configurar la escala en el perfil local
□ T042 Llevar a pendientes-clarificacion.md los NEEDS CLARIFICATION abiertos en este plan
Dependencies & Execution Order
Phase Dependencies
Setup (Phase 1): depende de plan.md

Foundational (Phase 2): depende de Setup — BLOCKS las user stories

User Story 1 (Phase 3): depende de Foundational

User Story 2 (Phase 4): depende de US1 (misma lógica, otro tipo)

User Story 3 (Phase 5): depende de Foundational

User Story 4 (Phase 6): depende de Foundational

Polish (Phase 7): depende de todas las US

Dependencias con otros casos de uso del M3
Realizar check-out (UC1): lo dispara cuando el check-out es LATE. UC1 llama a CalculatePenaltyPort.calculate(...).

Reportar no asistencia (UC2): lo dispara obligatoriamente. UC2 llama a CalculatePenaltyPort.calculate(...).

Actualizar score (UC4): lo recibe cuando affects_trust_score = TRUE. UC4 implementa UpdateTrustScorePort, que UC3 llama.

Notificar sanción (UC6): lo recibe siempre. UC6 implementa NotifyPenaltyPort, que UC3 llama.

Dependencias con otros módulos
Módulo 1: sin integración en este UC.

Módulo 2: sin integración en este UC. La notificación a M2 la hace UC6.

Parallel Opportunities
En Foundational: T006 a T011

En US1: T012, T013, T014 (tests)

En US2: T018, T019 (tests)

En US3: T021, T022 (tests) — paralelizable con US1

En US4: T026, T027 (tests) y T031, T032, T033 (frontend)

En Polish: T035 a T041

Notes
La numeración T0XX es propia de este plan

[P] tasks = different files, no dependencies

[US1] a [US4] = trazabilidad a la user story

La sección Contratos es la única fuente del JSON de UC3

Decisiones adoptadas de M2: convenciones (camelCase, MAYUSCULA_CON_GUION_BAJO, nunca null, ISO-8601 con -05:00, RFC 9457), nombres de tablas en snake_case

Decisiones propias de M3: UC3 no habla con M1 ni con M2; comunicación interna por puertos directos en memoria; idempotencia por source_event_id; la escala progresiva se consulta por número de infracciones; el conteo acumula retrasos e inasistencias.
