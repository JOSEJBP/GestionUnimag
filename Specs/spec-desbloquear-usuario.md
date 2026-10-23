# Feature Specification: Desbloquear Usuario

**Created:** 2026-10-17

> Un estudiante está bloqueado mientras su score sea 0 (RN-BLO-01). El desbloqueo manual está definido en RN-BLO-04; el bloqueo y el score en `reglas-negocio-m3.md`.

## User Scenarios & Testing

### User Story 1 - Desbloquear manualmente a un estudiante (Priority: P1)
Como personal administrativo, quiero desbloquear a un estudiante cuyo score llegó a 0, indicando el motivo, para que pueda volver a hacer reservas sin esperar el período de recuperación.

**Why this priority:** Es la salida manual del bloqueo; sin ella, un estudiante bloqueado por un caso justificado dependería solo de esperar 30 días sin infracciones.

**Independent Test:** Con un estudiante con score 0, desbloquearlo con un motivo y verificar que su score queda en 5, que el cambio aparece en su historial y en la auditoría, y que la siguiente consulta de cumplimiento de M2 ya no lo muestra como bloqueado.

**Acceptance Scenarios:**

**Scenario: Desbloqueo exitoso**
- **Given** un estudiante bloqueado (score 0).
- **When** el personal administrativo lo desbloquea e indica el motivo.
- **Then** M3 sube su score al valor configurado (5 por defecto), levanta sus sanciones vigentes (RN-BLO-05), registra el cambio en el historial con tipo DESBLOQUEO_MANUAL, y guarda la auditoría (quién, cuándo y motivo). M2 se entera en su siguiente consulta de cumplimiento del estudiante (RN-INT-06), sin que M3 le publique nada.

**Scenario: Estudiante que no está bloqueado**
- **Given** un estudiante con score mayor que 0, por ejemplo uno que ya recuperó puntos automáticamente.
- **When** el personal administrativo intenta desbloquearlo.
- **Then** el sistema rechaza la operación e informa que el estudiante no está bloqueado.

**Scenario: Desbloqueo sin motivo**
- **Given** un estudiante bloqueado.
- **When** el personal administrativo intenta desbloquearlo sin indicar el motivo.
- **Then** el sistema rechaza la operación e informa que el motivo es obligatorio.

**Scenario: Usuario sin permisos**
- **Given** una persona sin rol de personal administrativo autorizado.
- **When** intenta desbloquear a un estudiante.
- **Then** el sistema rechaza la operación.

### User Story 2 - Consultar los estudiantes bloqueados (Priority: P2)
Como personal administrativo, quiero ver la lista de estudiantes bloqueados con su fecha de bloqueo y su última sanción, para decidir a quién desbloquear.

**Why this priority:** Facilita la decisión, pero el desbloqueo puede hacerse sobre un estudiante conocido aunque no exista el listado.

**Independent Test:** Con tres estudiantes bloqueados y uno no bloqueado, verificar que el listado muestra solo los tres.

**Acceptance Scenarios:**

**Scenario: Listado de bloqueados**
- **Given** estudiantes con score 0.
- **When** el personal administrativo abre el listado.
- **Then** el sistema muestra, ordenados por fecha de bloqueo, cada estudiante con la fecha en que llegó a 0, su última sanción (tipo y fecha), los días transcurridos sin infracciones y las sanciones vigentes que se levantarían al desbloquearlo, para que el personal administrativo vea las consecuencias antes de decidir.

**Scenario: Sin bloqueados**
- **Given** ningún estudiante con score 0.
- **When** el personal administrativo abre el listado.
- **Then** el sistema muestra un estado vacío informativo.

### User Story 3 - Devolver el acceso completo a reservar (Priority: P1)
Como personal administrativo, quiero que al desbloquear a un estudiante también se levanten sus sanciones vigentes, para que realmente pueda volver a reservar.

**Why this priority:** El score puede llegar a 0 por inasistencias o por un préstamo sin devolver; si el estudiante tiene sanciones por inasistencia vigentes y siguieran activas, el desbloqueo no le devolvería el acceso a reservar espacios.

**Independent Test:** Desbloquear a un estudiante con score 0 y dos sanciones vigentes, y verificar que su score queda en 5, que las sanciones quedan levantadas y que la consulta de cumplimiento que usa M2 ya no devuelve sanciones ni bloqueo.

**Acceptance Scenarios:**

**Scenario: Desbloqueo con sanciones vigentes**
- **Given** un estudiante bloqueado que además tiene una o más sanciones vigentes.
- **When** el personal administrativo lo desbloquea.
- **Then** su score pasa a 5, las sanciones vigentes pasan a "Levantada" con el motivo del desbloqueo (RN-BLO-05), la consulta de cumplimiento que usa M2 deja de reportarle restricciones, y el estudiante puede volver a reservar.

**Scenario: Sanciones levantadas no reducen el historial**
- **Given** una sanción que ya redujo el score cuando se creó.
- **When** se levanta por el desbloqueo.
- **Then** el score no se modifica por el levantamiento; el historial conserva la reducción original (RN-SCO-02).

**Scenario: Nueva sanción tras el desbloqueo**
- **Given** un estudiante desbloqueado manualmente con score 5.
- **When** se le aplica una sanción nueva (−5).
- **Then** su score vuelve a 0 y queda bloqueado de nuevo.

## Edge Cases

- **Recuperación automática y desbloqueo manual simultáneos:** solo uno debe aplicarse; el segundo se rechaza porque el estudiante ya no está bloqueado, y el score no se suma dos veces.
- **Estudiante no identificable:** el sistema debe rechazar la operación e informar que no fue posible identificarlo.
- **M2 no disponible:** no afecta el desbloqueo; M3 no le publica nada. Cuando M2 vuelva a consultar a M3, ya verá al estudiante sin restricciones.
- **Reserva que M2 está validando mientras se desbloquea:** M2 consulta a M3 en cada operación y no guarda copia, así que puede negar una reserva justo antes del desbloqueo; el estudiante la repite después.
- **Falla al actualizar el score o al levantar una sanción:** el desbloqueo es una sola operación; si algo falla, no se aplica nada (ni el score ni el levantamiento), el sistema informa el error y el estudiante no queda en un estado intermedio.
- **Estudiante bloqueado sin sanciones vigentes:** el desbloqueo solo sube el score; no hay sanciones que levantar.
- **Desbloqueos repetidos al mismo estudiante:** se permiten sin límite, y cada uno queda en la auditoría (RN-BLO-04).
- **Estudiante con un préstamo vencido sin devolver:** el desbloqueo se aplica, pero el descuento diario por retraso continúa (RN-SCO-11) y puede dejarlo otra vez en 0; el sistema avisa al personal administrativo antes de confirmar.
- **Parámetro del score tras el desbloqueo no configurado:** el sistema debe rechazar la operación e informar el motivo; no usa un valor supuesto.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe permitir al personal administrativo autorizado desbloquear a un estudiante cuyo score sea 0.
- **FR-002:** El sistema debe exigir un motivo para cada desbloqueo manual.
- **FR-003:** El sistema debe rechazar el desbloqueo de un estudiante que no esté bloqueado, informando el motivo.
- **FR-004:** El sistema debe fijar el score del estudiante en el valor configurado para el desbloqueo (5 por defecto).
- **FR-005:** El sistema debe registrar el cambio en el historial del score con tipo DESBLOQUEO_MANUAL, score antes y después, y motivo.
- **FR-006:** El sistema debe guardar la auditoría del desbloqueo: quién lo realizó, cuándo y con qué motivo.
- **FR-007:** El sistema debe reflejar el desbloqueo y el levantamiento de las sanciones en la consulta de cumplimiento que M2 usa (RN-INT-06) desde el momento en que se completa la operación, sin publicar eventos a M2.
- **FR-008:** El sistema debe levantar, en la misma operación del desbloqueo, todas las sanciones vigentes del estudiante, marcándolas "Levantada" con el motivo del desbloqueo, y sin modificar el score por ese levantamiento.
- **FR-009:** El sistema debe listar los estudiantes bloqueados con su fecha de bloqueo, su última sanción y los días sin infracciones.
- **FR-010:** El sistema debe restringir este caso de uso al personal administrativo autorizado.
- **FR-011:** El sistema debe resolver el desbloqueo concurrente con la recuperación automática sin duplicar el aumento de score.

## Key Entities
- **Estudiante:** Persona bloqueada que se desbloquea.
- **Score_Confianza / Historial_Score:** Valor del score e historial donde se registra el cambio (ver `reglas-negocio-m3.md`).
- **Sancion:** Sanciones vigentes del estudiante que el desbloqueo pasa a "Levantada" (ver `reglas-negocio-m3.md`, RN-SAN).
- **Auditoria_Desbloqueo:** Registro con estudiante, responsable, fecha, motivo, score antes y después, y sanciones levantadas.
- **Configuracion_Sistema:** Score tras el desbloqueo manual (5).
- **Personal administrativo:** Actor que desbloquea.

## Integración con Módulos Externos

| Módulo | Tipo de relación | Justificación |
|---|---|---|
| Módulo 2 (Reservas) | **Sin integración directa** | M2 niega reservas a estudiantes bloqueados o sancionados consultando a M3 en cada operación (`GET /api/v1/compliance/people/{code}`) y sin guardar copia (RN-BLO-07, RN-SAN-11, RN-INT-06). El desbloqueo se refleja solo en esa consulta; M3 no publica ningún evento de desbloqueo. |
| Módulo 1 (Activos y Espacios) | **Sin integración directa** | El desbloqueo no afecta el inventario. |

## Success Criteria

### Measurable Outcomes
- **SC-001:** El 100% de los desbloqueos válidos deben dejar el score en el valor configurado, con historial y auditoría.
- **SC-002:** El 100% de los desbloqueos sin motivo o sobre estudiantes no bloqueados deben ser rechazados.
- **SC-003:** El 100% de los desbloqueos deben dejar al estudiante sin sanciones vigentes y en condiciones de reservar.
- **SC-004:** El 100% de los desbloqueos deben reflejarse en la consulta de cumplimiento de M2 apenas se complete la operación.
- **SC-005:** El 0% de los desbloqueos simultáneos con la recuperación automática deben sumar puntos dos veces.
- **SC-006:** El sistema debe completar el desbloqueo en menos de 2 segundos.
