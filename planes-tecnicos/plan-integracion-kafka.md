# Implementation Plan: Integración Kafka de M3 con M1 y M2

**Date**: 2026-10-05
**Fuentes del contrato**: [KAFKA.md](../KAFKA.md) y [PLAN (2).md](../PLAN%20%282%29.md), documentos recibidos de M1; [UC9](../plan-uc9-recibir-reporte-no-asistencia.md) y [UC12](../plan-uc12-recibir-check-out.md), planes recibidos de M2
**Plan general**: [plan.md](../Especs/plan.md)
**Módulo que mantiene este plan**: Módulo 3

## Propósito y alcance

Este plan reúne los contratos de mensajería de M3 con M1 y M2. No reemplaza ni modifica los documentos fuente: registra los nombres, topics y campos que aparecen en ellos, y separa explícitamente los contratos fuente de lo que aún requiere acuerdo entre módulos.

M3 persiste el cambio local y el mensaje en el outbox en una misma transacción. El publicador común de M3 entrega el evento a Kafka. Un acuse técnico del broker no confirma que M1 haya consumido el mensaje o actualizado el inventario.

## Integración con M1

| Flujo M3 → M1 | Nombre en los documentos de M1 | Uso en M3 |
|---|---|---|
| Finalización regular | `reserva.finalizada` en `KAFKA.md`; `reserva-finalizada` en `PLAN (2).md`; tipo `EndedReservation` | UC6/check-out cuando el recurso se devuelve en estado óptimo |
| Reporte de daño | `reporte.danos` en `KAFKA.md`; `reporte-de-daños` en `PLAN (2).md`; tipo `DamageReport` | UC8/reportar novedad técnica |

Ambos se proponen en el topic `unimag.m3.uso-novedades.v1`. La tabla de topología marca el nombre del topic con `@@@`, por lo que el valor no se considera confirmado para despliegue.

### Contrato de reporte de daño propuesto por M1

Los siguientes nombres aparecen en el ejemplo de `KAFKA.md`; se conservan tal como están en la fuente hasta que M1 publique una versión corregida:

| Campo propuesto | Nombre en el ejemplo | Observaciones |
|---|---|---|
| Identificador del evento | `eventoId` | Difiere de `eventId`, usado por el resto de los ejemplos de `KAFKA.md`; confirmar nombre canónico. |
| Tipo | `type: "DamageReport"` | Identificador del tipo de evento. |
| Versión | `version: "1.0"` | Se presenta como string en el ejemplo de daño. |
| Fecha/hora | `timestamp` | ISO-8601 con offset en el ejemplo. |
| Módulo emisor | `source: "MODULO_3"` | Coherente con el estándar general de headers. |
| Recurso | `data.resourceId` | Número en el ejemplo, aunque otros pasajes describen la clave Kafka como string. |
| Categoría | `data.recourseCategory` | Errata aparente; otros ejemplos escriben `resourceCategory`. No renombrar sin confirmación. |
| Reserva M2 | `data.id_reservation_m2` | Campo nullable/no definido para daños detectados fuera de una utilización; M1 debe precisar obligatoriedad. |
| Novedad M3 | `data.id_novedad_m3` | Candidato a correlación con el reporte local; confirmar si ese es el identificador requerido. |
| Estudiante | `data.student.student_code`, `data.student.mail` | La fuente no indica si ambos son obligatorios ni si M1 necesita estos datos. |
| Descripción | `data.description` | Descripción textual del daño. |
| Estado físico reportado | `data.physical_state` | El ejemplo usa `DAÑADO`; M1 determina la transición de disponibilidad. |

El ejemplo contiene una coma final inválida en `physical_state`, por lo que no es JSON válido. Tampoco define claramente los campos opcionales ni el evento de confirmación de consumo. No construir el DTO de producción a partir de una “corrección” inferida de estas discrepancias.

### Clave Kafka y particionamiento

`KAFKA.md` describe la clave como `recurso_id` (String) en la tabla de topics, como `resourceId` en prosa y como número en los payloads. `PLAN (2).md` afirma que M1 consume los eventos `reserva-finalizada` y `reporte-de-daños`, pero no resuelve esta inconsistencia de la clave. El nombre y tipo canónicos de la key deben confirmarse con M1. La intención documentada es ordenar los eventos por recurso.

### Responsabilidad de M1

`KAFKA.md` indica para `DamageReport` que M1 pasa la disponibilidad a `EN_MANTENIMIENTO`, actualiza el estado físico a `MAL_ESTADO` o `DAÑADO` y registra la novedad en `Historial_Recurso`. `PLAN (2).md` confirma que los cambios de inventario M3→M1 deben usar eventos Kafka, no el endpoint REST de cambio manual. M1 determina y aplica la transición del inventario; M3 no le envía un costo del recurso.

## Integración con M2

Los contratos siguientes se extraen de UC9 y UC12 de M2. M3 publica los hechos de no asistencia y check-out; M2 publica los acuses para que M3 pueda correlacionar el resultado. El cálculo de penalización permanece interno a M3: no se publica a M2 un evento de penalización.

### UC9 — reporte y anulación de no asistencia

**M3 → M2**, topic `module3.reservation.no-show.v1`. UC9 define `NoShowReported` y también `NoShowReportVoided` en el mismo topic. La clave de partición indicada por UC9 es `reservation_id`, para conservar el orden por reserva.

```json
{
  "eventId": "6a1f3b84-2c57-4e90-81d6-9f4e0a7c3b25",
  "type": "NoShowReported",
  "version": 1,
  "occurredAt": "2026-09-01T10:11:03-05:00",
  "data": {
    "reservationId": "9f3c1d7e-5b42-4a19-8c0d-2f7e6a1b3c45",
    "verifiedAt": "2026-09-01T10:10:30-05:00",
    "verifiedBy": "MODULO_3",
    "note": "Se verificó en sitio a los 10 minutos."
  }
}
```

`reservationId` y `verifiedAt` son obligatorios; `verifiedBy` y `note` son opcionales. `verifiedAt` es la hora de constatación y no debe sustituirse por la hora de emisión o recepción. UC9 especifica para la anulación el tipo `NoShowReportVoided`, con `reservationId` y `voidedReason` en `data`; también va por el topic anterior.

```json
{
  "eventId": "b9d2e5a7-4f81-4c36-92be-7a0c1d8f3e64",
  "type": "NoShowReportVoided",
  "version": 1,
  "occurredAt": "2026-09-01T11:02:14-05:00",
  "data": {
    "reservationId": "9f3c1d7e-5b42-4a19-8c0d-2f7e6a1b3c45",
    "voidedReason": "El reporte se envió por error: la persona sí se presentó."
  }
}
```

**M2 → M3**, topic `module2.reservation.no-show-ack.v1`, tipo `NoShowReportAcknowledged`. M2 emite un acuse por reporte, tanto si se acepta como si se rechaza. M3 correlaciona el acuse con `data.sourceEventId`.

```json
{
  "eventId": "e7c4a018-5b93-4d27-86fa-1c2e9d0b4f73",
  "type": "NoShowReportAcknowledged",
  "version": 1,
  "occurredAt": "2026-09-01T10:11:05-05:00",
  "data": {
    "reservationId": "9f3c1d7e-5b42-4a19-8c0d-2f7e6a1b3c45",
    "sourceEventId": "6a1f3b84-2c57-4e90-81d6-9f4e0a7c3b25",
    "accepted": true,
    "absenceRegisteredAt": "2026-09-01T10:11:05-05:00",
    "resourceReleased": true,
    "holder": { "code": "2019114045" },
    "reservedTime": {
      "start": "2026-09-01T10:00:00-05:00",
      "end": "2026-09-01T12:00:00-05:00"
    }
  }
}
```

En rechazos, UC9 define `accepted: false`, `rejection` y `detail`; `TOO_EARLY` incluye `acceptedFrom`, y `RESERVATION_CANCELLED` incluye `reservationStatus`. El detalle de los demás códigos permanece en el contrato fuente UC9.

### UC12 — check-out de activo o revisión de espacio

**M3 → M2**, topic `module3.reservation.check-out.v1`, tipo `CheckOutRegistered`. El mismo tipo cubre la devolución de un activo y la revisión de un espacio:

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

`data.reservationId` y `data.occurredAt` son obligatorios. En una revisión de espacio se añade `data.verdict`, con `SIN_NOVEDAD` o `REQUIERE_MANTENIMIENTO`; no se envía `verdict` para un activo. No se incluyen descripción ni campos de daños: ese reporte se envía a M1 conforme a UC8.

**M2 → M3**, topic `module2.reservation.check-out-ack.v1`, tipo `CheckOutAcknowledged`. `data.sourceEventId` correlaciona la respuesta con el evento de M3. Para un activo, UC12 define `kind: ASSET_RETURN`, `accepted`, `occurredAt`, `receivedAt`, `dueAt`, `loanClosed` y `quotaReleased`. Para un espacio define `kind: SPACE_REVIEW`, `accepted`, `occurredAt`, `receivedAt`, `verdict`, `verdictRecorded` y `slotReleased: false`. Un rechazo incluye `accepted: false`, `rejection` y `detail`; `SLOT_NOT_FINISHED` además incluye `acceptedFrom`.

Ejemplo de acuse aceptado de activo, según UC12:

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

El acuse de espacio usa `kind: SPACE_REVIEW`, agrega `verdict`, `verdictRecorded` y `slotReleased: false`. UC12 no define en este plan un topic Kafka para `LoanDeclaredLost`, pero sí propone este payload para el evento M2 → M3:

```json
{
  "eventId": "f6a3c914-8b50-4e27-83dc-1f9b0d2e5a76",
  "type": "LoanDeclaredLost",
  "version": 1,
  "occurredAt": "2026-09-17T03:00:00-05:00",
  "data": {
    "reservationId": "4b8e2a16-9c37-4d58-b1fa-6e0c74d9b2a3",
    "status": "NO_DEVUELTO_PERDIDO",
    "dueAt": "2026-09-10T22:00:00-05:00",
    "declaredAt": "2026-09-17T03:00:00-05:00",
    "overdueDays": 7,
    "resource": { "id": "ACT-004512", "assetTag": "ACT-004512", "category": "ACTIVO" },
    "holder": {
      "userId": "5f1b9c2d-7a34-4e81-b0f6-3c8d1e9a4b72",
      "code": "2019114045",
      "name": "Nombre del estudiante"
    }
  }
}
```

Ese mensaje no incluye valoración económica ni prescribe una sanción; cualquier penalización se calcula internamente en M3. `LoanDeclaredLost` se registra como **pendiente de contrato** —topic, clave y consumo deben acordarse—, no como integración lista para implementar ni como uno de los dos flujos confirmados de check-out.

## Contrato interno de fallo del reporte desde UC6

UC6 requiere que un fallo al iniciar el subflujo UC8 no impida completar el check-out. Se define en este plan un evento interno de M3 para representar el rechazo del reporte, aunque los otros planes todavía no implementen un consumidor:

**Nombre**: `DamageReportRegistrationRejected`
**Canal**: evento de aplicación/dominio interno de M3. No se publica a Kafka ni a M1.
**Productor**: UC8, cuando el intento de registrar el daño se rechaza por una condición conocida.
**Consumidor previsto**: UC6, en una etapa posterior, para informar que el check-out se completó pero el reporte de daño no quedó registrado.

```java
public record DamageReportRegistrationRejected(
    String eventId,
    String usageId,
    DamageReportRejectionCode code,
    Instant occurredAt
) {}

public enum DamageReportRejectionCode {
    MISSING_DESCRIPTION,
    RESOURCE_NOT_IDENTIFIED,
    DUPLICATE_DAMAGE_REPORT,
    USAGE_NOT_DETERMINABLE
}
```

El evento solo describe un rechazo del subflujo de reporte; no afirma que el daño haya sido registrado, publicado o procesado por M1. `usageId` correlaciona el rechazo con el check-out que lo originó. El intento inválido no crea fila de reporte ni mensaje `DamageReport` en outbox. Los fallos inesperados de infraestructura no se convierten silenciosamente en este resultado de negocio: deben exponerse y manejarse según las reglas de error de M3.

El contrato queda definido, pero **no se presupone que UC6 ya consuma el evento ni que exista una notificación al estudiante**. Hasta que se implemente ese consumidor, UC8 devuelve un resultado explícito de rechazo al llamador. Este desacoplamiento no cambia la regla funcional de UC6: el check-out se completa aunque el reporte de daño no se registre.

## Tareas

- [ ] K001 Confirmar con M1 el nombre final del topic `unimag.m3.uso-novedades.v1` (marcador `@@@`).
- [ ] K002 Confirmar identificador canónico (`eventId` o `eventoId`), key Kafka y tipo de key.
- [ ] K003 Corregir con M1 `recourseCategory` y la coma final, y acordar obligatoriedad/tipos de `id_reservation_m2`, `id_novedad_m3` y campos `student`.
- [ ] K004 Acordar cómo representar un daño independiente que no puede asociarse a una reserva/estudiante.
- [ ] K005 Confirmar valores de `physical_state` y que M1 calcula por sí mismo el estado de disponibilidad.
- [ ] K006 Implementar/validar el mapper, outbox y pruebas de contrato para los eventos M1 una vez acordado el esquema.
- [ ] K007 Mantener `DamageReportRegistrationRejected` como contrato interno; implementar su consumo desde UC6 cuando se actualice el plan técnico de check-out.
- [ ] K008 Acordar con M2 los topics de acuse UC9/UC12, los campos de cada sobre/payload, la clave de partición y la semántica de los acuses.
- [ ] K009 Acordar si M3 consumirá `LoanDeclaredLost` y, de ser así, definir topic, clave, esquema y comportamiento consumidor antes de implementarlo.

## Referencias relacionadas

- [Plan técnico UC8: Reportar novedad técnica](./plan-uc-reportar-novedad-tecnica.md)
- [Plan técnico UC6: Realizar check-out](./plan-uc-realizar-checkout.md)
- [KAFKA.md (fuente M1)](../KAFKA.md)
- [PLAN (2).md (fuente M1)](../PLAN%20%282%29.md)
- [UC9 (fuente M2)](../plan-uc9-recibir-reporte-no-asistencia.md)
- [UC12 (fuente M2)](../plan-uc12-recibir-check-out.md)
