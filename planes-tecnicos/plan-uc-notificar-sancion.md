# Implementation Plan: Notificar sanción

**Date**: 2026-10-05
**Spec**: [notificar-sancion.md](../Especs/notificar-sancion.md)
**Plan general**: [plan.md](../Especs/plan.md)
**Módulo**: Módulo 3 — Sanciones y Cumplimiento
**Planes relacionados**: [plan-uc1-actualizar-score.md](./plan-uc1-actualizar-score.md) — produce los cambios de score; [plan-uc2-calcular-penalizacion.md](./plan-uc2-calcular-penalizacion.md) — lo invoca al registrar una penalización; [plan-uc3-bloquear-usuario.md](./plan-uc3-bloquear-usuario.md) — produce bloqueos/desbloqueos por score

> **Contrato propuesto:** no se encontró un contrato publicado del Módulo 2 para estos eventos. El topic y el esquema descritos aquí son una propuesta de M3 para planificar e implementar el productor; deben validarse con M2 antes de integrar en ambientes compartidos.

## Summary

Este caso de uso transforma cambios de score, penalizaciones y bloqueos en solicitudes de notificación para el estudiante. La notificación se registra de forma durable dentro de M3 y se publica a Módulo 2 por Kafka, sin hacer que el flujo que produjo el cambio espere a que M2 procese el mensaje.

**Enfoque técnico:**

1. Implementa `NotifyPenaltyPort` para que `Calcular penalización` pueda solicitar la notificación con el resultado ya registrado.
2. Recibe también los cambios de score y los bloqueos/desbloqueos desde sus puertos de integración internos. No vuelve a calcular score, sanciones ni estado de bloqueo.
3. Valida que cada solicitud tenga destinatario, tipo de cambio y datos obligatorios; no construye ni publica mensajes con información incompleta.
4. En la misma transacción local del caso de uso originador, registra la solicitud en el outbox y una referencia idempotente. La publicación a Kafka ocurre después del commit.
5. Un publicador reintenta errores transitorios con backoff. La clave de idempotencia evita que un reintento de M3 produzca dos solicitudes de notificación para el mismo evento.
6. Un evento de M3 confirma que la solicitud fue publicada/aceptada por el broker, no que el estudiante ya leyó el mensaje. La confirmación de entrega final depende de M2 y queda señalada como decisión pendiente.

## Technical Context

**Language/Version**: Java 21 (backend).
**Primary Dependencies**: Spring Boot 4.x, Spring Data JPA, Spring Validation, Spring Security OAuth2 Resource Server, Spring for Apache Kafka y Flyway. Reutiliza la configuración Kafka, outbox y dispatcher del plan general.
**Storage**: MySQL 8. Reutiliza `outbox_message`; añade `notification_request` para trazabilidad e idempotencia si la estructura existente del outbox no conserva los datos de negocio requeridos.
**Testing**: JUnit 5 + Mockito, `@WebMvcTest` si se exponen consultas administrativas, Testcontainers MySQL y Kafka, ArchUnit.
**Target Platform**: Servidor Linux con JVM 21.
**Project Type**: Servicio interno del backend M3; no requiere una página independiente.
**Performance Goals**: publicar el evento en menos de 5 segundos desde el cambio originador (SC-001). La escritura del outbox debe permanecer en el camino local y no esperar a Kafka.
**Constraints**:
- Las solicitudes de score deben llevar motivo, delta y score resultante.
- Las reducciones que provienen de una penalización deben incluir los días de suspensión cuando existan.
- Los bloqueos incluyen motivo y condiciones de reactivación; los desbloqueos incluyen motivo y score actual.
- Una falla de publicación no revierte score, sanción ni bloqueo.
- Los valores de score se validan en el rango de negocio de 0 a 50.
- La zona horaria es `America/Bogota`; los timestamps externos usan ISO-8601 con offset.

## Integración con otros módulos

### Lo que M3 consume de Módulo 2

**No consume eventos de M2 en esta propuesta.** El flujo de notificación es productor M3 → consumidor M2. Si M2 publica confirmaciones de entrega en el futuro, se podrán integrar detrás de un puerto de entrada separado; no se asume esa confirmación para el primer corte.

### Lo que M3 produce para Módulo 2

Propuesta de topic versionado. M2 debe consumirlo y realizar la entrega al estudiante:

| Evento | Topic propuesto | Clave Kafka | Cuándo |
|---|---|---|---|
| `NotificationRequested` | `module3.sanctions.notification-requested.v1` | `studentCode` | Cuando queda registrado un cambio notificable |

La clave por estudiante conserva el orden de los eventos de una misma persona. Cada mensaje lleva un `eventId` único para deduplicación, independiente de la clave Kafka.

### Endpoints

**No expone endpoints REST a M2 ni al frontend para enviar notificaciones.** El contrato de entrada a este caso de uso son puertos internos de M3. Un endpoint de consulta administrativa de entregas puede añadirse si el equipo lo necesita; no es requisito de la spec.

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan.md
├── notificar-sancion.md
├── actualizar-score-confianza.md
├── calcular-penalizacion.md
└── bloquear-usuario.md

planes-tecnicos/
├── plan-uc-notificar-sancion.md
├── plan-uc1-actualizar-score.md
├── plan-uc2-calcular-penalizacion.md
└── plan-uc3-bloquear-usuario.md
```

### Source Code (repository root)

Solo los archivos que crea o toca este plan. La estructura completa de capas está en [plan.md](../Especs/plan.md).

```text
backend/
└── src/main/java/com/university/sanctions/
    ├── business/
    │   └── service/NotificationService.java
    ├── domain/
    │   ├── model/
    │   │   ├── NotificationRequest.java
    │   │   └── NotificationType.java
    │   └── port/
    │       ├── in/
    │       │   └── RequestNotificationPort.java
    │       └── out/
    │           ├── NotifyPenaltyPort.java
    │           ├── NotificationRequestRepository.java
    │           └── NotificationOutboxPort.java
    ├── infrastructure/
    │   ├── persistence/jpa/
    │   │   ├── entity/NotificationRequestEntity.java
    │   │   ├── repository/JpaNotificationRequestRepository.java
    │   │   ├── adapter/NotificationRequestRepositoryAdapter.java
    │   │   └── mapper/NotificationRequestMapper.java
    │   └── integration/module2/kafka/
    │       ├── NotificationRequestedEvent.java
    │       └── NotificationRequestedPublisher.java
    └── src/main/resources/db/migration/
        └── V__notification_request.sql
```

El evento de negocio y el evento de integración son tipos distintos: el primero modela la solicitud de M3; el segundo es el payload versionado serializado a Kafka. El dispatcher debe reutilizar la infraestructura outbox ya definida en `plan.md`, no crear un publicador paralelo.

## Decisiones de diseño de este caso de uso

| FR | Qué pide | Dónde se implementa |
|---|---|---|
| FR-001 | Notificar reducciones y aumentos de score | Adaptadores internos llaman `RequestNotificationPort` al registrar el cambio |
| FR-002, FR-003 | Notificar bloqueo y desbloqueo por score | Adaptador de UC3 llama el mismo puerto con tipo y contexto de bloqueo |
| FR-004, FR-005 | Incluir motivo, delta, score y suspensión cuando aplique | `NotificationRequest` y mapper del evento |
| FR-006, FR-007 | Incluir causa/condiciones del bloqueo o motivo/score del desbloqueo | Tipos `BLOCKED` y `UNBLOCKED` del request |
| FR-008 | Reintentar fallos de publicación | `OutboxPublisher` con backoff y número de intentos persistido |
| FR-009 | Evitar duplicados ante reintentos | Clave única `(sourceEventId, notificationType)` |
| FR-010 | No afectar el cambio originador | Insertar el outbox en la transacción local; publicar después del commit |

1. **No enviar desde el hilo del caso de uso.** El puerto registra el mensaje de forma transaccional; el dispatcher realiza la publicación asíncrona.
2. **Idempotencia del negocio y del transporte.** `sourceEventId` se deriva del evento originador y es único para cada notificación/tipo. Kafka usa `eventId` para que M2 deduplique mensajes reentregados.
3. **Sin información incompleta.** La validación ocurre antes de persistir; una solicitud inválida genera un error explícito y no se convierte en una notificación genérica.
4. **No confundir publicación con entrega.** La confirmación Kafka acredita publicación en el broker. El estado `PUBLISHED` no prueba que M2 haya entregado el mensaje al estudiante.
5. **Orden por estudiante.** `studentCode` será la clave Kafka propuesta. M2 puede procesar mensajes de estudiantes distintos en paralelo.

## Contratos

### 1. Puerto de salida — NotifyPenaltyPort

Se conserva el método ya referenciado por el plan de Calcular penalización. UC6 adapta su resultado a `NotificationRequest` y lo registra en el outbox.

```java
public interface NotifyPenaltyPort {
    void notify(PenaltyResult result);
}
```

### 2. Puerto interno — RequestNotificationPort

Lo usan los flujos de score y bloqueo para solicitudes que no nacen de una penalización.

```java
public interface RequestNotificationPort {
    void request(NotificationRequest request);
}

public record NotificationRequest(
    String sourceEventId,
    String studentCode,
    NotificationType type,
    String reason,
    Integer scoreDelta,
    Integer currentScore,
    Integer suspensionDays,
    String reactivationConditions,
    Instant occurredAt
) {}

public enum NotificationType {
    SCORE_DECREASE,
    SCORE_INCREASE,
    BLOCKED,
    UNBLOCKED
}
```

Los campos que no aplican a un tipo se omiten al serializar; no se envían como `null`. `sourceEventId`, `studentCode`, `type`, `reason` y `occurredAt` son obligatorios. Delta y score son obligatorios para cambios de score; condiciones de reactivación para bloqueos; score actual para desbloqueos. `suspensionDays` es obligatorio únicamente cuando la penalización lo aplica.

### 3. Evento propuesto para Módulo 2

```json
{
  "eventId": "2d70c402-f34f-4d91-9f28-e0d5421a6b54",
  "eventType": "NotificationRequested",
  "occurredAt": "2026-10-05T11:15:00-05:00",
  "studentCode": "2023123456",
  "notificationType": "SCORE_DECREASE",
  "sourceEventId": "pen-10231",
  "reason": "Infracción por retraso",
  "scoreDelta": -5,
  "currentScore": 45,
  "suspensionDays": 3
}
```

El campo `suspensionDays` solo se incluye si aplica. Para `BLOCKED`, el payload incluye `reason` y `reactivationConditions`; para `UNBLOCKED`, incluye `reason` y `currentScore`. El topic `module3.sanctions.notification-requested.v1`, este envelope y las reglas de compatibilidad son **propuestas de M3**, pendientes de validación con M2.

### 4. Estados y errores

| Estado interno | Significado |
|---|---|
| `PENDING` | Solicitud guardada en outbox, pendiente de publicación |
| `PUBLISHED` | Kafka confirmó que el broker recibió el evento |
| `RETRYING` | Falló un intento transitorio y hay reintento programado |
| `FAILED` | Se agotó la política de reintentos; requiere alerta/revisión |

La falla permanente se registra con código interno y datos diagnósticos sin incluir contenido sensible en logs. El evento originador conserva su estado. La confirmación de entrega final al estudiante no está cubierta mientras M2 no ofrezca un ACK.

## Fases de implementación

### Phase 1: Contratos y tipos
- [ ] T001 Definir `NotificationType`, `NotificationRequest` y sus validaciones por tipo.
- [ ] T002 Implementar `RequestNotificationPort` y `NotifyPenaltyPort` como adaptadores hacia el registro transaccional del outbox.
- [ ] T003 Confirmar la existencia y semántica de `outbox_message` y su dispatcher a partir de UC1.

### Phase 2: Persistencia e idempotencia
- [ ] T004 Añadir migración para trazabilidad de solicitud si el outbox actual no cubre fuente, tipo y estado.
- [ ] T005 Crear índice único por `source_event_id` y `notification_type`.
- [ ] T006 Persistir payload versionado con clave Kafka por estudiante.

### Phase 3: Publicación Kafka
- [ ] T007 Configurar el topic propuesto `module3.sanctions.notification-requested.v1`.
- [ ] T008 Implementar serialización, confirmación del broker, backoff y estado terminal `FAILED`.
- [ ] T009 Añadir métricas y alertas para pendientes envejecidos, retries y fallos permanentes.

### Phase 4: Pruebas
- [ ] T010 Pruebas unitarias para mapeo de cada tipo de notificación y campos obligatorios.
- [ ] T011 Pruebas de idempotencia con dos solicitudes del mismo evento.
- [ ] T012 Pruebas de integración MySQL para commit/rollback conjunto con el caso de uso originador.
- [ ] T013 Pruebas Kafka para éxito, indisponibilidad, reintento y orden por estudiante.

## Decisiones pendientes de validar

- Topic, envelope, nombres de campos y política de compatibilidad con el consumidor de M2.
- Si M2 publicará un ACK de entrega al estudiante; sin él, M3 solo puede garantizar publicación al broker.
- Cómo cumplir SC-001 (entrega en menos de cinco segundos) en condiciones de indisponibilidad o cola atrasada; la disponibilidad del consumidor M2 queda fuera del control de M3.
