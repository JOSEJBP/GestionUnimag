# Feature Specification: Reportar Novedad Técnica (Daño)

**Created:** 2026-09-04

## User Scenarios & Testing

### User Story 1 - Reporte de daño durante el check-out (Priority: P1)
Como estudiante, al momento de realizar el check-out de un recurso, quiero poder reportar que este presenta un daño, para dejar constancia y que se gestione el cobro correspondiente.

**Why this priority:** Es el momento natural en que se detecta el daño con mayor frecuencia; capturarlo en el mismo flujo de check-out asegura trazabilidad inmediata entre la utilización y el daño.

**Independent Test:** Puede probarse de forma independiente simulando el check-out de un recurso e indicando que presenta un daño, verificando que el sistema registra el reporte y dispara el cobro correspondiente.

**Acceptance Scenarios:**

**Scenario: Reporte de daño en el momento del check-out**
- **Given** un estudiante realizando el check-out (`«extend»`) de un recurso que tiene actualmente en uso.
- **When** indica que el recurso presenta un daño y describe la novedad.
- **Then** el sistema registra el reporte de daño vinculado a la utilización, cambia el estado del recurso y dispara obligatoriamente (`«include»`) el caso de uso *Generar cobro por daño o reposición*.

### User Story 2 - Reporte de daño fuera del flujo de check-out (Priority: P2)
Como encargado de inventario, quiero poder reportar un daño detectado en un recurso de forma independiente al check-out, para dejar constancia aun cuando el estudiante no lo haya reportado.

**Why this priority:** Cubre los casos en que el daño se detecta después de la devolución (por ejemplo, en una revisión periódica), lo cual es menos frecuente pero igualmente necesario para mantener el inventario confiable.

**Independent Test:** Puede probarse de forma independiente registrando un reporte de daño sobre un recurso ya devuelto, sin pasar por el flujo de check-out, y verificando que el sistema lo registra y dispara el cobro correspondiente.

**Acceptance Scenarios:**

**Scenario: Reporte de daño detectado en revisión de inventario**
- **Given** un recurso previamente devuelto, sin daño reportado en su check-out.
- **When** un encargado de inventario reporta un daño detectado posteriormente.
- **Then** el sistema registra el reporte vinculado a la última utilización identificable del recurso, cambia el estado del recurso y dispara el caso de uso *Generar cobro por daño o reposición*.

## Edge Cases

- **Recurso no identificado:** el sistema debe rechazar el reporte e informar que no fue posible identificar el recurso asociado a la novedad.
- **Reporte duplicado sobre la misma utilización:** el sistema debe impedir un segundo reporte de daño para la misma utilización y evitar generar cobros duplicados.
- **Reporte sin descripción o evidencia mínima:** el sistema debe rechazar el reporte e informar que se requiere una descripción del daño para poder registrarlo.
- **Daño reportado sobre un recurso ya reasignado a otro estudiante:** el sistema debe vincular el reporte a la última utilización identificable antes de la reasignación, o rechazarlo si no puede determinarse con certeza, informando el motivo.
- **Error al registrar el reporte:** el sistema debe informar que el reporte no pudo registrarse y evitar dejar un registro incompleto.
- **Error al cambiar el estado del recurso:** el sistema debe informar que el reporte se registró pero el estado del recurso no pudo actualizarse, marcando la inconsistencia para revisión.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe permitir reportar un daño técnico como parte del flujo de *Realizar check-out* (`«extend»`).
- **FR-002:** El sistema debe permitir reportar un daño técnico de forma independiente al check-out, por parte de un encargado de inventario.
- **FR-003:** El sistema debe identificar el recurso asociado al reporte y, cuando aplique, la utilización y el estudiante correspondientes.
- **FR-004:** El sistema debe registrar la descripción del daño y, si está disponible, evidencia asociada (por ejemplo, fotografías).
- **FR-005:** El sistema debe disparar obligatoriamente (`«include»`) el caso de uso *Generar cobro por daño o reposición* una vez registrado correctamente el reporte.
- **FR-006:** El sistema debe actualizar el estado del recurso (por ejemplo, a "En mantenimiento" o "Fuera de servicio") cuando se registre un daño.
- **FR-007:** El sistema debe impedir el registro de un reporte de daño duplicado para la misma utilización.
- **FR-008:** El sistema debe informar cuando el recurso no pueda identificarse o cuando falte la descripción mínima requerida.

## Key Entities
- **Recurso:** Activo universitario sobre el cual se reporta el daño; su estado cambia como consecuencia del reporte.
- **Reporte_Daño:** Registro de la novedad técnica, con descripción, evidencia, fecha, recurso y utilización asociados, y quien reporta.
- **Utilización:** Relación entre el estudiante y el recurso durante la cual (o después de la cual) se detecta el daño.
- **Estudiante:** Referenciado cuando el daño puede vincularse a una utilización específica.

## Success Criteria

### Measurable Outcomes
- **SC-001:** El 100% de los reportes de daño registrados correctamente deben disparar el caso de uso *Generar cobro por daño o reposición*.
- **SC-002:** El 100% de los recursos con un daño reportado deben quedar en un estado distinto de "Disponible" inmediatamente después del reporte.
- **SC-003:** El 0% de los reportes de daño deben duplicarse para la misma utilización.
- **SC-004:** El sistema debe registrar el reporte de daño en menos de 2 segundos desde que se recibe la información.
