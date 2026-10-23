# Feature Specification: Registrar Check-out (Devolución)

**Created:** 2026-10-17
**Updated:** 2026-10-22 (integración alineada con los contratos de M1 y M2; el retraso se penaliza con descuento diario de score)

> Las reglas de sanción, score y cobro que este caso de uso dispara están en `reglas-negocio-m3.md` (RN-SAN, RN-SCO, RN-COB). Las reglas de integración con M1 y M2 están en la sección 5 del mismo documento (RN-INT).

## User Scenarios & Testing

### User Story 1 - Registrar la devolución de un recurso o espacio en uso (Priority: P1)
Como estudiante, quiero registrar el check-out de un recurso o espacio que tengo en uso, para finalizar su utilización y dejar constancia de la devolución.

**Why this priority:** Es la funcionalidad central del módulo: sin check-out no se sabe cuándo terminó el uso, si fue a tiempo ni si el recurso volvió en buen estado, y de ello dependen el score, las sanciones y los cobros.

**Independent Test:** Con un estudiante que tenga una utilización activa (una reserva que M2 informó y que ya inició), registrar el check-out y verificar que se guarda la fecha de devolución, que la utilización queda finalizada, que se publica la devolución a M1 y que se publica el check-out a M2.

**Acceptance Scenarios:**

**Scenario: Check-out a tiempo y sin novedades**
- **Given** un estudiante con una utilización activa cuya fecha límite aún no ha pasado.
- **When** registra el check-out sin reportar daños.
- **Then** M3 registra el check-out con la fecha de devolución, finaliza la utilización, publica a M1 `reserva.finalizada` (RN-INT-05), publica a M2 `CheckOutRegistered` (RN-INT-03) y suma al score los puntos por entrega a tiempo (RN-SCO-03).

**Scenario: Check-out con retraso**
- **Given** una utilización activa de un activo cuya fecha límite ya pasó.
- **When** el estudiante registra el check-out.
- **Then** M3 registra el check-out con los minutos de retraso, finaliza la utilización, publica la devolución a M1 y el check-out a M2, y detiene el descuento diario por retraso (RN-SCO-11). El check-out no crea ninguna sanción (RN-SAN-02) ni descuenta puntos por sí mismo, y los puntos que ya se habían descontado por los días de retraso no se devuelven. No se otorgan puntos por la entrega.

**Scenario: Check-out con reporte de daño**
- **Given** un estudiante que está registrando el check-out de un recurso o espacio.
- **When** indica que presenta un daño.
- **Then** M3 registra el check-out normalmente y dispara (`«extend»`) *Reportar novedad técnica*. A M1 se le publica `reporte.danos` en lugar de `reserva.finalizada`, así M1 no marca el recurso como disponible (RN-INT-05). A M2 se le publica `CheckOutRegistered` sin ningún dato del daño si es un activo, o con el veredicto `REQUIERE_MANTENIMIENTO` si es un espacio (RN-INT-03). Aunque la entrega haya sido a tiempo, no se otorgan puntos de score (RN-SCO-04), porque la entrega "óptima" exige que no haya daño.

### User Story 2 - Rechazo del check-out por estado inválido (Priority: P2)
Como sistema, quiero rechazar un check-out cuando no exista una utilización activa válida, para evitar registros duplicados o inconsistentes.

**Why this priority:** Es el escenario de fallo de la funcionalidad principal; sin estas validaciones se podría finalizar dos veces la misma utilización u otorgar puntos o sanciones duplicadas.

**Independent Test:** Intentar un check-out sobre una utilización ya finalizada, sobre un estudiante sin utilización activa, sobre una reserva cancelada, y enviar dos solicitudes simultáneas para la misma utilización.

**Acceptance Scenarios:**

**Scenario: Utilización ya finalizada**
- **Given** una utilización finalizada por un check-out previo.
- **When** se intenta registrar otro check-out sobre ella.
- **Then** el sistema rechaza la operación e informa que la utilización ya fue finalizada.

**Scenario: Estudiante sin utilización activa**
- **Given** un estudiante sin ninguna reserva o préstamo en uso entre las que M2 informó (por ejemplo, la reserva no ha iniciado, fue cancelada o no ha llegado a M3).
- **When** intenta registrar un check-out.
- **Then** el sistema rechaza la operación e informa que no existe una utilización activa asociada.

**Scenario: Solicitudes simultáneas**
- **Given** dos solicitudes de check-out enviadas casi al mismo tiempo para la misma utilización.
- **When** el sistema las procesa.
- **Then** registra un único check-out exitoso y rechaza la segunda informando que la utilización ya fue finalizada.

### User Story 3 - Devolución por un estudiante sancionado o bloqueado (Priority: P2)
Como estudiante con una sanción vigente o con el usuario bloqueado, quiero poder registrar la devolución de lo que tengo en uso, para regularizar mi situación y no agravarla.

**Why this priority:** Si la sanción o el bloqueo impidieran devolver, un estudiante moroso no podría regularizarse y el recurso seguiría fuera de circulación.

**Independent Test:** Con un estudiante sancionado y otro bloqueado (score 0), registrar el check-out y verificar que se acepta en ambos casos.

**Acceptance Scenarios:**

**Scenario: Check-out con sanción vigente o usuario bloqueado**
- **Given** un estudiante con una sanción vigente o con el score en 0 (bloqueado).
- **When** registra el check-out de un recurso que tiene en uso.
- **Then** el sistema acepta el check-out con normalidad; las sanciones y el bloqueo solo restringen nuevas reservas, no devoluciones (RN-SAN-05, RN-BLO-02).

### User Story 4 - Descontar puntos mientras un préstamo sigue sin devolverse (Priority: P2)
Como sistema, quiero descontar puntos de score por cada día que un préstamo de un activo sigue vencido, para que no devolver tenga consecuencia aunque el estudiante nunca registre el check-out.

**Why this priority:** Sin esto, un estudiante que nunca devuelve el recurso no recibiría ninguna consecuencia, porque el retraso ya no genera una sanción al registrar el check-out.

**Independent Test:** Con un préstamo vencido y un estudiante con score 50, simular el paso de los días y verificar el descuento de cada día, que el día 7 el score queda en 0 y que la devolución detiene el descuento.

**Acceptance Scenarios:**

**Scenario: Descuento diario**
- **Given** un préstamo de un activo cuya fecha límite pasó hace un día completo, sin check-out, de un estudiante que tenía 50 puntos al vencer.
- **When** corre el proceso diario.
- **Then** M3 descuenta la séptima parte del score que tenía al vencer (8 puntos), redondeada hacia arriba, y registra el cambio en el historial con tipo REDUCCION_RETRASO (RN-SCO-11).

**Scenario: Día 7**
- **Given** un préstamo con seis días de descuento aplicados.
- **When** corre el proceso diario el día 7.
- **Then** M3 descuenta lo que falte para completar el score que tenía al vencer, el score llega a 0 y el estudiante queda bloqueado (RN-BLO-01). M2 declara perdido el préstamo y M3 genera el cobro por pérdida (RN-SAN-12, RN-COB-12).

**Scenario: Devolución a mitad del retraso**
- **Given** un préstamo con tres días de descuento aplicados.
- **When** el estudiante registra el check-out.
- **Then** el descuento se detiene, los puntos ya descontados no se devuelven y el check-out no otorga los puntos de entrega a tiempo.

**Scenario: Retraso de menos de un día**
- **Given** un préstamo devuelto pocas horas después de su fecha límite.
- **When** el estudiante registra el check-out.
- **Then** el check-out se clasifica CON_RETRASO, no se descuentan puntos y no se otorgan los puntos de entrega a tiempo.

**Scenario: Varios préstamos vencidos**
- **Given** un estudiante con dos préstamos de activos vencidos y sin devolver.
- **When** corre el proceso diario.
- **Then** M3 aplica un solo descuento ese día, el que corresponde al préstamo más antiguo, y no suma el del otro (RN-SCO-11).

### User Story 5 - Conciliar el check-out con la respuesta de M2 (Priority: P3)
Como sistema, quiero procesar el acuse que M2 envía por cada check-out informado, para saber si M2 lo aceptó y corregir los casos que no.

**Why this priority:** El check-out ya está registrado y sus consecuencias aplicadas; el acuse sirve para detectar inconsistencias entre módulos y para reenviar la revisión de un espacio que M2 aún no admite.

**Independent Test:** Simular un acuse aceptado, un rechazo `SLOT_NOT_FINISHED` con su hora `acceptedFrom` y un rechazo de otro tipo, y verificar el estado final del check-out en cada caso.

**Acceptance Scenarios:**

**Scenario: Acuse aceptado**
- **Given** un check-out informado a M2.
- **When** M2 acusa que lo aceptó.
- **Then** el check-out queda con acuse "Aceptado". Si es un activo, M3 conserva la fecha límite (`dueAt`) que M2 informa en el acuse.

**Scenario: Revisión de espacio antes de que termine la franja**
- **Given** un check-out de un espacio informado a M2 antes de que termine la franja reservada.
- **When** M2 lo rechaza con `SLOT_NOT_FINISHED` y la hora `acceptedFrom`.
- **Then** el check-out se mantiene registrado y M3 vuelve a enviar el evento cuando llegue esa hora, sin duplicar sus consecuencias.

**Scenario: Rechazo de otro tipo**
- **Given** un check-out informado a M2.
- **When** M2 lo rechaza por otro motivo (por ejemplo, un activo con datos de daño o una reserva ya cerrada).
- **Then** el check-out se mantiene registrado, queda con acuse "Rechazado" y su motivo, y se marca para revisión del personal administrativo; no se revierte.

## Edge Cases

- **Reserva no recibida todavía de M2:** el sistema debe rechazar el check-out e invitar a reintentar más tarde; no debe registrarlo con datos supuestos de la reserva o préstamo.
- **Reserva sin fecha límite o sin titular en la copia local:** el sistema debe rechazar el check-out e informar que no fue posible validar la utilización.
- **Reserva cancelada:** el sistema debe rechazar el check-out e informar que la reserva fue cancelada (RN-INT-01).
- **Reserva académica o no sancionable:** no admite check-out en M3.
- **Recurso o espacio no identificable:** el sistema debe rechazar el check-out e informar que no fue posible identificar el recurso asociado.
- **Fecha límite ya pasada hace mucho tiempo:** el retraso se calcula contra la fecha límite original que informó M2. A los 7 días el descuento ya llegó a todo el score y M2 declara perdido el préstamo (`LoanDeclaredLost`, RN-SAN-12); un check-out posterior se registra con normalidad, no descuenta más y el personal administrativo decide si anula el cobro por pérdida.
- **Varios préstamos vencidos a la vez:** el estudiante recibe un solo descuento por día, el del préstamo vencido más antiguo que siga sin devolverse; los demás no descuentan adicionalmente. Si el más antiguo se devuelve, rige el siguiente (RN-SCO-11).
- **Falta un parámetro (por ejemplo, los días de retraso):** el check-out se registra igual, no se descuentan puntos con valores supuestos y la inconsistencia queda marcada para revisión.
- **Reserva de espacio:** en un espacio no se mide retraso; el check-out de un espacio sin daño es A_TIEMPO.
- **Falla en un paso posterior (score, sanción o novedad técnica):** el check-out se mantiene registrado y la inconsistencia queda marcada para reintento; nunca se revierte la devolución.
- **Falla al publicar a M1 o a M2:** el check-out se conserva junto con el evento pendiente para reintento; publicar al broker no confirma que M1 o M2 lo hayan procesado.
- **Evento duplicado de M2 (reentrega):** la copia local y los acuses se procesan una sola vez por `eventId` (RN-INT-02).
- **Daño indicado sin descripción:** se solicita completar la descripción antes de confirmar el check-out; si el estudiante desmarca el daño, el check-out continúa sin reporte.
- **Error al registrar el check-out:** el sistema debe informar que no pudo registrarse y evitar dejar un registro incompleto o la utilización en estado ambiguo.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe permitir al estudiante registrar el check-out de un recurso o espacio que tenga en uso.
- **FR-002:** El sistema debe identificar la utilización activa del estudiante a partir de las reservas que M2 informó (copia local, RN-INT-01) y obtener su fecha límite: la fecha de vencimiento del préstamo en un activo, o el fin de la ocupación en un espacio. No consulta a M2 por REST.
- **FR-003:** El sistema debe registrar el check-out con la fecha y hora de devolución generadas por el servidor, no por el cliente.
- **FR-004:** El sistema debe finalizar la utilización cuando el check-out se registre correctamente.
- **FR-005:** El sistema debe clasificar el check-out de un activo como A_TIEMPO o CON_RETRASO comparando la fecha de devolución con la fecha límite del préstamo (sin tolerancia), y registrar los minutos de retraso cuando aplique. El check-out de un espacio no mide retraso.
- **FR-006:** El sistema debe sumar al score los puntos por entrega a tiempo cuando el check-out sea A_TIEMPO y no tenga novedad técnica (RN-SCO-03).
- **FR-007:** El sistema debe detener el descuento diario por retraso cuando se registre el check-out de un préstamo vencido, sin devolver los puntos ya descontados y sin crear sanción (RN-SAN-02, RN-SCO-11).
- **FR-008:** El sistema debe permitir reportar un daño durante el check-out, disparando (`«extend»`) *Reportar novedad técnica* sin bloquear el registro del check-out.
- **FR-009:** El sistema debe publicar a M1 un único evento por check-out: `reserva.finalizada` si no hay novedad técnica o `reporte.danos` si la hay, de modo que M1 no registre el recurso como disponible antes de procesar el daño (RN-INT-05).
- **FR-010:** El sistema debe impedir que una utilización finalizada se vuelva a finalizar con otro check-out.
- **FR-011:** El sistema debe garantizar que, ante solicitudes simultáneas sobre la misma utilización, solo una se registre exitosamente.
- **FR-012:** El sistema debe permitir el check-out a estudiantes con sanciones vigentes o bloqueados.
- **FR-013:** El sistema debe conservar como pendientes los eventos hacia M1 y M2 cuando falle su publicación, y reintentarlos sin duplicarlos.
- **FR-014:** El sistema debe informar al estudiante el resultado del check-out (a tiempo o con retraso) y la consecuencia aplicada.
- **FR-015:** El sistema debe informar el motivo cuando rechace el check-out: sin utilización activa, ya finalizada, reserva cancelada, recurso no identificado o reserva aún no recibida de M2.
- **FR-016:** El sistema debe publicar a M2 `CheckOutRegistered` con el identificador de la reserva y el instante real de la devolución; en un activo sin ningún dato de daño, y en un espacio con el veredicto `SIN_NOVEDAD` o `REQUIERE_MANTENIMIENTO` (RN-INT-03).
- **FR-017:** El sistema debe procesar el acuse de M2: reenviar la revisión de un espacio a la hora `acceptedFrom` cuando M2 responda `SLOT_NOT_FINISHED`, y marcar para revisión cualquier otro rechazo, sin revertir el check-out.
- **FR-018:** El sistema debe ejecutar un proceso diario que, para cada préstamo de un activo vencido y sin check-out, descuente puntos por cada día completo de retraso hasta el día 7 (RN-SCO-11), registre cada descuento en el historial con tipo REDUCCION_RETRASO y lo aplique una sola vez por día aunque el proceso se repita. Si el estudiante tiene varios préstamos vencidos, aplica un solo descuento por día, el del préstamo vencido más antiguo.
- **FR-019:** El sistema debe llevar el score a 0 el día 7 de un préstamo sin devolver, de modo que el estudiante quede bloqueado (RN-BLO-01).

## Key Entities
- **Estudiante:** Persona que tiene el recurso o espacio en uso y realiza el check-out.
- **Reserva_M2 (copia local):** Reserva informada por M2: identificador, origen, titular (código institucional), recurso (categoría, tipo, ubicación), ocupación y, en activos, retiro y fecha límite del préstamo. Su dueño es M2; M3 la actualiza con cada evento (ver RN-INT-01).
- **Utilización:** Período en que un recurso o espacio está bajo responsabilidad de un estudiante, derivado de una reserva de M2. Contiene fecha límite y estado (Activa, Finalizada).
- **Descuento_Retraso:** Registro de cada descuento diario de un préstamo vencido (préstamo, día, puntos), que garantiza que cada día se descuente una sola vez.
- **Check_out:** Registro de la devolución, con fecha, resultado (A_TIEMPO / CON_RETRASO), minutos de retraso, indicador de novedad técnica, estado del acuse de M2 (Pendiente, Aceptado, Rechazado) y relación con estudiante, recurso y utilización.
- **Recurso / Espacio:** Activo cuyo inventario pertenece a M1; M3 conserva una instantánea de sus datos descriptivos (placa o ID, tipo, ubicación) tomada de la reserva que informó M2.
- **Evento_Saliente:** Evento pendiente de publicación hacia M1 o M2, con estado y reintentos.
- **Sancion / Score_Confianza:** Consecuencias que este caso de uso produce (ver `reglas-negocio-m3.md`).

## Integración con Módulos Externos

| Módulo | Tipo de relación | Justificación |
|---|---|---|
| Módulo 1 (Activos y Espacios) | **Reactivo (evento asíncrono)** | M3 informa la devolución (`reserva.finalizada`) o el daño (`reporte.danos`) en `unimag.m3.uso-novedades.v1`; M1, dueño del inventario, decide y aplica el estado. El registro del check-out no debe bloquearse esperando a M1. Los payloads están por acordar con M1. |
| Módulo 2 (Reservas) | **Reactivo (eventos)** | M2 no expone HTTP a otros módulos. M3 recibe las reservas y cancelaciones por Kafka (`ReservationRecordCreated`, `ReservationCancelled`), publica `CheckOutRegistered` y recibe el acuse `CheckOutAcknowledged`. La sanción que resulte no se envía a M2: M2 la consulta a M3 cuando la necesita (RN-SAN-11, RN-INT-06). |

## Success Criteria

### Measurable Outcomes
- **SC-001:** El 100% de los check-outs válidos deben quedar registrados con fecha de devolución y finalizar la utilización correspondiente.
- **SC-002:** El 100% de los check-outs a tiempo y sin novedad deben sumar los puntos de score, y el 0% de los check-outs con retraso deben otorgarlos.
- **SC-003:** El 100% de los check-outs con reporte de daño deben disparar *Reportar novedad técnica* sin impedir el registro del check-out.
- **SC-004:** El 0% de las utilizaciones deben tener más de un check-out exitoso, incluso ante solicitudes simultáneas.
- **SC-005:** El 100% de los check-outs registrados deben tener su evento hacia M1 y hacia M2 publicado o pendiente de reintento; ninguno se pierde.
- **SC-006:** El sistema debe registrar el check-out en menos de 2 segundos desde que recibe la solicitud; no depende de que M1 o M2 estén disponibles.
- **SC-007:** El 100% de los check-outs de estudiantes sancionados o bloqueados deben aceptarse.
- **SC-008:** El 0% de los check-outs con daño deben publicarse a M1 como `reserva.finalizada`.
- **SC-009:** El 100% de los eventos de M2 reentregados deben procesarse una sola vez.
- **SC-010:** El 100% de los préstamos de activos que lleguen al día 7 sin check-out deben dejar al estudiante con score 0, y el 0% deben descontar más de una vez por día.
