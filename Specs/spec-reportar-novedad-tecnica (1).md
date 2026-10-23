# Feature Specification: Reportar Novedad Técnica (Daño)

**Created:** 2026-10-17
**Updated:** 2026-10-22 (integración alineada con los contratos de M1; gestión de cobros por el personal administrativo)

> El cobro que genera cada reporte está definido en `reglas-negocio-m3.md` (RN-COB). Las reglas de integración con M1 están en RN-INT-05 y RN-INT-07.

## User Scenarios & Testing

### User Story 1 - Reporte de daño durante el check-out (Priority: P1)
Como estudiante, al registrar el check-out de un recurso o espacio, quiero poder reportar que presenta un daño, para dejar constancia inmediata y que el inventario se actualice.

**Why this priority:** El check-out es el momento en que el daño se detecta con más frecuencia; capturarlo en el mismo flujo asegura trazabilidad directa entre el uso y el daño.

**Independent Test:** Simular el check-out de un recurso indicando un daño con descripción, y verificar que M3 registra el reporte, publica `reporte.danos` a M1 y genera el cobro.

**Acceptance Scenarios:**

**Scenario: Reporte de daño en el check-out**
- **Given** un estudiante registrando el check-out (`«extend»`) de un recurso o espacio que tenía en uso.
- **When** indica que presenta un daño y escribe la descripción.
- **Then** M3 registra el reporte vinculado a la utilización, al recurso y al estudiante, con estado "Pendiente"; publica la novedad a M1 como `reporte.danos` (que reemplaza a `reserva.finalizada` en ese check-out, RN-INT-05) y genera automáticamente el cobro al estudiante responsable (RN-COB-01).

**Scenario: Reporte con evidencia**
- **Given** un estudiante que reporta un daño en el check-out.
- **When** adjunta fotografías además de la descripción.
- **Then** M3 registra el reporte con la evidencia asociada.

### User Story 2 - Reporte de daño fuera del flujo de check-out (Priority: P2)
Como personal administrativo, quiero reportar un daño detectado en un recurso o espacio de forma independiente al check-out, para dejar constancia aunque el estudiante no lo haya reportado.

**Why this priority:** Cubre los daños detectados después de la devolución, por ejemplo en una revisión de inventario; es menos frecuente pero necesario para mantener el inventario confiable.

**Independent Test:** Registrar un reporte sobre un recurso ya devuelto, sin pasar por el check-out, y verificar que se registra, se publica `reporte.danos` a M1 y se intenta generar el cobro.

**Acceptance Scenarios:**

**Scenario: Daño detectado en revisión posterior**
- **Given** un recurso devuelto sin daño reportado en su check-out.
- **When** el personal administrativo reporta un daño con su descripción.
- **Then** M3 registra el reporte vinculado a la última utilización identificable, con estado "Pendiente", publica `reporte.danos` a M1 y genera automáticamente el cobro al estudiante de esa utilización (RN-COB-01).

**Scenario: Daño sin utilización identificable**
- **Given** un recurso cuya última utilización no puede determinarse con certeza.
- **When** el personal administrativo reporta el daño.
- **Then** M3 registra el reporte sin estudiante asociado y lo publica a M1; el cobro no se genera y el reporte queda marcado para revisión.

### User Story 3 - Responder a M1 si un recurso tiene novedades pendientes (Priority: P2)
Como Módulo 1, quiero preguntar a M3 si un recurso tiene novedades técnicas sin resolver antes de una transición crítica de estado, para no liberar un recurso dañado.

**Why this priority:** M1 no ejecuta la transición si no puede validar esta condición; sin esta respuesta M1 queda bloqueado para liberar recursos que pasaron por mantenimiento.

**Independent Test:** Con un recurso con un reporte pendiente y otro sin reportes, consultar `GET /api/v1/novedades/pendientes?recursoId=` y verificar la respuesta de cada uno.

**Acceptance Scenarios:**

**Scenario: Recurso con novedad pendiente**
- **Given** un recurso con un reporte de daño en estado "Pendiente".
- **When** M1 consulta las novedades pendientes del recurso.
- **Then** M3 responde `tieneNovedadPendiente: true` (RN-INT-07).

**Scenario: Recurso sin novedades pendientes**
- **Given** un recurso sin reportes de daño, o con todos "Resueltos".
- **When** M1 consulta las novedades pendientes del recurso.
- **Then** M3 responde `tieneNovedadPendiente: false`.

**Scenario: Novedad que se resuelve**
- **Given** un recurso con un reporte "Pendiente".
- **When** M1 informa por `estado.actualizado` que el recurso salió de mantenimiento.
- **Then** M3 marca como "Resuelta" las novedades pendientes de ese recurso y las siguientes consultas de M1 responden `false`.

### User Story 4 - Registrar el pago o la anulación de un cobro (Priority: P2)
Como personal administrativo, quiero registrar que un cobro fue pagado o anularlo con un motivo, para mantener al día el estado de los cobros que M3 genera por daños y por préstamos perdidos.

**Why this priority:** M3 genera los cobros pero el dinero se recauda fuera del sistema; sin este registro todos quedarían "Pendiente" para siempre y no habría forma de corregir un cobro injusto, por ejemplo uno por un daño que ya existía.

**Independent Test:** Con un cobro "Pendiente", marcarlo como pagado, y con otro anularlo con motivo, y verificar el estado, la auditoría y lo que ve el estudiante.

**Acceptance Scenarios:**

**Scenario: Registro de pago**
- **Given** un cobro en estado "Pendiente".
- **When** el personal administrativo registra que fue pagado.
- **Then** el cobro pasa a "Pagado", y queda en la auditoría quién lo registró y cuándo.

**Scenario: Anulación**
- **Given** un cobro en estado "Pendiente".
- **When** el personal administrativo lo anula e indica el motivo (por ejemplo, daño preexistente).
- **Then** el cobro pasa a "Anulado" con el motivo y la auditoría; el estudiante lo ve así en su cumplimiento.

**Scenario: Anulación sin motivo**
- **Given** un cobro "Pendiente".
- **When** el personal administrativo intenta anularlo sin motivo.
- **Then** el sistema rechaza la operación e informa que el motivo es obligatorio.

**Scenario: Cobro ya resuelto**
- **Given** un cobro "Pagado" o "Anulado".
- **When** se intenta cambiar su estado.
- **Then** el sistema rechaza la operación e informa que el cobro ya fue resuelto.

## Edge Cases

- **Usuario sin permisos para gestionar cobros:** el sistema rechaza la operación.
- **Recurso desconocido en la consulta de M1:** M3 responde `tieneNovedadPendiente: false`, porque no tiene reportes de ese recurso.
- **Consulta de M1 con M3 degradado:** M3 debe responder en pocos segundos; si no lo hace, M1 trata el caso como fallo de validación y no ejecuta la transición.
- **Daño independiente del check-out:** se publica a M1 pero no se informa a M2, porque M2 no acepta datos de daño.

- **Recurso no identificado:** el sistema debe rechazar el reporte e informar que no fue posible identificar el recurso.
- **Reporte sin descripción:** el sistema debe rechazar el reporte e informar que la descripción del daño es obligatoria. Durante el check-out, se pide completarla antes de confirmar.
- **Reporte duplicado sobre la misma utilización:** el sistema debe impedir un segundo reporte y evitar notificar dos veces a M1.
- **Recurso ya reasignado a otro estudiante:** el reporte se vincula a la última utilización anterior a la reasignación, o se registra sin estudiante si no puede determinarse con certeza.
- **Evidencia inválida:** si el archivo supera el tamaño máximo o no es de un tipo permitido, el sistema debe rechazar esa evidencia sin perder el resto del reporte.
- **Error al registrar el reporte:** el sistema debe informar que no pudo registrarse y evitar dejar un registro incompleto.
- **Falla al publicar la novedad a M1:** el reporte se conserva junto con el evento pendiente para reintento; publicar al broker no confirma que M1 procesó el daño.
- **Falla al generar el cobro:** el reporte se mantiene registrado y la inconsistencia queda marcada para revisión.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe permitir al estudiante reportar un daño técnico como parte del flujo *Registrar check-out* (`«extend»`).
- **FR-002:** El sistema debe permitir al personal administrativo reportar un daño técnico de forma independiente del check-out.
- **FR-003:** El sistema debe identificar el recurso o espacio afectado y, cuando sea posible, la utilización y el estudiante correspondientes.
- **FR-004:** El sistema debe registrar la descripción obligatoria del daño, la evidencia opcional, la fecha, quién reporta y una instantánea de los datos del recurso (placa o ID, tipo, ubicación).
- **FR-005:** El sistema debe publicar a M1 el evento `reporte.danos` en `unimag.m3.uso-novedades.v1` (clave: identificador del recurso) para que M1 decida y aplique el estado del recurso (por ejemplo, "En Mantenimiento"), según RN-INT-05.
- **FR-006:** El sistema debe generar automáticamente el cobro por daño o reposición una vez registrado el reporte, y guardar en el reporte el resultado del cobro: Generado, Sin costo, Sin responsable o Error (RN-COB-01 a RN-COB-05).
- **FR-007:** El sistema debe impedir el registro de más de un reporte de daño por utilización.
- **FR-008:** El sistema debe conservar el evento pendiente para reintento cuando falle su publicación hacia M1, sin duplicarlo.
- **FR-009:** El sistema debe informar el motivo cuando rechace un reporte: recurso no identificado, descripción faltante, duplicado o evidencia inválida.
- **FR-010:** El sistema debe aplicar este caso de uso tanto a recursos físicos como a espacios (por ejemplo, daño en equipamiento fijo de un salón).
- **FR-011:** El sistema debe exponer `GET /api/v1/novedades/pendientes?recursoId={id}` y responder `{ "recursoId", "tieneNovedadPendiente" }` según existan reportes del recurso en estado "Pendiente" (RN-INT-07).
- **FR-012:** El sistema debe marcar los reportes del recurso como "Resueltos" cuando M1 informe por `estado.actualizado` que salió de mantenimiento.
- **FR-013:** El sistema debe responder la consulta de novedades pendientes aunque Kafka no esté disponible.
- **FR-014:** El sistema debe contar el reporte de daño atribuido a un estudiante como una infracción para el conteo de recuperación de puntos (RN-SCO-05).
- **FR-015:** El sistema debe permitir al personal administrativo registrar el pago de un cobro "Pendiente", pasándolo a "Pagado" (RN-COB-06).
- **FR-016:** El sistema debe permitir al personal administrativo anular un cobro "Pendiente" con un motivo obligatorio, pasándolo a "Anulado" (RN-COB-06, RN-COB-09).
- **FR-017:** El sistema debe guardar en una auditoría quién cambió el estado de un cobro, cuándo y con qué motivo, y rechazar cambios sobre cobros ya "Pagado" o "Anulado".
- **FR-018:** El sistema debe tomar el monto de los cobros del catálogo de costos de M3, que mantiene el personal administrativo (RN-COB-10), y generar también el cobro por préstamo perdido al recibir `LoanDeclaredLost` de M2 (RN-COB-12).

## Key Entities
- **Reporte_Daño:** Registro de la novedad técnica, con descripción, evidencia, fecha, quién reporta, recurso, utilización y estudiante (opcional), e instantánea de datos del recurso. Incluye el estado de la novedad (Pendiente, Resuelta) y el estado del cobro asociado (Generado, Sin costo, Sin responsable, Error).
- **Recurso / Espacio:** Activo afectado; su estado lo decide M1 a partir de la novedad.
- **Utilización:** Relación entre el estudiante y el recurso durante o después de la cual se detecta el daño.
- **Estudiante:** Responsable cuando el daño puede vincularse a una utilización.
- **Cobro / Catalogo_Costos:** Consecuencia económica del reporte o de la pérdida de un préstamo, con estado Pendiente, Pagado o Anulado (ver `reglas-negocio-m3.md`).
- **Auditoria_Cobro:** Registro de cada cambio de estado de un cobro: quién, cuándo, estado anterior y nuevo, y motivo.
- **Personal administrativo:** Actor que reporta daños fuera del check-out.
- **Evento_Saliente:** Evento pendiente de publicación hacia M1.
- **Módulo 1 (actor externo):** Consume la novedad y consulta las novedades pendientes de un recurso (RN-INT-07).

## Integración con Módulos Externos

| Módulo | Tipo de relación | Justificación |
|---|---|---|
| Módulo 1 (Activos y Espacios) | **Reactivo (evento asíncrono)** | M3 publica `reporte.danos`; M1, dueño del inventario, decide el cambio de estado. El reporte no debe bloquearse esperando a M1. Si el contrato de M1 no define confirmación de consumo, M3 no puede medir cuándo se aplicó el estado. El payload del evento está en blanco en el contrato de M1 y se acuerda con ese equipo. |
| Módulo 1 (Activos y Espacios) | **Proactivo (REST, M1 → M3)** | Antes de una transición crítica, M1 consulta a M3 si hay novedades pendientes y necesita una respuesta inmediata (RN-INT-07). |
| Módulo 1 (Activos y Espacios) | **Reactivo (evento, M1 → M3)** | M3 consume `estado.actualizado` para resolver las novedades cuando el recurso sale de mantenimiento. [POR DEFINIR con M1: que ese evento cubra también este cambio.] |
| Módulo 2 (Reservas) | **Sin integración directa** | M2 no acepta datos de daño en el check-out; el daño va de M3 a M1. |

## Success Criteria

### Measurable Outcomes
- **SC-001:** El 100% de los reportes de daño registrados correctamente deben comunicarse a M1 o quedar pendientes de reintento.
- **SC-002:** El 0% de las utilizaciones deben tener más de un reporte de daño.
- **SC-003:** El 100% de los reportes registrados deben intentar generar su cobro y guardar el resultado en el reporte.
- **SC-004:** El 100% de los reportes sin descripción deben ser rechazados.
- **SC-005:** El sistema debe registrar el reporte en menos de 2 segundos desde que recibe la información.
- **SC-006:** El sistema debe responder la consulta de novedades pendientes de M1 en menos de 2 segundos.
- **SC-007:** El 100% de los recursos con un reporte "Pendiente" deben responder `tieneNovedadPendiente: true`.
- **SC-008:** El 100% de los cambios de estado de un cobro deben quedar en la auditoría, y el 0% de los cobros "Pagado" o "Anulado" deben poder modificarse.
