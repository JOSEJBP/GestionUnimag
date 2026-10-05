# Implementation Plan: Reportar no asistencia (UC7)

**Date**: 2026-10-05
**Spec**: [reportar-no-asistencia.md](./reportar-no-asistencia.md)
**Plan general**: [plan.md](./plan.md)
**Módulo**: Módulo 3 — Sanciones y Cumplimiento
**Planes previos**: plan-uc-realizar-check-out.md — UC6 crea la `Usage` local que este UC consulta para saber si hubo check-in (ver NC-01)
**Planes relacionados**: [plan-uc2-calcular-penalizacion.md](./plan-uc2-calcular-penalizacion.md) — UC2 lo dispara obligatoriamente («include»); notificar-sancion.md — UC5 notifica la sanción resultante (lo dispara UC2, no este UC)

## Summary

UC7 es el **disparador de las sanciones por inasistencia**. Un profesor o administrador reporta que un estudiante no asistió a su reserva; el sistema valida que el reporte sea legítimo (la reserva existe, no tuvo check-in, su ventana ya venció y no hay otro reporte), lo registra y dispara obligatoriamente `Calcular penalización` (UC2) con `penaltyType = NO_SHOW`. Sin este UC, la inasistencia no deja rastro ni consecuencia.

UC7 **no decide la sanción** (eso es de UC2), **no toca el score** (UC1) y **no notifica la sanción** (UC5). Solo valida, registra, publica la confirmación del registro para M2 y dispara el cálculo.

**Enfoque técnico:**

1. Expone un **puerto de entrada** (`ReportNoShowPort`) que usa el controlador REST. No tiene otros disparadores internos: el actor siempre es una persona (profesor o administrador).
2. Para validar consulta la reserva a través de un **puerto de salida** (`ReservationLookupPort`) que devuelve una vista mínima: estudiante, recurso, ventana de tiempo, si hubo check-in y si fue cancelada. M2 es dueño de las reservas; M3 no las modifica.
3. Valida en este orden: reserva identificable → estudiante identificable → no duplicado → sin check-in → ventana finalizada (FR-002, FR-003, FR-004, FR-007, FR-008).
4. **Registrar y penalizar son dos pasos con transacciones separadas.** La spec exige que, si la penalización falla, *el reporte quede registrado* y se marque la inconsistencia para revisión. Por eso no se envuelve todo en una sola transacción (a diferencia de UC2/UC1, donde sí es atómico):
   - **T1**: validar + `INSERT no_show_report` (`penalty_status = PENDING`) + `INSERT outbox_message` (`NoShowReported`).
   - **T2**: llamar a `CalculatePenaltyPort.calculate(...)` (UC2, transacción propia).
   - **T3**: marcar el reporte `TRIGGERED` (con `penalty_id`) o `FAILED` + `needs_review = true`.
5. La **confirmación del registro** se publica a M2 por Kafka vía outbox (`NoShowReported`). No bloquea el registro: si Kafka o M2 están caídos, el evento espera (spec: Reactivo/cola).
6. Si la penalización falló, un administrador puede **reintentar** el disparo. El reintento es idempotente porque UC2 usa `source_event_id = reportId` con clave única.
7. El UC tiene **4 user stories**: reporte exitoso (US1), rechazos por validación (US2), inconsistencia y reintento de penalización (US3) y consulta/pantalla (US4). La spec solo declara una user story con 3 escenarios; US2 y US3 se derivan de esos escenarios y de los Edge Cases.

## Technical Context

**Language/Version**: Java 21 (backend); JavaScript con React + Vite (frontend)
**Primary Dependencies**: Spring Boot 4.x (Web, Data JPA, Validation, Security OAuth2 Resource Server, Spring for Apache Kafka solo para el productor vía outbox), Flyway, springdoc-openapi. **Ninguna nueva** respecto a `plan.md`.
**Storage**: MySQL 8. UC7 **escribe** `no_show_report` y `outbox_message`. **Lee** la vista local de reservas/`usage` (ver NC-01) y, vía UC2, `penalty_history`.
**Testing**: JUnit 5 + Mockito (dominio y casos de uso), `@WebMvcTest` (controladores), Testcontainers MySQL (persistencia, unicidad, transacciones separadas), Testcontainers Kafka (publicación de `NoShowReported`), ArchUnit (regla de dependencias entre capas).
**Target Platform**: Servidor Linux con JVM 21; navegador web para el frontend.
**Project Type**: Web: backend y frontend separados (React + Vite).
**Performance Goals**: registrar el reporte **y** disparar la penalización en **menos de 2 segundos** (SC-004). T1 y T3 son transacciones locales cortas; T2 es la transacción de UC2 (también < 2 s por su propio SC). La publicación a Kafka queda fuera del camino crítico (outbox).
**Constraints**:
- Un solo reporte por reserva, garantizado por clave única en BD (FR-007, SC-002).
- Reserva con check-in registrado → rechazo (FR-003, SC-003).
- Ventana de la reserva aún vigente → rechazo (FR-004).
- Si el cálculo de penalización falla, el reporte **no se revierte**: queda `FAILED` + `needs_review` (Edge Case "Error al disparar el cálculo").
- Si falla el registro, no debe quedar un registro incompleto (Edge Case "Error al registrar"): T1 es atómica.
- Los eventos a M2 no bloquean ni se revierten si M2/Kafka no están disponibles.
- La zona horaria es `America/Bogota`; la comparación con el fin de ventana usa un `Clock` inyectado.
**Scale/Scope**: Mismo orden que M2 (miles de estudiantes). Una pantalla de reporte y consulta para profesores/administradores y un panel de revisión de inconsistencias para administración.

## Integración con otros módulos

### Lo que M3 consume de M1 y M2 (adoptado tal cual)

| Evento | Topic | Para qué lo consume UC7 | Dónde lo definió M2 |
|---|---|---|---|
| `NoShowReportAcknowledged` | `module2.reservation.no-show-ack.v1` | Registrar `m2_acknowledged_at` en el reporte. **Informativo**: el flujo no depende de él (NC-03) | `plan-uc9` §2 |

**De M1:** no aplica; UC7 no habla con M1.

**Datos de la reserva:** M2 es el dueño de la reserva. Cómo llega a M3 la información de reservas *sin check-in* está abierto en **NC-01**. El plan lo aísla detrás de `ReservationLookupPort` para que resolverlo no cambie el dominio.

**Estos contratos se adoptan tal cual.** M3 no los redefine.

### Lo que M3 produce para M2 (creado por M3)

La spec pide que el registro del reporte publique una confirmación que M2 consume de forma asíncrona para informar al profesor o al estudiante.

| Evento | Topic | Cuándo se emite | Por qué M2 lo necesita | FR |
|---|---|---|---|---|
| `NoShowReported` | `module3.sanctions.no-show-reported.v1` | Al registrar el reporte (T1, vía outbox) | M2 informa al profesor/estudiante y puede marcar la reserva como inasistida | FR-005, spec §Integración |

**Este topic es creado por M3.** M2 lo consume cuando esté listo. M3 lo publica igual.

> La **sanción** resultante (días de suspensión, cambio de score) la notifica UC5 por sus propios eventos. `NoShowReported` solo confirma que **el reporte quedó registrado**.

### Endpoints que M3 expone para M2

**UC7 no expone endpoints a M2.**

### Endpoints que M3 expone para su propio frontend

| Endpoint | Método | Para qué | Rol |
|---|---|---|---|
| `POST /api/v1/no-show-reports` | POST | Reportar la inasistencia de una reserva | `PROFESOR` o `ADMIN` |
| `GET /api/v1/no-show-reports/{id}` | GET | Consultar un reporte | `PROFESOR`, `ADMIN` o `DIRECCION_PROGRAMA` |
| `GET /api/v1/no-show-reports` | GET | Listar reportes (el profesor ve los suyos; admin ve todos y filtra por `needsReview`) | `PROFESOR`, `ADMIN` o `DIRECCION_PROGRAMA` |
| `POST /api/v1/admin/no-show-reports/{id}/retry-penalty` | POST | Reintentar el disparo de la penalización de un reporte `FAILED` | `ADMIN` |

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan.md                                    # Plan general del M3
├── reportar-no-asistencia.md                   # Spec de este plan
├── calcular-penalizacion.md                    # «include» que dispara
└── ...                                         # resto de specs
```

### Source Code (repository root)

Solo los archivos que crea o toca este plan. La organización completa está en `plan.md`.

```text
backend/
├── src/main/java/com/university/sanctions/
│   ├── presentation/
│   │   ├── controller/
│   │   │   ├── NoShowReportController.java          # POST, GET, GET list
│   │   │   └── AdminNoShowReportController.java     # POST retry-penalty
│   │   └── dto/
│   │       ├── NoShowReportRequest.java
│   │       ├── NoShowReportResponse.java
│   │       └── NoShowReportListResponse.java
│   │
│   ├── business/
│   │   ├── service/
│   │   │   ├── NoShowReportService.java             # implementa ReportNoShowPort (orquesta T1→T2→T3)
│   │   │   ├── NoShowRegistrar.java                 # @Transactional: validar + guardar + outbox (T1)
│   │   │   └── NoShowPenaltyTrigger.java            # T2 + T3: llama a UC2 y marca el resultado
│   │   └── event/
│   │       └── NoShowAckListener.java               # consume el ack de M2 (informativo)
│   │
│   ├── domain/
│   │   ├── model/
│   │   │   ├── NoShowReport.java                    # reserva, estudiante, quien reporta, estado
│   │   │   ├── ReservationView.java                 # vista mínima de la reserva (de M2)
│   │   │   ├── ReporterRole.java                    # enum: PROFESOR, ADMIN
│   │   │   └── PenaltyTriggerStatus.java            # enum: PENDING, TRIGGERED, FAILED
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   └── ReportNoShowPort.java            # puerto de entrada
│   │   │   └── out/
│   │   │       ├── NoShowReportRepository.java
│   │   │       ├── ReservationLookupPort.java       # vista de la reserva (NC-01)
│   │   │       ├── NoShowReportedPublisherPort.java # inserta NoShowReported en outbox
│   │   │       └── CalculatePenaltyPort.java        # ya definido en UC2 (se usa, no se redefine)
│   │   └── error/
│   │       ├── ReservationNotFoundException.java
│   │       ├── StudentNotIdentifiedException.java
│   │       ├── CheckInAlreadyRegisteredException.java
│   │       ├── ReservationWindowActiveException.java
│   │       ├── DuplicateNoShowReportException.java
│   │       └── NoShowReportNotFoundException.java
│   │
│   └── infrastructure/
│       ├── persistence/
│       │   └── jpa/
│       │       ├── entity/
│       │       │   └── NoShowReportEntity.java
│       │       ├── repository/
│       │       │   └── JpaNoShowReportRepository.java
│       │       ├── adapter/
│       │       │   ├── NoShowReportRepositoryAdapter.java
│       │       │   └── ReservationLookupAdapter.java   # implementación inicial sobre tabla local (NC-01)
│       │       └── mapper/
│       │           └── NoShowReportMapper.java
│       ├── messaging/
│       │   ├── kafka/
│       │   │   ├── NoShowReportedPublisher.java     # arma el payload y lo inserta en outbox
│       │   │   ├── NoShowAckConsumer.java           # @KafkaListener del ack de M2
│       │   │   └── evento/
│       │   │       ├── NoShowReportedEvent.java
│       │   │       └── NoShowReportAcknowledgedEvent.java
│       │   └── outbox/                              # (existe, de UC6) — se reusa
│       └── config/
│           ├── SecurityConfig.java
│           └── UseCasesConfig.java
│
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       └── V11__no_show_report.sql
│
└── src/test/java/com/university/sanctions/
    ├── contract/
    │   ├── NoShowReportControllerTest.java
    │   └── AdminNoShowReportControllerTest.java
    ├── integration/
    │   ├── NoShowReportIT.java                      # persistencia, unicidad, T1/T2/T3
    │   ├── NoShowReportedPublishIT.java             # outbox → Kafka
    │   └── NoShowPenaltyRetryIT.java                # fallo de UC2 y reintento
    └── unit/
        ├── NoShowReportServiceTest.java
        ├── NoShowRegistrarTest.java                 # validaciones con Clock inyectado
        └── NoShowPenaltyTriggerTest.java

frontend/
└── src/
    ├── pages/
    │   └── AbsencePage.jsx                          # reporte + consulta (administración / profesor)
    ├── components/
    │   ├── NoShowReportForm.jsx                     # formulario de reporte
    │   ├── NoShowReportList.jsx                     # tabla de reportes
    │   ├── PenaltyTriggerBadge.jsx                  # PENDING / TRIGGERED / FAILED
    │   └── NoShowReviewPanel.jsx                    # reportes con needsReview + botón reintentar
    └── services/
        └── noShowApi.js                             # cliente HTTP
```

**Structure Decision**: se respeta la estructura de capas de `plan.md` (presentation / business / domain / infrastructure) con el paquete raíz `com.university.sanctions`. Siguiendo los planes los demás UC, los contratos de entrada y salida viven como puertos en `domain/port/in` y `domain/port/out`. La orquestación de los tres pasos vive en `business/service` repartida en tres colaboradores (`NoShowReportService`, `NoShowRegistrar`, `NoShowPenaltyTrigger`) para que cada transacción sea un bean independiente y `@Transactional` se aplique correctamente (una llamada interna dentro de la misma clase no pasaría por el proxy de Spring). La persistencia, el consumidor/productor Kafka y el outbox viven en `infrastructure`.

## Decisiones de diseño de este caso de uso

**Dónde queda cada FR.**

| FR | Qué pide | Dónde se implementa |
|---|---|---|
| FR-001 | Permitir a profesor/administrador reportar la inasistencia | `NoShowReportController.POST` + `NoShowReportService.report(...)`, rol `PROFESOR` o `ADMIN` |
| FR-002 | Identificar la reserva y el estudiante | `ReservationLookupPort.findById(...)` → `ReservationView.studentCode` |
| FR-003 | Verificar que no haya check-in | `NoShowRegistrar` → `CheckInAlreadyRegisteredException` si `ReservationView.checkedIn = true` |
| FR-004 | Verificar que la ventana ya finalizó | `NoShowRegistrar` compara `windowEnd` con `Clock.now()` → `ReservationWindowActiveException` |
| FR-005 | Registrar fecha, reserva, estudiante y quien reporta | `NoShowReportRepository.save(...)` en T1 |
| FR-006 | Disparar obligatoriamente Calcular penalización | `NoShowPenaltyTrigger` llama a `CalculatePenaltyPort.calculate(...)` (T2) |
| FR-007 | Impedir reporte duplicado | Clave única en `no_show_report.reservation_id` + verificación previa |
| FR-008 | Informar reserva no identificable / con check-in / en ventana vigente | Excepciones de dominio → `ProblemDetail` (ver Contratos §5) |

**Decisiones justificadas.**

1. **Registrar y penalizar NO comparten transacción.** Es la decisión central y es deliberadamente distinta de UC2/UC1. La spec dice que, si falla el cálculo, "el reporte quedó registrado pero la penalización no pudo iniciarse, marcando la inconsistencia para revisión". Con una sola transacción, un fallo en UC2 revertiría el reporte y esa frase sería imposible de cumplir. Con tres pasos, el reporte sobrevive y queda `FAILED` + `needs_review`.

2. **UC7 sí exige que UC2 se dispare (`«include»`), pero tolera que falle.** SC-001 pide que el 100 % de los reportes válidos *disparen* el cálculo. "Disparar" se mide como *intentar la llamada*; el reintento (US3) cubre el caso de fallo transitorio y deja visible el caso de fallo permanente (por ejemplo, UC2 responde `NoActiveRuleException` porque falta configurar la regla `NO_SHOW`).

3. **Idempotencia del disparo con `sourceEventId = reportId`.** Se envía a UC2 como `PenaltyRequest.sourceEventId`; UC2 tiene clave única en `penalty_history.source_event_id`. Si T2 tuvo éxito pero T3 falló (el proceso murió antes de marcar `TRIGGERED`), el reintento recibe `DuplicatePenaltyException`. UC7 lo trata como **éxito**, recupera el `penaltyId` existente y marca `TRIGGERED`. Esto requiere que UC2 permita buscar una penalización por `sourceEventId` (ver Dependencias).

4. **Validación de duplicado en dos niveles.** Una verificación previa (`existsByReservationId`) da el mensaje claro al usuario; la clave única `UNIQUE (reservation_id)` es la garantía real ante concurrencia (dos profesores reportando a la vez). Si el `INSERT` choca con la clave, se traduce a `DuplicateNoShowReportException` (SC-002).

5. **La vista de la reserva se aísla detrás de `ReservationLookupPort`.** La spec dice que M2 es el dueño de las reservas pero no define cómo M3 se entera de una reserva sin check-in (NC-01). El puerto devuelve `ReservationView(reservationId, studentCode, resourceId, windowStart, windowEnd, checkedIn, cancelled)`. La implementación inicial lee la tabla local; si se decide REST a M2 o un evento nuevo, solo cambia el adaptador.

6. **Una reserva cancelada no se puede reportar.** La spec no lo menciona, pero reportar inasistencia sobre una reserva cancelada sería sancionar a alguien por algo que no ocurrió. Se trata como `RESERVATION_NOT_FOUND` con detalle propio (NC-04 para confirmarlo).

7. **`occurredAt` de la infracción = fin de la ventana de la reserva**, no el momento del reporte. La infracción ocurrió cuando la ventana terminó sin check-in; esa fecha alimenta el conteo de infracciones por período de UC2. La fecha del reporte se guarda aparte en `reported_at`.

8. **La confirmación a M2 sale por outbox dentro de T1.** Se inserta en `outbox_message` en la misma transacción que el reporte; el `OutboxPublisher` (reusado de UC6) la lleva a Kafka. Si Kafka o M2 están caídos, el reporte y la penalización no se afectan (spec: "no debe bloquearse esperando esa entrega").

9. **El reportante sale del JWT, no del cuerpo.** `reported_by` y `reported_by_role` se toman del token; el cliente no puede suplantar a otro.

10. **Un reporte nunca se elimina ni se edita.** Es evidencia de un incumplimiento. Solo cambian los campos de seguimiento (`penalty_status`, `penalty_id`, `needs_review`, `m2_acknowledged_at`).

11. **El reintento es manual (administrador), sin job automático.** Un fallo de UC2 por falta de regla/escala no se arregla reintentando solo; requiere que alguien configure y luego reintente. Si en la práctica hay fallos transitorios frecuentes, se puede añadir un `@Scheduled` después sin cambiar el contrato.

## Contratos

Se aplican las convenciones comunes de M2 (camelCase, MAYUSCULA_CON_GUION_BAJO, nunca null, ISO-8601 con `-05:00`, RFC 9457). M3 las adopta para sus propios contratos.

### 1. Puerto de entrada — `ReportNoShowPort`

```java
public interface ReportNoShowPort {

    /**
     * Valida, registra el reporte y dispara Calcular penalización.
     * @throws ReservationNotFoundException       reserva no identificada, o cancelada
     * @throws StudentNotIdentifiedException      la reserva no permite asociar un estudiante válido
     * @throws CheckInAlreadyRegisteredException la reserva ya tiene check-in
     * @throws ReservationWindowActiveException   la ventana de la reserva no ha finalizado
     * @throws DuplicateNoShowReportException     ya existe un reporte para la reserva
     */
    NoShowReportResult report(ReportNoShowCommand command);

    /** Reintenta el disparo de la penalización de un reporte en FAILED. */
    NoShowReportResult retryPenalty(String reportId, String adminCode);
}

public record ReportNoShowCommand(
    String reservationId,
    String reporterCode,      // sale del JWT
    ReporterRole reporterRole // PROFESOR o ADMIN, sale del JWT
) {}

public record NoShowReportResult(
    String reportId,
    String reservationId,
    String studentCode,
    PenaltyTriggerStatus penaltyStatus, // TRIGGERED o FAILED
    String penaltyId,                   // nulo si FAILED
    boolean needsReview,
    Instant reportedAt
) {}
```

### 2. Puerto de salida — `ReservationLookupPort`

```java
public interface ReservationLookupPort {
    Optional<ReservationView> findById(String reservationId);
}

public record ReservationView(
    String reservationId,
    String studentCode,       // nulo si no se puede asociar
    String resourceId,
    Instant windowStart,
    Instant windowEnd,
    boolean checkedIn,
    boolean cancelled
) {}
```

### 3. Puerto de salida — `CalculatePenaltyPort` (ya definido en UC2)

UC7 lo usa tal cual. La solicitud que construye:

```java
new PenaltyRequest(
    view.studentCode(),
    view.resourceId(),
    PenaltyType.NO_SHOW,
    report.id(),              // sourceEventId  → idempotencia en UC2
    "NO_SHOW_REPORT",         // sourceEventType
    view.windowEnd()          // occurredAt
);
```

### 4. Evento a M2 — `module3.sanctions.no-show-reported.v1`

Evento: `NoShowReported`. Cuándo se emite: al registrar el reporte (T1, vía outbox). El payload sigue el envoltorio de M2.

```json
{
  "eventId": "nsr-4b6e82c1-84ef-4b2a-9e01-2c7d8b5f3a64",
  "type": "NoShowReported",
  "version": 1,
  "occurredAt": "2026-10-05T11:15:00-05:00",
  "source": "MODULO_3",
  "data": {
    "reportId": "7d1c3e52-9a04-4f67-b1c8-3e5a2d9f6b10",
    "reservationId": "res-20261005-0042",
    "studentCode": "2023123456",
    "resourceId": "ACT-004512",
    "reportedBy": "profesor@unimagdalena.edu.co",
    "reportedByRole": "PROFESOR",
    "windowEnd": "2026-10-05T10:00:00-05:00",
    "reportedAt": "2026-10-05T11:15:00-05:00"
  }
}
```

| Campo | Tipo | Nota |
|---|---|---|
| `reportId` | string | Id del reporte en M3 |
| `reservationId` | string | El mismo id que usa M2 |
| `reportedByRole` | enum | `PROFESOR`, `ADMIN` |
| `windowEnd` | instante | Fin de la ventana que se incumplió |

**Clave de partición:** `studentCode`.

### 5. Endpoints REST

#### `POST /api/v1/no-show-reports`

Rol `PROFESOR` o `ADMIN`. El reportante sale del JWT.

```json
{
  "reservationId": "res-20261005-0042"
}
```

Respuesta `201 Created` (penalización disparada):

```json
{
  "id": "7d1c3e52-9a04-4f67-b1c8-3e5a2d9f6b10",
  "reservationId": "res-20261005-0042",
  "studentCode": "2023123456",
  "resourceId": "ACT-004512",
  "reportedBy": "profesor@unimagdalena.edu.co",
  "reportedByRole": "PROFESOR",
  "reportedAt": "2026-10-05T11:15:00-05:00",
  "penaltyStatus": "TRIGGERED",
  "penaltyId": "pen-88391",
  "needsReview": false,
  "message": "Inasistencia registrada. Se aplicó la penalización correspondiente."
}
```

Respuesta `201 Created` (reporte registrado, penalización fallida — Edge Case de la spec):

```json
{
  "id": "7d1c3e52-9a04-4f67-b1c8-3e5a2d9f6b10",
  "reservationId": "res-20261005-0042",
  "studentCode": "2023123456",
  "resourceId": "ACT-004512",
  "reportedBy": "profesor@unimagdalena.edu.co",
  "reportedByRole": "PROFESOR",
  "reportedAt": "2026-10-05T11:15:00-05:00",
  "penaltyStatus": "FAILED",
  "needsReview": true,
  "message": "La inasistencia quedó registrada, pero la penalización no pudo iniciarse. Quedó marcada para revisión."
}
```

> Un fallo de penalización **no es un error HTTP**: el reporte sí se creó, por eso `201` con `penaltyStatus: FAILED`.

#### `GET /api/v1/no-show-reports/{id}`

Devuelve el mismo objeto de arriba. Un `PROFESOR` solo puede ver los reportes que él hizo (`403` si no).

#### `GET /api/v1/no-show-reports?needsReview=true&page=1&pageSize=20`

```json
{
  "reports": [ { "id": "7d1c3e52-...", "reservationId": "res-20261005-0042", "studentCode": "2023123456", "penaltyStatus": "FAILED", "needsReview": true, "reportedAt": "2026-10-05T11:15:00-05:00" } ],
  "pagination": { "page": 1, "pageSize": 20, "totalPages": 1, "total": 1 }
}
```

`PROFESOR` ve solo los suyos; `ADMIN` y `DIRECCION_PROGRAMA` ven todos y pueden filtrar por `needsReview`.

#### `POST /api/v1/admin/no-show-reports/{id}/retry-penalty`

Rol `ADMIN`. Sin cuerpo. Respuesta `200 OK` con el reporte actualizado (`TRIGGERED` o, si vuelve a fallar, `FAILED` con `needsReview: true`). `409 PENALTY_ALREADY_TRIGGERED` si el reporte ya estaba `TRIGGERED`.

### 6. Errores

| Código HTTP | `code` | Cuándo |
|---|---|---|
| 400 | `INVALID_REQUEST` | Falta `reservationId` |
| 404 | `RESERVATION_NOT_FOUND` | No fue posible identificar la reserva (o está cancelada) (FR-008) |
| 404 | `NO_SHOW_REPORT_NOT_FOUND` | El reporte consultado no existe |
| 409 | `CHECK_IN_ALREADY_REGISTERED` | La reserva ya cuenta con check-in (FR-003, FR-008) |
| 409 | `RESERVATION_WINDOW_ACTIVE` | La ventana de la reserva no ha finalizado (FR-004, FR-008) |
| 409 | `DUPLICATE_NO_SHOW_REPORT` | Ya existe un reporte para la reserva (FR-007) |
| 409 | `PENALTY_ALREADY_TRIGGERED` | Reintento sobre un reporte ya penalizado |
| 422 | `STUDENT_NOT_IDENTIFIED` | No fue posible asociar la inasistencia a un estudiante válido |
| 500 | `PERSISTENCE_ERROR` | Fallo al registrar; no queda registro incompleto |

`ProblemDetail` de ejemplo:

```json
{
  "type": "https://sanctions.unimagdalena.edu.co/errors/check-in-already-registered",
  "title": "La reserva ya tiene check-in",
  "status": 409,
  "detail": "La reserva res-20261005-0042 ya cuenta con check-in registrado; no se puede reportar inasistencia.",
  "instance": "/api/v1/no-show-reports",
  "code": "CHECK_IN_ALREADY_REGISTERED",
  "reservationId": "res-20261005-0042"
}
```

### 7. Tablas

#### `no_show_report`

| Columna | Tipo | Nota |
|---|---|---|
| `id` | varchar(36) PK | UUID |
| `reservation_id` | varchar(50) NOT NULL | **UNIQUE** (FR-007) |
| `student_code` | varchar(20) NOT NULL | |
| `resource_id` | varchar(50) NOT NULL | |
| `window_end` | timestamp NOT NULL | Fin de la ventana incumplida |
| `reported_by` | varchar(100) NOT NULL | Del JWT |
| `reported_by_role` | varchar(20) NOT NULL | `PROFESOR`, `ADMIN` |
| `reported_at` | timestamp NOT NULL | |
| `penalty_status` | varchar(20) NOT NULL | `PENDING`, `TRIGGERED`, `FAILED` |
| `penalty_id` | varchar(36), nulo | Id en `penalty_history` (UC2) |
| `needs_review` | boolean NOT NULL DEFAULT false | |
| `failure_reason` | varchar(255), nulo | Motivo del último fallo de UC2 |
| `penalty_attempts` | int NOT NULL DEFAULT 0 | |
| `m2_acknowledged_at` | timestamp, nulo | Ack informativo de M2 |
| `created_at` | timestamp NOT NULL | |
| `updated_at` | timestamp NOT NULL | |

```sql
CREATE UNIQUE INDEX no_show_report_reservation ON no_show_report (reservation_id);
CREATE INDEX no_show_report_student ON no_show_report (student_code, reported_at DESC);
CREATE INDEX no_show_report_review ON no_show_report (needs_review, reported_at DESC);
```

La clave única en `reservation_id` es lo que hace cumplir FR-007 y SC-002 a nivel de base de datos. `outbox_message` e `inbox_message` ya existen (UC6); UC7 los reusa.

### 8. Tipos del frontend

```js
export const PenaltyTriggerStatus = {
  PENDING: "PENDING",
  TRIGGERED: "TRIGGERED",
  FAILED: "FAILED",
};

export const reportNoShow = async (reservationId) => {
  const response = await fetch("/api/v1/no-show-reports", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ reservationId }),
  });
  if (!response.ok) throw await response.json();
  return response.json();
};

export const getNoShowReports = async ({ needsReview, page = 1 } = {}) => {
  const params = new URLSearchParams({ page });
  if (needsReview !== undefined) params.set("needsReview", needsReview);
  const response = await fetch(`/api/v1/no-show-reports?${params}`);
  if (!response.ok) throw await response.json();
  return response.json();
};

export const retryNoShowPenalty = async (reportId) => {
  const response = await fetch(`/api/v1/admin/no-show-reports/${reportId}/retry-penalty`, {
    method: "POST",
  });
  if (!response.ok) throw await response.json();
  return response.json();
};
```

### 9. Fixtures compartidos

```text
backend/src/test/resources/contracts/
├── api-no-show-report-201.json
├── api-no-show-report-201-penalty-failed.json
├── api-no-show-reports-list-200.json
├── api-no-show-report-404-reservation.json
├── api-no-show-report-409-check-in.json
├── api-no-show-report-409-window-active.json
├── api-no-show-report-409-duplicate.json
├── api-no-show-report-422-student.json
└── event-no-show-reported.json
```

## Phase 1: Setup

- [ ] T001 Añadir a `application.yml` el topic `sanctions.kafka.topic.no-show-reported=module3.sanctions.no-show-reported.v1` y el topic de ack `sanctions.kafka.topic.no-show-ack=module2.reservation.no-show-ack.v1`
- [ ] T002 [P] Extender `SanctionsProperties` con esos parámetros y validarlos al arrancar
- [ ] T003 [P] Registrar un bean `Clock` (zona `America/Bogota`) si no existe ya de UC1, para poder inyectarlo en las validaciones de ventana

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: dónde guardar el reporte y cómo consultar la reserva.

- [ ] T004 Escribir `V11__no_show_report.sql` con la tabla `no_show_report` y sus índices de Contratos §7 (incluida la clave única en `reservation_id`)
- [ ] T005 [P] Crear el modelo de dominio: `NoShowReport`, `ReservationView`, `ReporterRole`, `PenaltyTriggerStatus` en `domain/model/`
- [ ] T006 [P] Definir `ReportNoShowPort` en `domain/port/in/` y `NoShowReportRepository`, `ReservationLookupPort`, `NoShowReportedPublisherPort` en `domain/port/out/`
- [ ] T007 [P] Verificar que `CalculatePenaltyPort` (UC2) y las clases `PenaltyRequest`/`PenaltyResult` están disponibles; si UC2 aún no se implementó, dejar un doble de prueba
- [ ] T008 [P] Crear las excepciones de `domain/error/` listadas en Project Structure
- [ ] T009 [P] Implementar `NoShowReportEntity`, `JpaNoShowReportRepository` y `NoShowReportRepositoryAdapter` con su mapper
- [ ] T010 [P] Implementar `ReservationLookupAdapter` (versión inicial sobre la tabla local; **depende de NC-01**)
- [ ] T011 [P] Implementar `NoShowReportedPublisher` (arma el payload y lo inserta en `outbox_message`) y el record `NoShowReportedEvent`
- [ ] T012 Verificar que `outbox_message` e `inbox_message` (de UC6) existen y son accesibles

**Checkpoint**: existe dónde guardar el reporte, cómo leer la reserva y cómo encolar el evento a M2

## Phase 3: User Story 1 — Reporte exitoso de inasistencia (Priority: P1)

**Goal**: un profesor o administrador reporta la inasistencia de una reserva vencida sin check-in; el sistema registra el reporte, encola la confirmación a M2 y dispara obligatoriamente Calcular penalización.

**Independent Test**: crear una reserva sin check-in con ventana vencida, llamar a `POST /api/v1/no-show-reports` y comprobar que (a) hay una fila en `no_show_report` con `penalty_status = TRIGGERED`, (b) hay una fila en `outbox_message` con `NoShowReported`, (c) `CalculatePenaltyPort` recibió `NO_SHOW` con `sourceEventId = reportId`.

### Tests for User Story 1

- [ ] T013 [P] [US1] Pruebas en `NoShowReportServiceTest.java` (unit, con puertos falsos): reporte válido → registra, encola y llama a UC2 con el `PenaltyRequest` esperado (`occurredAt = windowEnd`); el reportante y su rol salen del comando
- [ ] T014 [P] [US1] Prueba `NoShowReportIT.java` con Testcontainers MySQL: T1 es atómica (reporte + outbox en la misma transacción; fallo a mitad → rollback completo y sin registro incompleto); T3 deja `TRIGGERED` con `penalty_id`
- [ ] T015 [P] [US1] Prueba `NoShowReportedPublishIT.java` con Testcontainers Kafka: el `OutboxPublisher` publica `NoShowReported` con clave `studentCode` y el envoltorio de M2
- [ ] T016 [P] [US1] Prueba `NoShowReportControllerTest.java` con `@WebMvcTest`, contra los fixtures: `POST` válido → 201; sin sesión → 401; rol `ESTUDIANTE` → 403

### Implementation for User Story 1

- [ ] T017 [US1] Implementar `NoShowRegistrar.register(...)` (`@Transactional`): buscar reserva, guardar `NoShowReport` con `PENDING` e insertar en outbox (T1) — *las validaciones se completan en US2*
- [ ] T018 [US1] Implementar `NoShowPenaltyTrigger.trigger(...)`: construir `PenaltyRequest`, llamar a `CalculatePenaltyPort.calculate(...)` y marcar `TRIGGERED` con `penaltyId` (T2 + T3)
- [ ] T019 [US1] Implementar `NoShowReportService.report(...)` que orquesta T1 → T2 → T3 (sin `@Transactional` propio)
- [ ] T020 [US1] Implementar `NoShowReportController` con `POST /api/v1/no-show-reports` y los DTOs `NoShowReportRequest`/`NoShowReportResponse`; tomar reportante y rol del JWT

**Checkpoint**: el camino feliz funciona de punta a punta, incluido el disparo a UC2

## Phase 4: User Story 2 — Rechazo de reportes inválidos (Priority: P1)

**Goal**: el sistema rechaza con un mensaje claro los reportes sobre reservas con check-in, con ventana vigente, duplicados o no identificables.

**Independent Test**: para cada escenario de la spec (check-in realizado, reporte duplicado) y cada Edge Case (reserva no encontrada, ventana vigente, estudiante no identificado), comprobar el código HTTP, el `code` y que **no** se creó fila ni evento.

### Tests for User Story 2

- [ ] T021 [P] [US2] Pruebas en `NoShowRegistrarTest.java` con `Clock` inyectado: reserva inexistente; reserva cancelada; sin estudiante; con check-in; ventana vigente (`windowEnd` = ahora + 1 min); ventana recién vencida (`windowEnd` = ahora - 1 s, debe aceptar); duplicado
- [ ] T022 [P] [US2] Prueba `NoShowReportIT.java` (extensión): dos `POST` concurrentes sobre la misma reserva → una fila, un solo `201`, el otro `409 DUPLICATE_NO_SHOW_REPORT` (clave única); un rechazo no deja filas en `no_show_report` ni en `outbox_message`
- [ ] T023 [P] [US2] Pruebas en `NoShowReportControllerTest.java`: `404`, `409` (×3) y `422` contra los fixtures de Contratos §9; cada uno devuelve `ProblemDetail` con el `code` esperado

### Implementation for User Story 2

- [ ] T024 [US2] Completar `NoShowRegistrar` con las validaciones en este orden: reserva identificable → no cancelada → estudiante identificable → no duplicado → sin check-in → ventana finalizada
- [ ] T025 [US2] Traducir la violación de la clave única `reservation_id` a `DuplicateNoShowReportException` en `NoShowReportRepositoryAdapter`
- [ ] T026 [US2] Mapear cada excepción de dominio a su `ProblemDetail` (`@RestControllerAdvice`) según Contratos §6

**Checkpoint**: ningún reporte inválido se registra, y cada rechazo informa el motivo exacto

## Phase 5: User Story 3 — Inconsistencia y reintento de la penalización (Priority: P2)

**Goal**: si Calcular penalización falla, el reporte queda registrado y marcado para revisión; un administrador puede reintentar sin duplicar la sanción.

**Independent Test**: simular que `CalculatePenaltyPort` lanza `NoActiveRuleException`; comprobar `201` con `penaltyStatus = FAILED` y fila en `needs_review`; corregir el doble para que funcione, llamar al reintento y comprobar `TRIGGERED`.

### Tests for User Story 3

- [ ] T027 [P] [US3] Pruebas en `NoShowPenaltyTriggerTest.java`: UC2 lanza excepción → `FAILED`, `needs_review = true`, `failure_reason` y `penalty_attempts` incrementado; **el reporte y el evento a M2 se conservan**; UC2 lanza `DuplicatePenaltyException` → se trata como éxito y se recupera el `penaltyId`
- [ ] T028 [P] [US3] Prueba `NoShowPenaltyRetryIT.java`: reintento tras un fallo → `TRIGGERED` y `needs_review = false`; reintento sobre un reporte ya `TRIGGERED` → `409`; un reintento no genera una segunda penalización en `penalty_history`
- [ ] T029 [P] [US3] Prueba `AdminNoShowReportControllerTest.java`: `POST .../retry-penalty` con `ADMIN` → 200; con `PROFESOR` → 403; reporte inexistente → 404

### Implementation for User Story 3

- [ ] T030 [US3] Completar `NoShowPenaltyTrigger` con la rama de fallo (`markPenaltyFailed`) y el tratamiento de `DuplicatePenaltyException`
- [ ] T031 [US3] Implementar `NoShowReportService.retryPenalty(...)` y `AdminNoShowReportController`
- [ ] T032 [US3] Añadir a UC2 (`PenaltyHistoryRepository`) la búsqueda por `sourceEventId` necesaria para recuperar el `penaltyId` (coordinar con el plan de UC2)

**Checkpoint**: un fallo de la penalización no pierde el reporte y es recuperable

## Phase 6: User Story 4 — Consulta y pantalla de reporte (Priority: P3)

**Goal**: el profesor/administrador reporta desde la interfaz y consulta sus reportes; la administración ve y reintenta los que necesitan revisión. Además, M2 puede confirmar la recepción.

### Tests for User Story 4

- [ ] T033 [P] [US4] Prueba `NoShowReportControllerTest.java` (extensión): `GET /{id}` con el dueño → 200, con otro profesor → 403; `GET` lista filtrada por `needsReview`
- [ ] T034 [P] [US4] Prueba de `NoShowAckListener`: el ack de M2 actualiza `m2_acknowledged_at`; un ack duplicado se descarta vía `inbox_message`; un ack de un reporte inexistente se registra en log y no falla
- [ ] T035 [P] [US4] Pruebas de `NoShowReportForm` y `NoShowReviewPanel` (React Testing Library): errores del backend se muestran con su mensaje; el botón "Reintentar" solo aparece con `FAILED`

### Implementation for User Story 4

- [ ] T036 [US4] Implementar `GET /api/v1/no-show-reports/{id}` y `GET /api/v1/no-show-reports` con paginación y las reglas de visibilidad por rol
- [ ] T037 [P] [US4] Implementar `NoShowAckConsumer` y `NoShowAckListener` (**opcional hasta resolver NC-03**)
- [ ] T038 [P] [US4] Frontend: `noShowApi.js`
- [ ] T039 [P] [US4] Frontend: `NoShowReportForm.jsx` y `PenaltyTriggerBadge.jsx`
- [ ] T040 [P] [US4] Frontend: `NoShowReportList.jsx` y `NoShowReviewPanel.jsx`
- [ ] T041 [US4] Frontend: `AbsencePage.jsx` que integra el formulario, la lista y el panel de revisión (según rol)

**Checkpoint**: el flujo es utilizable de punta a punta desde la interfaz

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T042 [P] Verificar SC-001 (100 % de los reportes válidos disparan Calcular penalización)
- [ ] T043 [P] Verificar SC-002 (0 % de reportes duplicados, incluso con concurrencia)
- [ ] T044 [P] Verificar SC-003 (100 % de los intentos sobre reservas con check-in son rechazados)
- [ ] T045 [P] Verificar SC-004 (registro + disparo de la penalización en < 2 s)
- [ ] T046 [P] Registrar en logs cada reporte (reservationId, estudiante, reportante, resultado de la penalización) y cada fallo de UC2 con su causa
- [ ] T047 [P] Prueba ArchUnit: `domain` y `business` no importan JPA, Kafka ni clases de `infrastructure`
- [ ] T048 [P] Documentar en el README cómo reportar una inasistencia y cómo reintentar una penalización fallida
- [ ] T049 Llevar a `pendientes-clarificacion.md` los NEEDS CLARIFICATION abiertos en este plan (NC-01 a NC-05)

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: depende de `plan.md`
- **Foundational (Phase 2)**: depende de Setup — BLOCKS las user stories
- **User Story 1 (Phase 3)**: depende de Foundational
- **User Story 2 (Phase 4)**: depende de US1 (completa el mismo `NoShowRegistrar`)
- **User Story 3 (Phase 5)**: depende de US1
- **User Story 4 (Phase 6)**: depende de US1 (la lista y la pantalla necesitan reportes); el ack (T037) es independiente
- **Polish (Phase 7)**: depende de todas las US

### Dependencias con otros casos de uso del M3

- **Calcular penalización (UC2)**: UC7 lo dispara con `CalculatePenaltyPort.calculate(...)`. Requiere que exista una regla activa `NO_SHOW` y una escala configurada (el seed `V8` de UC2 las crea). Requiere además la búsqueda de penalización por `sourceEventId` (T032).
- **Realizar check-out (UC6)**: aporta la tabla local de `usage` y el `outbox_message`/`inbox_message`/`OutboxPublisher` que UC7 reusa. Define cómo se refleja el check-in (NC-01).
- **Notificar sanción (UC5)**: no lo llama UC7; lo dispara UC2 tras calcular la penalización.

### Dependencias con otros módulos

- **Módulo 2**: consume `module3.sanctions.no-show-reported.v1` (creado por M3). Produce `module2.reservation.no-show-ack.v1` (opcional, NC-03). Es el dueño de la reserva (NC-01).
- **Módulo 1**: sin integración.

### Parallel Opportunities

- En Foundational: T005 a T011
- En US1: T013 a T016 (tests)
- En US2: T021, T022, T023 (tests)
- En US3: T027, T028, T029 (tests) — paralelizable con US2
- En US4: T033, T034, T035 (tests) y T038 a T040 (frontend)
- En Polish: T042 a T048

## NEEDS CLARIFICATION abiertos

| Id | Pendiente | Supuesto con el que avanza el plan |
|---|---|---|
| NC-01 | **De dónde sale la información de una reserva sin check-in.** UC6 crea la `Usage` local a partir de `ReservationRecordCreated`, que parece representar el registro del uso (o sea, ya con check-in). Una reserva a la que nadie se presentó podría no existir nunca en M3. Opciones: (a) M2 emite un evento al crear la reserva y M3 guarda una vista local; (b) consulta REST síncrona a M2 al reportar; (c) el reporte lleva un snapshot de la reserva | Se usa `ReservationLookupPort`; el adaptador inicial lee una tabla local. Cambiar de opción solo cambia `ReservationLookupAdapter`. **Bloquea T010.** Si se elige (b), hay que añadir ese REST a la tabla de comunicación de `plan.md` |
| NC-02 | **Si un profesor puede reportar a cualquier estudiante o solo a los de sus cursos/reservas** | Cualquier `PROFESOR` o `ADMIN` autenticado puede reportar cualquier reserva |
| NC-03 | **Si M2 realmente produce `NoShowReportAcknowledged` y con qué payload.** El contrato figura en los planes de M3, pero la spec dice que M2 *consume* la confirmación | El consumo del ack es informativo y opcional (T037); el flujo no depende de él |
| NC-04 | **Qué hacer con una reserva cancelada** (la spec no la menciona) | Se rechaza con `RESERVATION_NOT_FOUND` |
| NC-05 | **Si hay un período de gracia** tras el fin de la ventana antes de poder reportar | No hay gracia: se puede reportar en cuanto `windowEnd < ahora` |

## Notes

- La numeración T0XX es propia de este plan
- [P] tasks = different files, no dependencies
- [US1] a [US4] = trazabilidad a la user story
- La sección Contratos es la única fuente del JSON de UC7
- Decisiones adoptadas de M2: convenciones (camelCase, MAYUSCULA_CON_GUION_BAJO, nunca null, ISO-8601 con -05:00, RFC 9457), nombres de tablas en snake_case
- Decisiones propias de M3: registrar y penalizar en transacciones separadas; el fallo de UC2 no revierte el reporte; idempotencia con `sourceEventId = reportId`; `occurredAt` = fin de la ventana; confirmación a M2 por outbox; reintento manual por administración
- Lo que M3 consume de M2: se adopta tal cual (no se redefine)
- Lo que M3 produce para M2: M3 lo crea y lo documenta. M2 lo consume cuando esté listo.
