# Feature Specification: Confirmar No Asistencia (Salón)

**Created:** 2026-10-17
**Updated:** 2026-10-22 (integración alineada con los contratos de M2; inasistencia a los 30 minutos)

> La sanción por inasistencia y su efecto en el score están en `reglas-negocio-m3.md` (RN-SAN-03, RN-SCO-02). Las reglas de integración con M2 están en RN-INT-01, RN-INT-02 y RN-INT-04.

## User Scenarios & Testing

### User Story 1 - Confirmar la inasistencia a una reserva de espacio (Priority: P1)
Como personal administrativo, quiero confirmar que un estudiante no se presentó a su reserva de un salón, para dejar constancia del incumplimiento y que se aplique la sanción correspondiente.

**Why this priority:** Es el único disparador de la sanción por inasistencia; sin él, este incumplimiento no quedaría penalizado y los salones reservados sin uso no tendrían consecuencia.

**Independent Test:** Con una reserva de salón cuyo inicio pasó hace más de 30 minutos, confirmar la inasistencia, simular el acuse de aceptación de M2 y verificar que se crea la sanción.

**Acceptance Scenarios:**

**Scenario: Confirmación aceptada por M2**
- **Given** una reserva de un espacio, informada por M2 y no cancelada, cuyo inicio pasó hace al menos 30 minutos, y en la que el personal constató en sitio que el estudiante no se presentó.
- **When** el personal administrativo confirma la inasistencia.
- **Then** M3 registra la confirmación (reserva, estudiante, instante de la constatación, quién confirma) con estado "Pendiente de acuse" y publica a M2 `NoShowReported`. Cuando M2 acusa que lo aceptó, la confirmación pasa a "Aceptada", M3 crea la sanción por inasistencia (RN-SAN-03) y reduce el score del estudiante una vez (RN-SCO-02).

**Scenario: Antes de los 30 minutos**
- **Given** una reserva cuyo inicio fue hace menos de 30 minutos.
- **When** se intenta confirmar la inasistencia.
- **Then** el sistema rechaza la confirmación e informa desde qué hora se admite (inicio de la reserva más 30 minutos). M2 solo exige 10 minutos, así que no debería rechazarla por anticipada; si aun así la rechaza con `TOO_EARLY` (por ejemplo, si se cambia el parámetro a menos de 10), la confirmación queda "Rechazada por M2" con la hora `acceptedFrom` y puede repetirse desde entonces.

**Scenario: Reserva cancelada**
- **Given** una reserva cancelada según los eventos que M2 informó, o que M2 rechaza con `RESERVATION_CANCELLED`.
- **When** se intenta confirmar la inasistencia.
- **Then** el sistema rechaza la confirmación e informa que la reserva fue cancelada; no se crea sanción.

**Scenario: Confirmación duplicada**
- **Given** una reserva con una inasistencia ya confirmada.
- **When** se intenta confirmar de nuevo.
- **Then** el sistema rechaza la segunda confirmación e informa que ya existe una registrada.

**Scenario: M2 rechaza el reporte**
- **Given** una confirmación "Pendiente de acuse".
- **When** M2 la rechaza por un motivo distinto de `TOO_EARLY` (por ejemplo, `RESERVATION_CANCELLED`).
- **Then** la confirmación queda "Rechazada por M2" con el motivo que M2 informó, no se crea sanción ni se modifica el score, y el personal administrativo ve el motivo en el listado.

### User Story 2 - Consultar las reservas pendientes de confirmación (Priority: P2)
Como personal administrativo, quiero ver las reservas de espacios cuyo inicio ya pasó más de 30 minutos y que aún no he confirmado, para decidir sobre cada una sin buscarlas manualmente.

**Why this priority:** Facilita la operación diaria, pero la confirmación puede hacerse sobre una reserva conocida aunque no exista este listado.

**Independent Test:** Con varias reservas de espacios pasadas, una cancelada y una ya confirmada, verificar que el listado muestra solo las pendientes.

**Acceptance Scenarios:**

**Scenario: Listado de pendientes**
- **Given** reservas de espacios, sancionables y no canceladas, cuyo inicio pasó hace más de 30 minutos, que no han sido confirmadas.
- **When** el personal administrativo abre el listado.
- **Then** M3 las toma de las reservas que M2 informó, excluye las canceladas y las ya confirmadas, y muestra, ordenadas por fecha, el espacio, el estudiante titular y la ventana de la reserva. Las confirmaciones "Rechazadas por M2" aparecen aparte con su motivo.

**Scenario: Sin pendientes**
- **Given** ninguna reserva pendiente de confirmar.
- **When** el personal administrativo abre el listado.
- **Then** el sistema muestra un estado vacío informativo.

### User Story 3 - Anular una inasistencia confirmada por error (Priority: P2)
Como personal administrativo, quiero anular una inasistencia que confirmé por error, indicando el motivo, para que el estudiante no quede sancionado injustamente.

**Why this priority:** Confirmar la ausencia depende de una constatación en sitio y puede equivocarse; sin una forma de corregirla, el estudiante cargaría con una sanción que no merece.

**Independent Test:** Con una inasistencia aceptada y su sanción vigente, anularla con motivo y verificar que la sanción queda anulada, que los puntos se restituyen y que M2 recibe `NoShowReportVoided`.

**Acceptance Scenarios:**

**Scenario: Anulación exitosa**
- **Given** una inasistencia "Aceptada" con su sanción vigente, que descontó 5 puntos.
- **When** el personal administrativo la anula e indica el motivo.
- **Then** la confirmación y la sanción pasan a "Anulada", el score recupera los puntos descontados (RN-SCO-12), la inasistencia deja de contar en `absenceCount` y como infracción, se guarda la auditoría (quién, cuándo y motivo) y M3 publica `NoShowReportVoided` a M2 (RN-SAN-14).

**Scenario: Anulación de una sanción ya cumplida**
- **Given** una inasistencia cuya sanción ya terminó por fecha.
- **When** el personal administrativo la anula con motivo.
- **Then** la confirmación y la sanción pasan a "Anulada" y se restituyen los puntos que descontó.

**Scenario: Anulación sin motivo**
- **Given** una inasistencia aceptada.
- **When** el personal administrativo intenta anularla sin motivo.
- **Then** el sistema rechaza la operación e informa que el motivo es obligatorio.

**Scenario: Confirmación ya anulada**
- **Given** una inasistencia anulada.
- **When** se intenta anular de nuevo.
- **Then** el sistema rechaza la operación e informa que ya fue anulada.

## Edge Cases

- **Reserva no encontrada en las reservas informadas por M2:** el sistema debe rechazar la confirmación e informar que no fue posible identificar la reserva.
- **Reserva que no es de un espacio, o académica / no sancionable:** el sistema debe rechazar la confirmación; este caso de uso aplica solo a reservas estudiantiles de salones y auditorios. M2 admite el reporte también para retiros de activos, pero la inasistencia por activos queda fuera de M3.
- **Estudiante titular no identificable:** el sistema debe rechazar la confirmación e informar que no fue posible asociarla a un estudiante válido.
- **M2 sin responder (sin acuse):** la confirmación queda "Pendiente de acuse" y el evento pendiente de reintento sin duplicarse; no se crea sanción hasta que M2 acuse. El listado muestra cuántas confirmaciones esperan acuse.
- **Evento duplicado o acuse reentregado:** se procesan una sola vez por `eventId`; un acuse con `duplicate` no crea otra sanción (RN-SAN-04, RN-INT-02).
- **Reserva sin confirmar que nadie reporta:** no se aplica ninguna sanción mientras no exista confirmación aceptada. No hay plazo máximo: el personal puede confirmar en cualquier momento después de los 30 minutos (RN-SAN-03).
- **Estudiante que llega después de los 30 minutos pero antes de que el personal confirme:** el personal constata en sitio, así que no confirma la inasistencia; M3 no sanciona sin confirmación.
- **Inasistencia confirmada por error:** se corrige con la anulación de la User Story 3, en cualquier momento y con motivo obligatorio.
- **Anulación de una confirmación que M2 rechazó o que sigue "Pendiente de acuse":** no hay sanción ni puntos que restituir; la confirmación pasa a "Anulada" y no se envía nada a M2 si nunca fue aceptada.
- **M2 no disponible al anular:** la anulación se registra en M3 (sanción anulada y puntos restituidos) y el evento a M2 queda pendiente de reintento, sin duplicarse.
- **Falta el parámetro de duración de la sanción:** la confirmación aceptada se mantiene, no se crea la sanción con valores supuestos y la inconsistencia queda marcada para reintento (RN-SAN-09).
- **Error al registrar la confirmación:** el sistema debe informar que no pudo registrarse y evitar dejar un registro incompleto.
- **Error al aplicar la sanción o reducir el score tras el acuse:** la confirmación se mantiene "Aceptada" y la inconsistencia queda marcada para reintento.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe listar las reservas de espacios sancionables, no canceladas, cuyo inicio pasó hace al menos 30 minutos y que estén pendientes de confirmación, a partir de las reservas y cancelaciones que M2 informó (RN-INT-01) y excluyendo las ya confirmadas.
- **FR-002:** El sistema debe permitir al personal administrativo confirmar la inasistencia de un estudiante sobre una reserva de espacio específica, indicando que constató la ausencia en sitio.
- **FR-003:** El sistema debe validar con la copia local de las reservas de M2 que la reserva existe, corresponde a un espacio, es sancionable, no está cancelada y su inicio pasó hace al menos 30 minutos. No consulta a M2 por REST.
- **FR-004:** El sistema debe identificar al estudiante titular de la reserva por su código institucional.
- **FR-005:** El sistema debe registrar la confirmación con fecha, instante de la constatación, reserva, estudiante, espacio y quién confirma, en estado "Pendiente de acuse".
- **FR-006:** El sistema debe publicar a M2 `NoShowReported` con el identificador de la reserva, el instante de la constatación y una nota opcional (RN-INT-04), sin bloquear la confirmación esperando a M2.
- **FR-007:** El sistema debe crear la sanción por inasistencia solo al recibir el acuse de aceptación de M2 (RN-SAN-03), con el descuento de score correspondiente (RN-SCO-02), y no crearla si M2 rechaza.
- **FR-008:** El sistema debe impedir confirmaciones duplicadas sobre la misma reserva.
- **FR-009:** El sistema debe informar el motivo cuando rechace una confirmación: reserva no identificada, cancelada, antes de los 30 minutos, no es espacio o ya confirmada; y mostrar el motivo cuando M2 la rechace.
- **FR-010:** El sistema debe mantener sin sanción las reservas sin confirmación aceptada.
- **FR-011:** El sistema debe restringir esta acción al personal administrativo autorizado.
- **FR-012:** El sistema debe conservar el evento hacia M2 como pendiente cuando falle su publicación, reintentarlo sin duplicarlo, y descartar por `eventId` los acuses reentregados.
- **FR-013:** El sistema debe permitir al personal administrativo anular en cualquier momento una inasistencia confirmada, con motivo obligatorio, y rechazar la anulación de una ya anulada.
- **FR-014:** El sistema debe, al anular, pasar la confirmación y su sanción a "Anulada", restituir los puntos que la sanción descontó (RN-SCO-12), excluirla de `absenceCount` y del conteo de infracciones, y guardar la auditoría (quién, cuándo y motivo).
- **FR-015:** El sistema debe publicar `NoShowReportVoided` a M2 cuando se anule una inasistencia que M2 había aceptado, conservándolo como pendiente si falla la publicación.
- **FR-016:** El sistema no debe imponer un plazo máximo para confirmar una inasistencia después de los 30 minutos.

## Key Entities
- **Reserva_M2 (copia local):** Reserva de un espacio informada por M2 (titular, espacio, ventana, si es sancionable, si fue cancelada); su dueño es M2.
- **Confirmacion_Inasistencia:** Registro del incumplimiento, con fecha, instante de la constatación, reserva, estudiante, espacio, identificación de quien confirma y estado (Pendiente de acuse, Aceptada, Rechazada por M2 con su motivo, Anulada).
- **Auditoria_Anulacion:** Registro de cada anulación: confirmación, responsable, fecha, motivo y puntos restituidos.
- **Estudiante:** Titular de la reserva sobre la que se confirma la inasistencia.
- **Personal administrativo:** Actor que confirma la inasistencia.
- **Sancion / Score_Confianza:** Consecuencias de la confirmación aceptada (ver `reglas-negocio-m3.md`).
- **Evento_Saliente:** Evento pendiente de publicación hacia M2.

## Integración con Módulos Externos

| Módulo | Tipo de relación | Justificación |
|---|---|---|
| Módulo 2 (Reservas) | **Reactivo (eventos)** | M2 no expone HTTP a otros módulos. M3 recibe las reservas y cancelaciones por Kafka, publica `NoShowReported` y recibe el acuse `NoShowReportAcknowledged`; M2, dueño de las reservas, decide si lo admite (desde el inicio más 10 minutos, reserva no cancelada; M3 solo reporta pasados 30 minutos) y libera el recurso. La sanción resultante no se envía a M2: M2 la consulta a M3 cuando la necesita (RN-SAN-11, RN-INT-06). |

## Success Criteria

### Measurable Outcomes
- **SC-001:** El 100% de las confirmaciones aceptadas por M2 deben crear su sanción por inasistencia.
- **SC-002:** El 0% de las reservas deben tener más de una confirmación de inasistencia.
- **SC-003:** El 100% de los intentos de confirmar sobre reservas canceladas o antes de los 30 minutos deben ser rechazados.
- **SC-004:** El 0% de las reservas deben sancionarse sin una confirmación aceptada por M2.
- **SC-005:** El sistema debe registrar la confirmación en menos de 2 segundos; no depende de que M2 esté disponible.
- **SC-006:** El listado de pendientes debe cargar en menos de 3 segundos.
- **SC-007:** El 100% de las confirmaciones registradas deben tener su evento publicado o pendiente de reintento hacia M2.
- **SC-008:** El 100% de las anulaciones deben restituir los puntos descontados, quedar en la auditoría y excluir la inasistencia del conteo; el 0% de las confirmaciones ya anuladas deben poder anularse de nuevo.
