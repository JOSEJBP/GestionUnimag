# Reglas de Negocio — Módulo 3: Control de Uso, Sanciones y Analítica

**Created:** 2026-10-17
**Updated:** 2026-10-22 (integración con M1 y M2; retraso por descuento diario; decisiones del equipo sobre sanciones, cobros y analítica)

## Propósito y alcance

Este documento reúne las reglas del módulo que **no son casos de uso del diagrama**: sanciones, score de confianza, bloqueo de usuario, cobros y las reglas de integración con M1 y M2. Son consecuencias que otros casos de uso producen, y se documentan una sola vez aquí. Los specs de los casos de uso las referencian por su código (por ejemplo, RN-SAN-02).

Casos de uso del diagrama: *Registrar check-out*, *Reportar novedad técnica*, *Confirmar no asistencia*, *Desbloquear usuario*, *Consultar mi cumplimiento* y *Consultar analítica de cumplimiento*.

| Regla | La dispara |
|---|---|
| Sanciones (RN-SAN) | El acuse de M2 a *Confirmar no asistencia* |
| Score de confianza (RN-SCO) | *Registrar check-out* (entrega a tiempo), el descuento diario por retraso, cada sanción nueva, el proceso diario de recuperación y *Desbloquear usuario* |
| Bloqueo (RN-BLO) | El score llega a 0 |
| Cobros (RN-COB) | *Reportar novedad técnica* (daño) y la pérdida de un préstamo declarada por M2 |
| Integración (RN-INT) | Todos los casos de uso que hablan con M1 o M2, y las consultas que M1 y M2 hacen a M3 |

## Parámetros configurables

Ningún valor va fijo en el código; todos se leen de la configuración del sistema (`Configuracion_Sistema`).

| Parámetro | Valor por defecto |
|---|---|
| Score inicial y máximo | 50 |
| Puntos por entrega a tiempo y sin novedad técnica | +2 |
| Puntos descontados por cada sanción | −5 |
| Días de retraso de un préstamo hasta que el descuento llega al total del score | 7 (coincide con el umbral de pérdida de M2) |
| Duración de la sanción por inasistencia | 7 días ("1 semana" según el documento del proyecto) |
| Minutos desde el inicio de la reserva para considerar una inasistencia | 30 |
| Días sin infracciones para recuperar puntos | 30 |
| Puntos de recuperación | +5 |
| Score tras un desbloqueo manual | 5 |
| Porcentaje del valor del recurso que se cobra por un préstamo perdido | 50% |

---

## 1. Sanciones

- **RN-SAN-01 — Tipo y alcance.** Existe un solo tipo de sanción: **INASISTENCIA**, que restringe las reservas nuevas de espacios (alcance `ESPACIOS` hacia M2). El retraso en un préstamo no genera una sanción: se penaliza con el descuento diario del score (RN-SAN-02, RN-SCO-11). El bloqueo por score 0 tampoco es una sanción: se deriva del score (RN-BLO-01).
- **RN-SAN-02 — Retraso.** No existe una sanción por mora. Un préstamo de un activo que no se devuelve en su fecha límite se penaliza únicamente con el descuento diario de puntos de RN-SCO-11, que llega al total del score el día 7 y deja al estudiante bloqueado.
- **RN-SAN-03 — Inasistencia.** Un estudiante que no se presenta dentro de los 30 minutos siguientes a la hora de inicio de su reserva de un espacio incumple la reserva. La sanción solo se crea cuando el personal administrativo confirma la inasistencia y M2 acusa **aceptado** el reporte (RN-INT-04), no en el momento de la confirmación. Nace en estado "Vigente", con la duración configurada (7 días) contados desde la fecha de registro de la ausencia que informa M2. Si M2 rechaza el reporte, no se crea sanción. No hay plazo máximo para confirmar: la única condición de tiempo es que hayan pasado los 30 minutos.
- **RN-SAN-04 — Una sanción por reserva.** Una reserva genera como máximo una sanción. Si M2 acusa un reporte repetido (`duplicate`), no se crea otra.
- **RN-SAN-05 — Solo restringen reservas nuevas.** Una sanción nunca impide registrar devoluciones (check-out), reportar daños ni consultar.
- **RN-SAN-06 — Estados y levantamiento.** Los estados son "Vigente", "Cumplida" (terminó por fecha), "Levantada" (terminó antes de tiempo por un desbloqueo manual) y "Anulada" (la inasistencia que la originó se anuló por error, RN-SAN-14). Un proceso programado interno marca como "Cumplida" toda sanción cuya fecha de fin ya pasó, dentro de los 5 minutos siguientes. Solo las sanciones "Vigente" se informan a M2 (RN-INT-06).
- **RN-SAN-07 — Sanciones simultáneas.** Varias sanciones pueden estar vigentes a la vez y corren en paralelo: cada una empieza cuando se crea y termina en su propia fecha. La restricción de reservas dura mientras haya al menos una vigente.
- **RN-SAN-08 — Cálculo de días.** Se cuentan días calendario en la zona horaria de la universidad (America/Bogota). La fecha de fin que se informa a M2 es inclusiva (último día en que la sanción rige).
- **RN-SAN-09 — Parámetro faltante.** Si falta un parámetro (por ejemplo, la duración de la inasistencia) o el estudiante no es identificable, no se crea la sanción con valores supuestos: se informa el motivo y la inconsistencia queda registrada para revisión.
- **RN-SAN-10 — Efecto en el score.** Cada sanción nueva reduce el score una sola vez (RN-SCO-02).
- **RN-SAN-11 — Cómo se enteran los demás.** M3 no publica eventos de sanción. M2 consulta las restricciones activas de un estudiante a M3 cada vez que lo necesita, mediante la consulta REST de RN-INT-06, y niega la reserva con el mensaje "sanción activa". Por eso M3 debe responder rápido y con datos al día; M2 no guarda copia.
- **RN-SAN-12 — Préstamo vencido que no se devuelve.** Mientras el activo no se devuelve, el estudiante pierde puntos cada día (RN-SCO-11). A los 7 días M2 declara perdido el préstamo y publica `LoanDeclaredLost`; M3 lo recibe, lo registra como expediente (estudiante, recurso, placa, días de mora) y genera un cobro por pérdida (RN-COB-12). Si el estudiante devuelve el recurso después, el check-out se registra con normalidad y el personal administrativo decide si anula el cobro.
- **RN-SAN-13 — Fallos posteriores.** Si falla un paso posterior a crear la sanción (por ejemplo, el score), la sanción se mantiene y la inconsistencia queda marcada para reintento; nunca se revierte.
- **RN-SAN-14 — Anulación de una inasistencia.** El personal administrativo puede anular en cualquier momento una inasistencia ya confirmada, con motivo obligatorio y registro de auditoría (quién, cuándo y por qué). La confirmación pasa a "Anulada"; su sanción también pasa a "Anulada", esté vigente o ya cumplida; los puntos descontados por ella se restituyen (RN-SCO-12); la inasistencia deja de contar en `absenceCount` y como infracción; y M3 publica `NoShowReportVoided` a M2 (RN-INT-04). Una confirmación ya anulada no se puede anular de nuevo.

## 2. Score de confianza

- **RN-SCO-01 — Valor y rango.** El score inicial es igual al máximo (50) y se crea al primer cambio si el estudiante no lo tiene. Siempre está entre 0 y el máximo.
- **RN-SCO-02 — Reducción por sanción.** Cada sanción nueva descuenta los puntos configurados (5), **una sola vez por sanción**. Levantar o cumplir la sanción no devuelve puntos. Si el score es menor que el descuento, queda en 0 y el historial registra la variación efectiva.
- **RN-SCO-03 — Aumento por entrega óptima.** Un check-out a tiempo (devolución ≤ fecha límite) y sin novedad técnica suma los puntos configurados (2), sin superar el máximo. Si el score ya está en el máximo, no cambia, pero el historial registra el evento. En un espacio no se mide retraso (RN-SCO-11): un check-out de espacio sin daño suma los puntos.
- **RN-SCO-04 — Sin puntos por retraso o daño.** Un check-out con retraso o con daño reportado no otorga puntos. El retraso baja el score por el descuento diario (RN-SCO-11), no por el check-out. El daño genera un cobro (RN-COB), no una reducción de score.
- **RN-SCO-05 — Recuperación por buen comportamiento.** Cada 30 días calendario sin infracciones, el score aumenta 5 puntos hasta el máximo. Los 30 días se cuentan desde la última infracción o desde la última recuperación, lo que sea más reciente: una nueva infracción reinicia el conteo. Son infracciones: una sanción nueva (una sanción anulada deja de contar), el primer descuento por retraso de un préstamo (RN-SCO-11) y un reporte de daño atribuido al estudiante. La recuperación aplica a todos los estudiantes con score menor al máximo y la ejecuta un proceso programado interno.
- **RN-SCO-06 — Historial.** Cada cambio queda registrado con fecha, tipo (AUMENTO_CHECKOUT, REDUCCION_SANCION, REDUCCION_RETRASO, RECUPERACION, DESBLOQUEO_MANUAL o RESTITUCION_ANULACION), puntos efectivos, motivo, score antes y después, y origen (check-out, sanción, préstamo, proceso o desbloqueo).
- **RN-SCO-07 — Idempotencia y concurrencia.** Cada origen cambia el score una sola vez, aunque se dispare de nuevo (incluida la reentrega de un evento de M2). Los cambios simultáneos del mismo estudiante se aplican sin pérdida y respetando los límites.
- **RN-SCO-08 — Cambio de parámetros.** Un cambio en los puntos o en el máximo solo afecta a los cambios futuros; el historial no se recalcula.
- **RN-SCO-09 — Fallos.** Si falla la actualización del score, el check-out o la sanción que la originó se mantienen y la inconsistencia queda marcada para reintento.
- **RN-SCO-10 — Cancelaciones.** Las cancelaciones que M2 informa no cambian el score, sin importar con cuánta anticipación se hagan.
- **RN-SCO-11 — Descuento por retraso en préstamos de activos.** Cuando un préstamo de un activo pasa de su fecha límite sin devolverse, un proceso programado interno descuenta puntos por cada día completo de retraso, hasta el día 7. El descuento de cada día es la séptima parte del score que el estudiante tenía cuando venció el préstamo, redondeada hacia arriba, y el día 7 se descuenta lo que falte para completar ese score; así el descuento total llega al score completo el día 7 y el estudiante queda en 0 (bloqueado). Nunca baja de 0. Un retraso menor a un día no descuenta puntos, aunque el check-out se clasifique como con retraso. Al devolver, el descuento se detiene y los puntos ya descontados no se devuelven. Un estudiante recibe un solo descuento por día, aunque tenga varios préstamos vencidos: es el que corresponde al préstamo vencido más antiguo que siga sin devolverse, según los días que ese préstamo lleva vencido; los demás préstamos no descuentan adicionalmente. Si el más antiguo se devuelve, rige el siguiente más antiguo con sus propios días de retraso. Solo aplica a préstamos de activos; en espacios no se mide retraso.

- **RN-SCO-12 — Restitución por anulación.** Cuando se anula una inasistencia (RN-SAN-14), el score recupera los puntos efectivos que esa sanción descontó, sin superar el máximo, y el historial lo registra con tipo RESTITUCION_ANULACION. Es la única situación en la que se devuelven puntos descontados por una sanción; levantarla por un desbloqueo o cumplirla no los devuelve (RN-SCO-02). Si el estudiante estaba bloqueado por ese descuento, deja de estarlo cuando el score supera 0.

## 3. Bloqueo de usuario

- **RN-BLO-01 — Definición.** Un estudiante está **bloqueado mientras su score sea 0**. No es un estado independiente: se deriva del score, así que no hay dos fuentes de verdad.
- **RN-BLO-02 — Efecto.** El estudiante bloqueado no puede hacer reservas nuevas de ningún tipo. Sí puede registrar check-outs, reportar daños al devolver y consultar su cumplimiento.
- **RN-BLO-03 — Salida automática.** Al recuperar puntos (RN-SCO-05) el score pasa de 0 a 5 y el bloqueo termina.
- **RN-BLO-04 — Salida manual.** El personal administrativo puede desbloquear a un estudiante con el caso de uso *Desbloquear usuario*: el score queda en el valor configurado (5) y se levantan sus sanciones vigentes (RN-BLO-05), con motivo obligatorio y registro de auditoría (quién, cuándo y por qué). No hay límite de desbloqueos por estudiante; cada uno queda en la auditoría.
- **RN-BLO-05 — Alcance del desbloqueo.** El desbloqueo manual levanta también todas las sanciones vigentes del estudiante (pasan a "Levantada" con el motivo del desbloqueo), porque, si siguieran vigentes, el estudiante no podría reservar espacios. El levantamiento no modifica el score. La salida automática por recuperación de puntos (RN-BLO-03) no levanta sanciones: estas siguen hasta su fecha de fin. Si el estudiante aún tiene un préstamo vencido, el descuento diario (RN-SCO-11) continúa y puede dejarlo otra vez en 0.
- **RN-BLO-06 — Sanciones con el score en 0.** Una sanción nueva se aplica y se registra con normalidad; su reducción efectiva de score es 0.
- **RN-BLO-07 — Cómo se entera M2.** M3 no publica eventos de bloqueo ni de desbloqueo. La consulta REST de RN-INT-06 incluye el bloqueo mientras el score sea 0 y deja de incluirlo en la consulta siguiente al desbloqueo.

## 4. Cobros

- **RN-COB-01 — Generación automática por daño.** Cuando *Reportar novedad técnica* registra un reporte de daño, se genera automáticamente un cobro por la reparación o reposición del recurso o espacio afectado.
- **RN-COB-02 — Monto.** Se toma del catálogo de costos del tipo de recurso, espacio o equipamiento, en pesos colombianos (COP). Un costo en cero, negativo o inexistente cuenta como "no configurado".
- **RN-COB-03 — Responsable.** Es el estudiante de la utilización asociada al reporte.
- **RN-COB-04 — Resultado de la generación.** El reporte guarda el resultado: **Generado**, **Sin costo** (no hay costo válido; queda para revisión), **Sin responsable** (no hay estudiante identificable; queda para revisión) o **Error** (falló la generación; queda para reintento). En todos los casos el reporte se conserva.
- **RN-COB-05 — Un cobro por reporte, con monto congelado.** No se crea un segundo cobro para el mismo reporte. El monto no cambia aunque el catálogo se modifique después.
- **RN-COB-06 — Estados y gestión.** El cobro nace "Pendiente", con fecha de generación, motivo y monto. El personal administrativo registra el pago ("Pagado") o la anulación ("Anulado", con motivo obligatorio), y cada cambio queda en una auditoría (quién, cuándo y por qué). El recaudo del dinero ocurre fuera de M3.
- **RN-COB-07 — Daño en espacios.** El costo se toma de la configuración del tipo de espacio o equipamiento; si no existe, aplica "Sin costo".
- **RN-COB-08 — Visibilidad.** El estudiante ve sus cobros en *Consultar mi cumplimiento*, y el personal administrativo ve el detalle por estudiante y los datos generales en *Consultar analítica de cumplimiento*.
- **RN-COB-09 — Daño preexistente.** Sin inspección previa al uso, el cobro recae en el responsable de la última utilización. La vía para corregirlo es que el personal administrativo anule el cobro con motivo (RN-COB-06).
- **RN-COB-10 — Origen del costo.** El costo sale de un catálogo propio de M3, que mantiene el personal administrativo. M3 no toma el valor del recurso de M1.
- **RN-COB-11 — Ni M1 ni M2 cobran.** M1 no calcula cobros por daños y M2 no acepta ningún dato de daños en el check-out; el cobro es responsabilidad exclusiva de M3.
- **RN-COB-12 — Cobro por préstamo perdido.** Al recibir `LoanDeclaredLost` (día 7), M3 genera un cobro "Pendiente" al estudiante titular, por un porcentaje del valor del recurso en el catálogo (parámetro, 50% por defecto), con motivo "Préstamo perdido". Un solo cobro por préstamo; el resultado (Generado, Sin costo, Error) queda en el expediente de pérdida.

## 5. Integración con M1 y M2

Estas reglas salen de los contratos publicados por cada equipo (repositorios de M1 y de M2). Donde el contrato aún está "por acordar" o en blanco, se indica.

- **RN-INT-01 — Copia local de las reservas.** M2 no expone HTTP a otros módulos; todo llega por Kafka. M3 consume `ReservationRecordCreated` (tópico `module2.reservation.record.v1`) y `ReservationCancelled` (tópico `module2.reservation.cancellation.v1`) y mantiene una copia local de cada reserva: identificador, origen, si es sancionable, recurso (id, nombre, categoría ACTIVO/ESPACIO, tipo, ubicación), titular (código institucional, nombre), ocupación (inicio y fin) y, para activos, el préstamo (retiro y fecha límite). Las reservas con origen ACADEMICO o no sancionables no generan check-out ni inasistencia. M3 no consulta a M2 por REST.
- **RN-INT-02 — Idempotencia y orden.** La entrega de M2 es *at-least-once*: M3 descarta por `eventId` los eventos ya procesados (bandeja de entrada) y confirma el offset solo después de guardar. El orden entre reservas distintas no está garantizado; para ordenar cancelaciones se usa `cancelledAt` (y `occurredAt` para el resto), nunca el orden de llegada. Un evento mal formado o sobre una reserva desconocida va al DLT. Lo que M3 publica sale por una bandeja de salida (outbox) en la misma transacción que el dato de negocio, con reintentos y sin duplicados.
- **RN-INT-03 — Check-out informado a M2.** Tras registrar un check-out, M3 publica `CheckOutRegistered` en `module3.reservation.check-out.v1`, con el `reservationId` y el instante de la devolución (`data.occurredAt`, el instante real, no el del envoltorio). Para un **activo** no se envía ningún campo de daño: M2 lo rechaza. Para un **espacio** se envía `verdict`: `REQUIERE_MANTENIMIENTO` si hubo daño y `SIN_NOVEDAD` si no. M2 responde con `CheckOutAcknowledged` (`module2.reservation.check-out-ack.v1`, "por acordar"): aceptado, o rechazado con un motivo. El check-out y el score de M3 **no esperan** el acuse (es un hecho ya ocurrido); un rechazo marca el check-out para revisión, salvo `SLOT_NOT_FINISHED` (M2 solo admite la revisión de un espacio después de terminada la franja), caso en que M3 reenvía el evento cuando llegue la hora `acceptedFrom` que M2 informa. El acuse de un activo trae `dueAt`, la fecha límite del préstamo.
- **RN-INT-04 — Inasistencia reportada a M2.** Al confirmar la inasistencia (a partir de los 30 minutos desde el inicio de la reserva, RN-SAN-03), M3 publica `NoShowReported` en `module3.reservation.no-show.v1`, con `reservationId`, `verifiedAt` (el instante en que el personal constató la ausencia, no el de envío), `verifiedBy` (`MODULO_3`) y una nota opcional. M2 solo lo admite desde el inicio de la reserva más 10 minutos, por lo que los 30 minutos de M3 siempre lo cumplen. M2 responde con `NoShowReportAcknowledged` (`module2.reservation.no-show-ack.v1`, "por acordar"). Aceptado: M2 libera el recurso y M3 crea la sanción (RN-SAN-03). Rechazado: `TOO_EARLY` (trae `acceptedFrom`; se puede reintentar a partir de esa hora), `RESERVATION_CANCELLED` o `ALREADY_RETURNED`; no se crea sanción y la confirmación queda como "Rechazada por M2" con el motivo. La sanción espera el acuse porque M2 es dueño del estado de la reserva y así no hace falta compensar una sanción ya aplicada. Al anular una inasistencia ya aceptada (RN-SAN-14), M3 publica `NoShowReportVoided` en el mismo tópico, con el `reservationId` y el motivo. [POR DEFINIR con M2: qué hace M2 al recibirlo, porque la reserva ya quedó finalizada y el recurso liberado.]
- **RN-INT-05 — Devolución y daño hacia M1.** M3 publica en `unimag.m3.uso-novedades.v1` (clave `recurso_id`, cabeceras `X-Correlation-Id`, `X-Event-Type`, `X-Source-Module`, `X-Event-Version`) uno de dos eventos por cada check-out: `reserva.finalizada` cuando el recurso vuelve sin novedad técnica (M1 lo pasa a `DISPONIBLE`) o `reporte.danos` cuando hay daño (M1 lo pasa a `EN_MANTENIMIENTO`). Nunca se envían los dos por el mismo check-out, para que M1 no marque disponible un recurso dañado. Un daño reportado fuera del check-out también sale como `reporte.danos`. M1 decide y aplica el estado; M3 no espera confirmación. Los esquemas de payload de M1 están en blanco y M1 describe `reserva.finalizada` solo para entregas "a tiempo y en óptimas condiciones": [POR DEFINIR con M1: payload de ambos eventos y si una devolución tardía sin daño también usa `reserva.finalizada`.]
- **RN-INT-06 — Consulta de cumplimiento para M2.** M3 expone `GET /api/v1/compliance/people/{code}`, donde `code` es el código institucional del estudiante. Responde con las sanciones vigentes `activeSanctions[{reason, startsOn, endsOn, scope}]` (`scope` es `ESPACIOS`, `ACTIVOS` o `TODO`; INASISTENCIA → `ESPACIOS`), `absenceCount` (inasistencias aceptadas y no anuladas del estudiante), `lateReturns` (check-outs con retraso) y `generatedAt`. Un estudiante bloqueado (score 0) aparece como una sanción sin `endsOn` y con alcance `TODO`; sin `endsOn`, M2 la trata como vigente. Un estudiante sin historial recibe `200` con el reporte vacío. Las sanciones "Cumplida" y "Levantada" no se incluyen. M2 no guarda copia y consulta en cada operación: si M3 no responde, M2 bloquea la confirmación de reservas (`503`) y deja pasar la consulta con aviso. [POR DEFINIR con M2: ventana de `absenceCount` y `lateReturns` (histórico o por periodo) y texto de `reason`.]
- **RN-INT-07 — Novedades pendientes para M1.** M3 expone `GET /api/v1/novedades/pendientes?recursoId={id}`, que responde `{ "recursoId": "...", "tieneNovedadPendiente": true|false }`. M1 la usa antes de una transición crítica y, si M3 no responde, no ejecuta la transición. Una novedad es "Pendiente" desde que se registra el reporte de daño y pasa a "Resuelta" cuando M1 informa por `estado.actualizado` (`unimag.m1.recursos-estados.v1`) que el recurso salió de mantenimiento. [POR DEFINIR con M1: que `estado.actualizado` se emita también para ese cambio, porque hoy se describe solo para cambios manuales hechos en M1.]
- **RN-INT-08 — Identificación.** El estudiante se identifica por su código institucional, y el recurso por el identificador de M1 (placa o ID). Si falta alguno, no se registra el evento con valores supuestos.
- **RN-INT-09 — Disponibilidad.** M1 y M2 consultan a M3 en línea para decidir; M3 debe responder esas consultas aunque Kafka esté caído. Lo que M3 publica en Kafka nunca bloquea el registro del check-out, del reporte ni de la confirmación.

Resumen de contratos:

| Dirección | Canal | Evento o endpoint | Estado del contrato |
|---|---|---|---|
| M2 → M3 | `module2.reservation.record.v1` | `ReservationRecordCreated` | Acordado en el plan general de M2 |
| M2 → M3 | `module2.reservation.cancellation.v1` | `ReservationCancelled` | Acordado en el plan general de M2 |
| M2 → M3 | `module2.reservation.record.v1` | `LoanDeclaredLost` | Por acordar |
| M2 → M3 | `module2.reservation.no-show-ack.v1` | `NoShowReportAcknowledged` | Por acordar |
| M2 → M3 | `module2.reservation.check-out-ack.v1` | `CheckOutAcknowledged` | Por acordar |
| M3 → M2 | `module3.reservation.no-show.v1` | `NoShowReported`, `NoShowReportVoided` | Propuesto por M2 |
| M3 → M2 | `module3.reservation.check-out.v1` | `CheckOutRegistered` | Propuesto por M2 |
| M2 → M3 (REST) | `GET /api/v1/compliance/people/{code}` | Reporte de cumplimiento | Propuesto por M2; M3 lo expone |
| M3 → M1 | `unimag.m3.uso-novedades.v1` | `reserva.finalizada`, `reporte.danos` | Tópico definido; payload en blanco |
| M1 → M3 | `unimag.m1.recursos-estados.v1` | `estado.actualizado` | Definido por M1 |
| M1 → M3 (REST) | `GET /api/v1/novedades/pendientes?recursoId=` | Novedades pendientes | Propuesto por M1; M3 lo expone |

M1 y M2 usan convenciones distintas (M2: envoltorio con `eventId`, `type`, `version`, `occurredAt`, `data` y clave `reservation_id`; M1: cabeceras Kafka y clave `recurso_id`). M3 respeta la del módulo que consume cada evento.

## 6. Decisiones pendientes

**Por definir dentro del equipo:** ninguno por ahora.

**Por acordar con M1 y M2 (contratos):**
- Payload de `reserva.finalizada` y `reporte.danos`, y si una devolución tardía sin daño también usa `reserva.finalizada` (RN-INT-05).
- Que `estado.actualizado` de M1 cubra la salida de mantenimiento (RN-INT-07).
- Los tres tópicos que M2 aún marca "por acordar": acuse de no asistencia, acuse de check-out y `LoanDeclaredLost` (RN-INT-03, RN-INT-04, RN-SAN-12).
- Ventana de `absenceCount` y `lateReturns`, y el texto de `reason` (RN-INT-06).
- Que M2 informe también el inicio de uso de una reserva a M3: hoy solo se lo informa a M1, así que M3 no sabe si el estudiante llegó y el personal constata la ausencia en sitio (RN-INT-04).
- Qué hace M2 al recibir `NoShowReportVoided` de una inasistencia ya aceptada (RN-INT-04).
- La analítica por facultad: la ficha de reserva de M2 trae el rol y el código del titular, pero no su facultad ni programa, y `GET /recursos/{id}` de M1 está descrita como exclusiva de Dirección/Monitor en un documento y como consumible por M2 y M3 en otro.
- Si M2 debe restringir nuevas reservas a quien tiene un préstamo vencido, usando sus propios datos; mientras tanto, M3 solo bloquea cuando el score llega a 0.

**Decisiones ya tomadas por el equipo (2026-10-22):** el retraso se penaliza con descuento diario de score hasta 0 al día 7 y no genera sanción por mora; las sanciones corren en paralelo; un reporte de daño es una infracción; cancelar con anticipación no cambia el score; el costo sale de un catálogo de M3 y el personal administrativo registra pagos y anulaciones; la inasistencia se considera a los 30 minutos de la hora de inicio; no hay límite de desbloqueos manuales; un préstamo perdido genera un cobro del 50% del valor del recurso; el personal administrativo ve el detalle por estudiante y los datos generales, y no se registra quién consulta cada detalle; un estudiante con varios préstamos vencidos recibe un solo descuento diario, el del más antiguo; no hay plazo máximo para confirmar una inasistencia; el personal administrativo puede anular una inasistencia en cualquier momento, con motivo y auditoría, y se restituyen los puntos.

**Propuestas por confirmar con el equipo:**
- La recuperación de +5 cada 30 días aplica a todos los estudiantes, no solo a los bloqueados (RN-SCO-05).
- El desbloqueo manual deja el score en 5, no en el máximo, y levanta las sanciones vigentes del estudiante (RN-BLO-04, RN-BLO-05).
- El bloqueo restringe solo reservas nuevas y nunca devoluciones (RN-BLO-02).
- El descuento por retraso es proporcional (una séptima parte del score al vencer, por día), no se devuelve al entregar, aplica solo a préstamos de activos y es uno solo por día aunque haya varios préstamos vencidos (RN-SCO-11).
- Al anular una inasistencia se restituyen los puntos que descontó (RN-SCO-12).
- La sanción por inasistencia se crea al recibir el acuse de aceptación de M2 (RN-SAN-03, RN-INT-04).
- Un check-out produce un solo evento hacia M1: `reserva.finalizada` o `reporte.danos`, nunca ambos (RN-INT-05).
- La inasistencia sigue aplicando solo a reservas de espacios, aunque M2 la admite también para retiros de activos.
- La inasistencia sigue confirmándola el personal administrativo (no es automática), porque M3 no sabe si el estudiante llegó (RN-INT-04).
- Un cobro por daño preexistente se corrige con la anulación del personal administrativo (RN-COB-09).
