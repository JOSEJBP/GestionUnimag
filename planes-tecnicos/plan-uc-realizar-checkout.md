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

**No consume eventos de M2 para este UC.** El check-out usa los datos locales de la utilización y del recurso para decidir la operación.

### Lo que M3 produce para M2

M3 publica confirmación de check-out por Kafka cuando el registro fue exitoso.

| Evento | Topic | Cuándo se emite | Uso |
|---|---|---|---|
| `CheckOutRecorded` | `module3.usage.checkout-recorded.v1` | cuando el check-out queda registrado | M2 confirma que la devolución terminó correctamente |

### Endpoints que M3 expone para M2

**No expone endpoints nuevos a M2.** Este UC se usa internamente por el frontend de M3.

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
│   │       └── CheckOutRecordedEvent.java
│   ├── domain/
│   │   ├── model/
│   │   │   ├── CheckOut.java
│   │   │   ├── Usage.java
│   │   │   ├── ResourceView.java
│   │   │   └── CheckoutStatus.java
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   └── RegisterCheckOutPort.java
│   │   │   └── out/
│   │   │       ├── UsageRepository.java
│   │   │       ├── ResourceLookupPort.java
│   │   │       ├── UpdateTrustScorePort.java
│   │   │       ├── CalculatePenaltyPort.java
│   │   │       └── ReportDamagePort.java
│   │   └── error/
│   │       ├── UsageNotFoundException.java
│   │       ├── UsageAlreadyClosedException.java
│   │       └── ResourceNotIdentifiedException.java
│   │
│   └── infrastructure/
│       ├── persistence/
│       │   └── jpa/
│       │       ├── entity/
│       │       │   └── CheckOutEntity.java
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
└── src/test/java/com/university/sanctions/
    ├── contract/
    │   └── CheckOutControllerTest.java
    ├── integration/
    │   └── CheckOutIT.java
    └── unit/
        ├── CheckOutServiceTest.java
        └── UsageStateTest.java

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

### 2. Respuesta de negocio

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

### 3. Errores

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
