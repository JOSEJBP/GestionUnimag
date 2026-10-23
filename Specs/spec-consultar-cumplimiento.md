# Feature Specification: Consultar Mi Cumplimiento (Score, Sanciones, Cobros)

**Created:** 2026-10-17
**Updated:** 2026-10-22 (se agrega la consulta que M2 hace a M3)

> Las reglas de score, sanciones, bloqueo y cobros que esta consulta muestra están en `reglas-negocio-m3.md`. La consulta que M2 hace a M3 sobre el mismo estudiante está en RN-INT-06.

## User Scenarios & Testing

### User Story 1 - Consultar el score de confianza y su historial (Priority: P1)
Como estudiante, quiero ver mi score de confianza, cómo sube y cómo baja, y el historial de cambios, para entender mi situación y verificar qué acciones lo afectaron.

**Why this priority:** El estudiante necesita transparencia sobre su reputación para corregir su comportamiento y entender el efecto de sus entregas.

**Independent Test:** Con un estudiante con tres cambios de score, abrir la consulta y verificar el valor actual, las reglas de aumento y reducción y la lista cronológica.

**Acceptance Scenarios:**

**Scenario: Score e historial**
- **Given** un estudiante autenticado con cambios de score registrados.
- **When** consulta su cumplimiento.
- **Then** el sistema muestra su score actual sobre el máximo, las reglas del score (por ejemplo, "+2 puntos por cada entrega a tiempo", "-5 puntos por cada sanción", "pérdida diaria de puntos por cada día de retraso en un préstamo, hasta quedar en 0 al día 7" y "+5 puntos por cada 30 días sin infracciones") y el historial con fecha, motivo y variación, diferenciando aumentos, reducciones (por sanción o por retraso), recuperaciones, desbloqueos y restituciones por anulación, del más reciente al más antiguo.

**Scenario: Estudiante sin historial**
- **Given** un estudiante sin cambios de score.
- **When** consulta su cumplimiento.
- **Then** el sistema muestra su score inicial (50/50) y un estado vacío informativo en el historial.

### User Story 2 - Consultar sanciones vigentes (Priority: P1)
Como estudiante, quiero consultar mis sanciones vigentes con su causa y fecha de fin, para saber cuándo podré volver a reservar.

**Why this priority:** La sanción solo es comprensible si el estudiante sabe por qué la tiene y cuándo termina; sin esta consulta se enteraría al ser rechazado en una reserva.

**Independent Test:** Con dos sanciones por inasistencia vigentes a la vez, verificar que cada una se muestra por separado con causa, alcance y fecha de fin propia.

**Acceptance Scenarios:**

**Scenario: Lista de sanciones vigentes**
- **Given** un estudiante con una o más sanciones vigentes.
- **When** consulta su cumplimiento.
- **Then** el sistema muestra una tarjeta por sanción con tipo, causa, alcance (reservas de espacios) y fecha de fin. Varias sanciones pueden estar vigentes a la vez, cada una con su propia fecha (RN-SAN-07).

**Scenario: Préstamo vencido sin devolver**
- **Given** un estudiante con un préstamo de un activo cuya fecha límite pasó y que aún no devuelve.
- **When** consulta su cumplimiento.
- **Then** el sistema muestra un aviso destacado con el recurso, los días de retraso, los puntos que ya perdió, cuánto pierde por cada día adicional y que al día 7 su score llegará a 0 (RN-SCO-11).

**Scenario: Usuario bloqueado**
- **Given** un estudiante con el score en 0 (bloqueado).
- **When** abre su cumplimiento.
- **Then** el sistema muestra un aviso destacado "Estado: bloqueado. Su score actual es 0 / 50", indica que no puede hacer reservas nuevas pero sí devolver lo que tiene en uso, y explica cómo salir del bloqueo: recuperar puntos tras 30 días sin infracciones (con la fecha estimada) o solicitar el desbloqueo al personal administrativo.

**Scenario: Sin sanciones**
- **Given** un estudiante sin sanciones vigentes.
- **When** consulta su cumplimiento.
- **Then** el sistema muestra "No tienes sanciones activas".

**Scenario: Aviso de restricción**
- **Given** un estudiante con una sanción que restringe nuevas reservas.
- **When** abre su cumplimiento.
- **Then** el sistema muestra un aviso destacado indicando que no puede reservar, el motivo y hasta cuándo.

### User Story 3 - Consultar cobros (Priority: P2)
Como estudiante, quiero consultar los cobros generados a mi nombre, para conocer el motivo y el monto.

**Why this priority:** El cobro es una consecuencia económica y el estudiante debe poder verla; el pago ocurre fuera de M3.

**Independent Test:** Con un cobro generado, verificar que aparece con motivo, recurso, monto, fecha y estado.

**Acceptance Scenarios:**

**Scenario: Lista de cobros**
- **Given** un estudiante con cobros generados.
- **When** consulta su cumplimiento.
- **Then** el sistema muestra cada cobro con motivo, recurso, monto, fecha y estado.

**Scenario: Sin cobros**
- **Given** un estudiante sin cobros.
- **When** consulta su cumplimiento.
- **Then** el sistema muestra un estado vacío informativo.

### User Story 4 - Responder a M2 las restricciones activas de un estudiante (Priority: P1)
Como Módulo 2, quiero consultar a M3 las sanciones vigentes y el bloqueo de un estudiante, para negar una reserva nueva con el mensaje "sanción activa" cuando corresponda.

**Why this priority:** Es la única forma en que las sanciones, el bloqueo y el desbloqueo tienen efecto sobre las reservas: M2 no guarda copia ni recibe eventos de sanción, y consulta a M3 en cada operación.

**Independent Test:** Con un estudiante con una sanción por inasistencia vigente, otro bloqueado, otro sin historial y uno recién desbloqueado, consultar `GET /api/v1/compliance/people/{code}` y verificar la respuesta de cada uno.

**Acceptance Scenarios:**

**Scenario: Estudiante con sanciones vigentes**
- **Given** un estudiante con dos sanciones por inasistencia vigentes a la vez.
- **When** M2 consulta su cumplimiento por código institucional.
- **Then** M3 responde `activeSanctions` con una entrada por sanción (`reason`, `startsOn`, `endsOn` inclusive y `scope` `ESPACIOS`), `absenceCount`, `lateReturns` y `generatedAt`.

**Scenario: Estudiante bloqueado**
- **Given** un estudiante con el score en 0.
- **When** M2 consulta su cumplimiento.
- **Then** la respuesta incluye una sanción sin `endsOn` y con `scope` `TODO` que representa el bloqueo, que M2 trata como vigente.

**Scenario: Estudiante sin historial**
- **Given** un estudiante sin sanciones ni registros.
- **When** M2 consulta su cumplimiento.
- **Then** M3 responde `200` con `activeSanctions` vacío y los contadores en cero.

**Scenario: Estudiante desbloqueado**
- **Given** un estudiante al que el personal administrativo acaba de desbloquear.
- **When** M2 consulta su cumplimiento.
- **Then** la respuesta ya no incluye el bloqueo ni las sanciones levantadas.

## Edge Cases

- **Consulta de M2 sin código institucional válido:** M3 responde un error de validación; M2 lo trata como no verificado.
- **Consulta de M2 con M3 lento o caído:** M2 bloquea la confirmación de una reserva (`503`) y deja pasar la consulta con aviso; por eso la respuesta debe ser rápida y con datos al día, sin depender de Kafka (RN-INT-09).
- **Estudiante no autenticado:** el sistema debe rechazar la consulta y pedir autenticación.
- **Sanción que vence durante la consulta:** al recargar, el sistema debe dejar de mostrarla como vigente.
- **Estudiante desbloqueado mientras consulta:** al recargar, el sistema debe quitar el aviso de bloqueo y mostrar el nuevo score.
- **Préstamo vencido sin devolver:** no es una sanción (RN-SAN-02); se muestra como aviso con el descuento de puntos, no en la lista de sanciones. Si M2 lo declaró perdido (día 7), aparece también el cobro por pérdida (RN-COB-12).
- **Historial extenso:** el sistema debe paginar el historial sin degradar el tiempo de respuesta.
- **Falla parcial (por ejemplo, los cobros no cargan pero el score sí):** el sistema debe mostrar el error solo en la sección afectada, sin bloquear las demás.
- **Datos de un recurso ya no disponibles en M1:** el sistema debe usar la instantánea guardada al registrar el evento.
- **Intento de consultar datos de otro estudiante:** el sistema debe rechazarlo.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe mostrar al estudiante su score actual sobre el máximo configurado.
- **FR-002:** El sistema debe mostrar las reglas vigentes para aumentar y reducir el score, incluida la recuperación por 30 días sin infracciones.
- **FR-003:** El sistema debe mostrar el historial del score en orden cronológico inverso, con fecha, motivo y variación, diferenciando aumentos, reducciones, recuperaciones y desbloqueos, y con paginación.
- **FR-004:** El sistema debe mostrar un estado vacío con el score inicial cuando no haya historial.
- **FR-005:** El sistema debe mostrar cada sanción vigente en una tarjeta con tipo, causa, alcance y fecha de fin.
- **FR-006:** El sistema debe mostrar "No tienes sanciones activas" cuando no existan sanciones vigentes.
- **FR-007:** El sistema debe mostrar un aviso destacado cuando exista una sanción que restrinja nuevas reservas, con su motivo y fecha de fin.
- **FR-008:** El sistema debe mostrar los cobros del estudiante con motivo, recurso, monto, fecha y estado (Pendiente, Pagado o Anulado, este último con su motivo).
- **FR-009:** El sistema debe restringir la consulta a los datos del propio estudiante autenticado.
- **FR-010:** El sistema debe mostrar siempre los datos vigentes, tomados de la fuente de verdad de M3.
- **FR-011:** El sistema debe manejar de forma independiente los estados de carga y de error de cada sección.
- **FR-012:** El sistema debe mostrar un aviso destacado cuando el estudiante esté bloqueado (score 0), con el efecto del bloqueo, la fecha estimada de la próxima recuperación de puntos y la forma de solicitar el desbloqueo al personal administrativo.
- **FR-013:** El sistema debe exponer `GET /api/v1/compliance/people/{code}` para M2, con el código institucional del estudiante, y responder las sanciones vigentes (`reason`, `startsOn`, `endsOn`, `scope`), `absenceCount`, `lateReturns` y `generatedAt` (RN-INT-06).
- **FR-014:** El sistema debe incluir el bloqueo (score 0) en esa respuesta como una sanción sin `endsOn` y con alcance `TODO`, y excluir las sanciones cumplidas o levantadas.
- **FR-015:** El sistema debe responder `200` con un reporte vacío cuando el estudiante no tenga historial, y calcular la respuesta en cada consulta a partir de los datos vigentes de M3, sin depender de Kafka.
- **FR-016:** El sistema debe mostrar un aviso destacado por cada préstamo vencido sin devolver, con los días de retraso, los puntos perdidos y el descuento diario (RN-SCO-11).

## Key Entities
- **Estudiante:** Titular de los datos consultados.
- **Score_Confianza / Historial_Score:** Valor actual y cambios registrados; el bloqueo se deriva de un score en 0 (ver `reglas-negocio-m3.md`, RN-SCO y RN-BLO).
- **Sancion:** Restricciones vigentes del estudiante (ver `reglas-negocio-m3.md`, RN-SAN).
- **Cobro:** Montos generados por daño (ver `reglas-negocio-m3.md`, RN-COB).

## Integración con Módulos Externos

| Módulo | Tipo de relación | Justificación |
|---|---|---|
| Módulo 1 (Activos y Espacios) | **Sin integración directa** | Es una consulta de lectura sobre datos propios de M3. Los datos descriptivos de los recursos se toman de las instantáneas guardadas al registrar cada evento. |
| Módulo 2 (Reservas) | **Proactivo (REST, M2 → M3)** | M2 consulta a M3 el cumplimiento de un estudiante en cada operación y necesita una respuesta inmediata y consistente para negar o permitir una reserva (RN-SAN-11, RN-INT-06). M3 es quien expone el endpoint; no publica eventos de sanción. |

## Success Criteria

### Measurable Outcomes
- **SC-001:** La consulta debe cargar el score, las sanciones y los cobros en menos de 3 segundos.
- **SC-002:** El 100% de las consultas deben mostrar únicamente datos del estudiante autenticado.
- **SC-003:** El 100% de las sanciones vigentes deben aparecer en la consulta, y ninguna sanción cumplida debe aparecer como vigente.
- **SC-004:** El 100% de los cobros generados deben aparecer en la consulta.
- **SC-005:** Un cambio de score, sanción o cobro debe reflejarse en la consulta en menos de 5 segundos.
- **SC-006:** El 100% de las fallas parciales deben afectar solo a la sección que falló.
- **SC-007:** El 100% de los estudiantes con score 0 deben ver el aviso de bloqueo.
- **SC-008:** El 100% de las consultas de M2 deben reflejar los cambios de sanción, bloqueo y desbloqueo apenas se completen.
- **SC-009:** La respuesta a M2 debe generarse en menos de 1 segundo.
