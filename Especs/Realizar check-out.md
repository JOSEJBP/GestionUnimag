# Feature Specification: Realizar check-out

**Created:** 31/08/2026

## User Scenarios & Testing

### User Story 1 - Realizar check-out de un recurso (Priority: P1)
Como estudiante, quiero realizar el check-out de un recurso que tengo actualmente en uso, para finalizar su utilización y dejar registrada la devolución.

**Why this priority:** Es la funcionalidad principal del caso de uso, ya que permite finalizar formalmente la utilización de un recurso y registrar la devolución realizada por el estudiante.

**Independent Test:** Puede probarse de forma independiente con un estudiante que tenga un recurso actualmente en uso. Al realizar el check-out, el sistema debe registrar la fecha de devolución, finalizar la utilización del recurso y disparar el flujo correspondiente según si la entrega fue a tiempo, con retraso, o con un daño reportado.

**Acceptance Scenarios:**

**Scenario: Check-out a tiempo**
- **Given** el estudiante tiene un recurso actualmente en uso y la devolución ocurre dentro del tiempo límite establecido.
- **When** el estudiante realiza el check-out del recurso.
- **Then** el sistema registra el check-out con la fecha de devolución, finaliza la utilización, y dispara (`«extend»`) *Actualizar score de confianza* para aumentar el score en 2 puntos.

**Scenario: Check-out con retraso**
- **Given** el estudiante tiene un recurso actualmente en uso y la devolución ocurre después del tiempo límite establecido.
- **When** el estudiante realiza el check-out del recurso.
- **Then** el sistema registra el check-out con la fecha de devolución, finaliza la utilización, y dispara (`«extend»`) *Calcular penalización*.

**Scenario: Check-out con reporte de daño**
- **Given** el estudiante está realizando el check-out de un recurso.
- **When** indica, durante el check-out, que el recurso presenta un daño.
- **Then** el sistema registra el check-out normalmente y dispara (`«extend»`) *Reportar novedad técnica*, sin que el reporte de daño impida completar el check-out.

### User Story 2 - Rechazo del check-out por estado inválido de la utilización (Priority: P2)
Como sistema, quiero rechazar un check-out cuando la utilización, el recurso o la solicitud no estén en un estado válido, para evitar registros duplicados o inconsistentes.

**Why this priority:** Es un escenario de fallo de la funcionalidad principal; sin estas validaciones, el sistema podría finalizar dos veces la misma utilización o registrar un check-out sobre datos inconsistentes.

**Independent Test:** Puede probarse de forma independiente intentando un check-out sobre una utilización ya finalizada, sobre un recurso no identificable, o enviando dos solicitudes simultáneas para la misma utilización.

**Acceptance Scenarios:**

**Scenario: Utilización ya finalizada**
- **Given** una utilización que ya fue finalizada mediante un check-out previo.
- **When** se intenta registrar un nuevo check-out sobre esa misma utilización.
- **Then** el sistema rechaza la operación e informa que la utilización ya fue finalizada.

**Scenario: Recurso no identificado**
- **Given** una utilización activa cuyo recurso asociado no puede identificarse en el sistema.
- **When** el estudiante intenta realizar el check-out.
- **Then** el sistema rechaza la operación e informa que no fue posible identificar el recurso asociado a la utilización.

**Scenario: Intentos simultáneos de check-out**
- **Given** dos solicitudes de check-out enviadas casi al mismo tiempo para la misma utilización.
- **When** el sistema procesa ambas solicitudes.
- **Then** el sistema registra un único check-out exitoso y rechaza la segunda solicitud, informando que la utilización ya fue finalizada.

## Edge Cases

- **Estudiante sin utilización activa:** el sistema debe rechazar el check-out e informar que no existe una utilización activa asociada al estudiante.
- **Error al registrar el check-out:** el sistema debe informar que el check-out no pudo registrarse y evitar que se almacene un registro incompleto.
- **Error al finalizar la utilización:** el sistema debe informar que no fue posible completar el check-out y evitar que la utilización quede en un estado inconsistente.
- **Error al registrar la fecha de devolución:** el sistema debe impedir la finalización del check-out e informar que no fue posible registrar correctamente la fecha de devolución.
- **Error al disparar el caso de uso correspondiente (Actualizar score, Calcular penalización o Reportar novedad técnica):** el sistema debe informar que el check-out se registró pero el flujo posterior no pudo iniciarse, marcando la inconsistencia para revisión.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe permitir al estudiante realizar el check-out de un recurso que tenga actualmente en uso.
- **FR-002:** El sistema debe identificar la utilización activa asociada al estudiante.
- **FR-003:** El sistema debe identificar el recurso asociado a la utilización activa.
- **FR-004:** El sistema debe registrar el check-out realizado por el estudiante.
- **FR-005:** El sistema debe registrar la fecha en que el estudiante realiza el check-out.
- **FR-006:** El sistema debe finalizar la utilización del recurso cuando el check-out sea registrado correctamente.
- **FR-007:** El sistema debe determinar si el check-out ocurre dentro o fuera del tiempo límite establecido para la utilización.
- **FR-008:** El sistema debe disparar (`«extend»`) *Actualizar score de confianza* cuando el check-out ocurra dentro del tiempo límite.
- **FR-009:** El sistema debe disparar (`«extend»`) *Calcular penalización* cuando el check-out ocurra fuera del tiempo límite.
- **FR-010:** El sistema debe permitir reportar un daño técnico durante el check-out, disparando (`«extend»`) *Reportar novedad técnica* sin bloquear el registro del check-out.
- **FR-011:** El sistema debe impedir que una utilización finalizada sea registrada nuevamente mediante un check-out.
- **FR-012:** El sistema debe garantizar que, ante intentos simultáneos de check-out sobre la misma utilización, solo uno se registre exitosamente.
- **FR-013:** El sistema debe informar al estudiante cuando el check-out haya sido registrado correctamente.
- **FR-014:** El sistema debe informar al estudiante cuando no exista una utilización activa que pueda ser finalizada mediante un check-out.
- **FR-015:** El sistema debe mantener asociado el check-out con el estudiante, el recurso y la utilización correspondiente.

## Key Entities
- **Estudiante:** Representa a la persona que tiene bajo su responsabilidad un recurso y realiza el check-out.
- **Recurso:** Representa el activo universitario que se encuentra en uso por parte del estudiante y cuya utilización será finalizada mediante el check-out.
- **Utilización:** Representa la relación entre el estudiante y el recurso durante el período en que este se encuentra bajo su responsabilidad. Incluye el tiempo límite configurado, usado para determinar si el check-out ocurre a tiempo o con retraso.
- **Check-out:** Representa la operación mediante la cual se registra la devolución y se finaliza la utilización del recurso. Contiene la fecha en que se realizó, si ocurrió a tiempo o con retraso, y se relaciona con el estudiante, el recurso y la utilización correspondiente.

## Integración con Módulos Externos

| Módulo | Tipo de relación | Justificación |
|---|---|---|
| Módulo 2 | **Reactivo (cola)** | El check-out publica un evento de confirmación; Módulo 2 lo consume de forma asíncrona para informar al estudiante. El registro del check-out no debe bloquearse esperando que la notificación se entregue. |

## Success Criteria

### Measurable Outcomes
- **SC-001:** El 100% de los check-outs realizados por estudiantes con una utilización activa deben quedar registrados con su fecha de devolución correspondiente.
- **SC-002:** El 100% de los check-outs registrados correctamente deben finalizar la utilización correspondiente.
- **SC-003:** El 100% de los check-outs registrados correctamente deben quedar asociados al estudiante, recurso y utilización correspondientes.
- **SC-004:** El 100% de los check-outs a tiempo deben disparar *Actualizar score de confianza*, y el 100% de los check-outs con retraso deben disparar *Calcular penalización*.
- **SC-005:** El 100% de los check-outs con reporte de daño deben disparar *Reportar novedad técnica* sin impedir el registro del check-out.
- **SC-006:** El 100% de los intentos de realizar check-out sin una utilización activa deben ser rechazados.
- **SC-007:** El sistema debe impedir el 100% de los intentos de registrar nuevamente un check-out para una utilización que ya haya sido finalizada, incluso ante solicitudes simultáneas.
