# Feature Specification: Reportar No Asistencia

**Created:** 2026-09-04

## User Scenarios & Testing

### User Story 1 - Reporte de inasistencia a una reserva (Priority: P1)
Como profesor o administrador, quiero reportar que un estudiante no asistió a su reserva de un recurso, para dejar constancia del incumplimiento y que se calcule la sanción correspondiente.

**Why this priority:** Es el disparador obligatorio del flujo de sanciones por inasistencia; sin él, este tipo de incumplimiento no quedaría registrado ni penalizado.

**Independent Test:** Puede probarse de forma independiente creando una reserva a nombre de un estudiante, sin registrar check-in, y reportando la inasistencia una vez vencida la ventana de la reserva.

**Acceptance Scenarios:**

**Scenario: Reporte exitoso de inasistencia**
- **Given** una reserva vigente de un estudiante sin check-in registrado, cuya ventana de tiempo ya venció.
- **When** el profesor o administrador reporta la inasistencia.
- **Then** el sistema registra el reporte (reserva, estudiante, fecha, quien reporta) y dispara obligatoriamente (`«include»`) el caso de uso *Calcular penalización*.

**Scenario: Intento de reportar inasistencia sobre una reserva con check-in ya realizado**
- **Given** una reserva en la que el estudiante ya registró check-in.
- **When** se intenta reportar inasistencia sobre esa reserva.
- **Then** el sistema rechaza el reporte e informa que la reserva ya cuenta con check-in registrado.

**Scenario: Intento de reporte duplicado**
- **Given** una reserva sobre la cual ya existe un reporte de inasistencia registrado.
- **When** se intenta reportar inasistencia nuevamente sobre la misma reserva.
- **Then** el sistema rechaza el segundo reporte e informa que ya existe uno registrado.

## Edge Cases

- **Reserva no encontrada o no identificada:** el sistema debe rechazar el reporte e informar que no fue posible identificar la reserva.
- **Reserva aún dentro de su ventana de tiempo vigente:** el sistema debe rechazar el reporte e informar que la ventana de la reserva no ha finalizado.
- **Estudiante no identificado a partir de la reserva:** el sistema debe rechazar el reporte e informar que no fue posible asociar la inasistencia a un estudiante válido.
- **Error al registrar el reporte:** el sistema debe informar que la inasistencia no pudo registrarse y evitar dejar un registro incompleto.
- **Error al disparar el cálculo de la penalización:** el sistema debe informar que el reporte quedó registrado pero la penalización no pudo iniciarse, marcando la inconsistencia para revisión.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe permitir a un profesor o administrador reportar la inasistencia de un estudiante sobre una reserva específica.
- **FR-002:** El sistema debe identificar la reserva y el estudiante asociados al reporte.
- **FR-003:** El sistema debe verificar que la reserva no tenga un check-in registrado antes de aceptar el reporte de inasistencia.
- **FR-004:** El sistema debe verificar que la ventana de tiempo de la reserva ya haya finalizado antes de aceptar el reporte.
- **FR-005:** El sistema debe registrar el reporte de inasistencia, incluyendo fecha, reserva, estudiante y quien lo reporta.
- **FR-006:** El sistema debe disparar obligatoriamente (`«include»`) el caso de uso *Calcular penalización* una vez registrado correctamente el reporte.
- **FR-007:** El sistema debe impedir el registro de un reporte de inasistencia duplicado para la misma reserva.
- **FR-008:** El sistema debe informar cuando la reserva no pueda identificarse, ya tenga check-in registrado, o aún esté dentro de su ventana vigente.

## Key Entities
- **Reserva:** Representa la solicitud previa de un estudiante para usar un recurso en una ventana de tiempo determinada.
- **Reporte_Inasistencia:** Registro del incumplimiento, con fecha, reserva asociada, estudiante y quien reporta.
- **Estudiante:** Titular de la reserva sobre la que se reporta la inasistencia.
- **Profesor / Administrador:** Actor que ejecuta el reporte.

## Success Criteria

### Measurable Outcomes
- **SC-001:** El 100% de los reportes de inasistencia válidos deben disparar el caso de uso *Calcular penalización*.
- **SC-002:** El 0% de los reportes de inasistencia deben duplicarse sobre la misma reserva.
- **SC-003:** El 100% de los intentos de reportar inasistencia sobre reservas con check-in ya realizado deben ser rechazados.
- **SC-004:** El sistema debe registrar el reporte y disparar la penalización en menos de 2 segundos.
