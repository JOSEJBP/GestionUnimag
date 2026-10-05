# Implementation Plan: Realizar check-out (UC0)

**Date**: 2026-10-05
**Spec**: [realizar-checkout.md](../Especs/realizar-checkout.md)
**Plan general**: [plan.md](../Especs/plan.md)
**Módulo**: Módulo 3 — Sanciones y Cumplimiento
**Planes previos**: ninguno
**Planes relacionados**: [plan-uc1-actualizar-score.md](./plan-uc1-actualizar-score.md) — dispara aumento cuando la entrega es `ON_TIME`; [plan-uc2-calcular-penalizacion.md](./plan-uc2-calcular-penalizacion.md) — dispara penalización cuando la entrega es `LATE`; [plan-uc-reportar-novedad-tecnica.md](./plan-uc-reportar-novedad-tecnica.md) — dispara el reporte de daño durante el check-out

## Summary

UC0 es la operación de cierre de la utilización del recurso. Es el punto de entrada del ciclo del préstamo: finaliza la relación estudiante-recurso, registra la fecha de devolución, y decide la continuación del flujo según el resultado de la entrega.

El caso de uso no define la sanción ni el score por sí mismo; solo validad, finaliza y dispara los flujos que corresponden:

- si la entrega fue a tiempo, dispara `Actualizar score de confianza`
- si fue tarde, dispara `Calcular penalización`
- si hubo daño, dispara `Reportar novedad técnica`

**Enfoque técnico:**

1. El check-out se ejecuta sobre una utilización activa del estudiante.
2. Se valida que el recurso exista y que la utilización esté en estado válido.
3. Se registra la fecha de devolución y se finaliza la utilización.
4. El resultado del check-out condiciona qué flujo interno se dispara.
5. La operación debe ser idempotente frente a solicitudes duplicadas para la misma utilización.

## Technical Context

**Language/Version**: Java 21; JavaScript/React + Vite para la UI.
**Primary Dependencies**: Spring Boot 4.x, Spring Web, Spring Data JPA, Validation, Security, Flyway, Kafka para notificaciones asíncronas a M2.
**Storage**: MySQL 8. Lee la tabla de utilizaciones y recursos; registra el check-out y finaliza la utilización.
**Testing**: JUnit 5 + Mockito, Testcontainers MySQL, `@WebMvcTest` para controladores y ArchUnit.
**Target Platform**: Linux server con JVM 21; frontend web para estudiante.
**Project Type**: Web: backend + frontend separados.
**Performance Goals**: el check-out se procesa en menos de 1 segundo para el caso feliz.
**Constraints**:
- solo puede existir un check-out válido por utilización
- la utilización debe estar activa al momento del cierre
- el reporte de daño no bloquea la finalización del check-out
- la zona horaria es `America/Bogota`

## Integración con otros módulos

### Lo que M3 consume de M2

M3 consume `CheckOutAcknowledged` desde `module2.reservation.check-out-ack.v1`, contrato definido en [Módulo 2 UC12, Contratos §2](../plan-uc12-recibir-check-out.md). El acuse permite correlacionar el check-out local con el cierre del préstamo que hizo M2.

### Lo que M3 produce para M2

M3 envía el check-out por el topic que consume UC12 de M2. El acuse de M2 vuelve por su topic dedicado. Esta es la integración de check-out entre módulos; el daño no se incluye en este evento y se reporta a M1 por UC8.

| Dirección | Evento | Topic | Uso |
|---|---|---|---|
| M3 → M2 | `CheckOutRegistered` | `module3.reservation.check-out.v1` | M2 recibe la devolución y cierra el préstamo, o registra la revisión del espacio |
| M2 → M3 | `CheckOutAcknowledged` | `module2.reservation.check-out-ack.v1` | M3 registra el resultado aceptado o rechazado |

### Endpoints que M3 expone para M2

**No expone endpoints REST a M2.** La integración se realiza por los topics anteriores.

### Endpoints que M3 expone para frontend

| Endpoint | Método | Para qué |
|---|---|---|
| `POST /api/v1/checkouts` | POST | registrar cierre de utilización |
| `GET /api/v1/checkouts/{id}` | GET | consultar un check-out |
| `GET /api/v1/checkouts/me` | GET | consultar los check-outs del estudiante |

## Project Structure

```text
backend/
├── src/main/java/com/university/sanctions/
│   ├── presentation/
│   │   ├── controller/
│   │   │   ├── CheckOutController.java
│   │   │   └── AdminCheckOutController.java
│   │   └── dto/
│   │       ├── CheckOutRequest.java
│   │       ├── CheckOutResponse.java
│   │       └── CheckOutSummaryResponse.java
│   ├── business/
│   │   ├── service/
│   │   │   └── CheckOutService.java
│   │   └── event/
│   │       ├── CheckOutRegisteredEvent.java
│   │       └── CheckOutAcknowledgedEvent.java
│   ├── domain/
│   │   ├── model/
│   │   │   ├── CheckOut.java
│   │   │   ├── Usage.java                         # incluye reservationId de M2 y fecha límite
│   │   │   ├── ResourceView.java
│   │   │   ├── CheckOutAcknowledgement.java
│   │   │   └── CheckoutStatus.java
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   ├── RegisterCheckOutPort.java
│   │   │   │   └── RecordCheckOutAcknowledgementPort.java
│   │   │   └── out/
│   │   │       ├── UsageRepository.java
│   │   │       ├── ResourceLookupPort.java
│   │   │       ├── CheckOutRegisteredPublisherPort.java
│   │   │       ├── UpdateTrustScorePort.java
│   │   │       ├── CalculatePenaltyPort.java
│   │   │       └── ReportDamagePort.java
│   │   └── error/
│   │       ├── UsageNotFoundException.java
│   │       ├── UsageAlreadyClosedException.java
│   │       └── ResourceNotIdentifiedException.java
│   │
│   └── infrastructure/
│       ├── messaging/
│       │   ├── CheckOutRegisteredPublisher.java
│       │   ├── CheckOutAcknowledgementListener.java
│       │   └── OutboxPublisher.java
│       ├── persistence/
│       │   └── jpa/
│       │       ├── entity/
│       │       │   ├── CheckOutEntity.java
│       │       │   └── OutboxMessageEntity.java
│       │       ├── repository/
│       │       │   └── JpaCheckOutRepository.java
│       │       ├── adapter/
│       │       │   └── CheckOutRepositoryAdapter.java
│       │       └── mapper/
│       │           └── CheckOutMapper.java
│       └── config/
│           ├── SecurityConfig.java
│           └── UseCasesConfig.java
│
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       └── V3__check_out.sql
│
├── src/test/java/com/university/sanctions/
    ├── contract/
    │   └── CheckOutControllerTest.java
    ├── integration/
    │   ├── CheckOutIT.java
    │   └── CheckOutAcknowledgementIT.java
    └── unit/
        ├── CheckOutServiceTest.java
        └── UsageStateTest.java
└── src/test/resources/contracts/
    ├── event-check-out-registered-asset.json
    ├── event-check-out-registered-space.json
    ├── event-check-out-ack-asset.json
    ├── event-check-out-ack-space.json
    └── event-check-out-ack-rejected.json

frontend/
└── src/
    ├── pages/
    │   └── CheckOutPage.jsx
    ├── components/
    │   ├── CheckOutSummaryCard.jsx
    │   └── ReturnResourceButton.jsx
    └── services/
        └── checkOutApi.js
```

Structure Decision: se respeta la estructura de capas de `plan.md` (presentation / business / domain / infrastructure). El paquete raíz es `com.university.sanctions`. La lógica de cierre y validación vive en `business/service/CheckOutService.java` y el modelo en `domain/model/`. Los puertos de salida se mantienen en `domain/port/out` para evitar acoplar el dominio a la implementación. El frontend se organiza por pantalla y componentes reutilizables.

## Decisiones de diseño de este caso de uso

| FR | Qué pide | Dónde se implementa |
|---|---|---|
| FR-001 | Registrar el check-out de una utilización activa | `CheckOutService.register(...)` |
| FR-002 | Validar que exista la utilización y el recurso | `UsageRepository` + `ResourceLookupPort` |
| FR-003 | Finalizar la utilización | `CheckOutService` y `Usage` state |
| FR-004 | Detectar si fue a tiempo o con retraso | `usage.deadline` + `Clock.now()` |
| FR-005 | Disparar score si el check-out fue a tiempo | `UpdateTrustScorePort.increaseForOnTimeCheckOut(...)` |
| FR-006 | Disparar penalización si el check-out fue tarde | `CalculatePenaltyPort.calculate(...)` |
| FR-007 | Permitir reporte de daño durante el check-out | `ReportDamagePort` |
| FR-008 | Evitar duplicados por utilización | clave única en `usage_id` |
| FR-009 | Rechazar check-out sin utilización activa | `UsageNotFoundException` / `UsageAlreadyClosedException` |
| FR-010 | UI de confirmación de devolución | `CheckOutPage.jsx` |

## Contratos

### 1. Puerto de entrada — RegisterCheckOutPort

```java
public interface RegisterCheckOutPort {
    CheckOutResult register(String studentCode, String usageId, Instant returnedAt, String observedDamageDescription);
}
```

### 2. Evento a M2 — `module3.reservation.check-out.v1`

Contrato copiado de [Módulo 2 UC12, Contratos §1](../plan-uc12-recibir-check-out.md). Para la devolución de un activo, M3 publica:

```json
{
  "eventId": "7b3e9c41-5a28-4f60-93d7-1e8a0c2f5b64",
  "type": "CheckOutRegistered",
  "version": 1,
  "occurredAt": "2026-09-09T16:42:11-05:00",
  "data": {
    "reservationId": "4b8e2a16-9c37-4d58-b1fa-6e0c74d9b2a3",
    "occurredAt": "2026-09-09T16:40:00-05:00"
  }
}
```

M2 exige `reservationId` y `data.occurredAt`. `eventId` es la clave de idempotencia del evento. El `occurredAt` superior es cuándo M3 publicó el evento; `data.occurredAt` es cuándo ocurrió la devolución y es el instante de negocio. No se envían descripciones ni datos del daño a M2: el daño se reporta a M1 mediante UC8.

Para revisión de un espacio, UC12 define este evento:

```json
{
  "eventId": "a5f2d738-9b61-4c05-87ea-3d0c1b9f4e26",
  "type": "CheckOutRegistered",
  "version": 1,
  "occurredAt": "2026-09-01T12:15:40-05:00",
  "data": {
    "reservationId": "9f3c1d7e-5b42-4a19-8c0d-2f7e6a1b3c45",
    "occurredAt": "2026-09-01T12:10:00-05:00",
    "verdict": "REQUIERE_MANTENIMIENTO"
  }
}
```

`verdict` solo se incluye para espacios (`SIN_NOVEDAD` o `REQUIERE_MANTENIMIENTO`), nunca para una devolución de activo. No se incluyen descripciones ni datos de daño.

### 3. Acuse de M2 — `module2.reservation.check-out-ack.v1`

Contrato copiado de [Módulo 2 UC12, Contratos §2](../plan-uc12-recibir-check-out.md). M3 consume el acuse para registrar el resultado del cierre. Ejemplo aceptado para un activo:

```json
{
  "eventId": "e1c8b350-7d24-4a96-b0f3-5c2e9a1d7b48",
  "type": "CheckOutAcknowledged",
  "version": 1,
  "occurredAt": "2026-09-09T16:42:14-05:00",
  "data": {
    "reservationId": "4b8e2a16-9c37-4d58-b1fa-6e0c74d9b2a3",
    "sourceEventId": "7b3e9c41-5a28-4f60-93d7-1e8a0c2f5b64",
    "kind": "ASSET_RETURN",
    "accepted": true,
    "occurredAt": "2026-09-09T16:40:00-05:00",
    "receivedAt": "2026-09-09T16:42:13-05:00",
    "dueAt": "2026-09-10T22:00:00-05:00",
    "loanClosed": true,
    "quotaReleased": true
  }
}
```

Para un espacio, UC12 define este acuse:

```json
{
  "data": {
    "reservationId": "9f3c1d7e-5b42-4a19-8c0d-2f7e6a1b3c45",
    "sourceEventId": "a5f2d738-9b61-4c05-87ea-3d0c1b9f4e26",
    "kind": "SPACE_REVIEW",
    "accepted": true,
    "occurredAt": "2026-09-01T12:10:00-05:00",
    "receivedAt": "2026-09-01T12:15:42-05:00",
    "verdict": "REQUIERE_MANTENIMIENTO",
    "verdictRecorded": true,
    "slotReleased": false
  }
}
```

Ejemplo de rechazo de UC12:

```json
{
  "data": {
    "reservationId": "9f3c1d7e-5b42-4a19-8c0d-2f7e6a1b3c45",
    "sourceEventId": "c4b1a826-3e57-4d09-96fa-8d2e0b7c1f35",
    "accepted": false,
    "rejection": "SLOT_NOT_FINISHED",
    "detail": "La franja de esta reserva termina a las 12:00. La revisión se admite después de esa hora.",
    "acceptedFrom": "2026-09-01T12:00:00-05:00"
  }
}
```

M3 correlaciona el acuse con el evento enviado mediante `data.sourceEventId`; usa `data.dueAt` junto con `data.occurredAt` para determinar si hubo retraso. Para una revisión de espacio, UC12 define `kind: SPACE_REVIEW`, `verdict`, `verdictRecorded` y `slotReleased: false`. Los rechazos de negocio se procesan según los códigos definidos en UC12; un acuse de evento ya procesado puede venir como `accepted: true` con `duplicate: true`.

### 4. Configuración Kafka

```properties
reservations.kafka.topic.check-out=module3.reservation.check-out.v1
reservations.kafka.topic.check-out-ack=module2.reservation.check-out-ack.v1
```

### 5. Respuesta de negocio

```json
{
  "checkoutId": "co-88417",
  "usageId": "usg-20260930-0187",
  "studentCode": "2023123456",
  "resourceId": "ACT-004512",
  "returnedAt": "2026-10-05T11:15:00-05:00",
  "isOnTime": true,
  "damageReported": false,
  "status": "COMPLETED"
}
```

### 6. Errores

| Código | Descripción |
|---|---|
| 404 | `USAGE_NOT_FOUND` |
| 409 | `USAGE_ALREADY_CLOSED` |
| 422 | `RESOURCE_NOT_IDENTIFIED` |
| 500 | `PERSISTENCE_ERROR` |

## Fases de implementación

### Phase 1: Foundation
- Validar los estados de uso y fechas de devolución
- Definir el modelo `CheckOut` y `Usage`
- Crear repositorios y adaptadores

### Phase 2: Core flow
- Implementar `CheckOutService`
- Finalizar utilización
- Disparar `UpdateTrustScorePort` o `CalculatePenaltyPort`

### Phase 3: Damage path
- Activar `ReportDamagePort` si el estudiante reporta daño al cerrar
- Confirmar que la operación de cierre no se revierte

### Phase 4: UI and tests
- Pantalla de confirmación
- Tests unitarios, contract y de integración
