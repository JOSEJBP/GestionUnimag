# Feature Specification: Generar Cobro por Daño o Reposición

**Created:** 2026-09-04

## User Scenarios & Testing

### User Story 1 - Generación automática del cobro (Priority: P1)
Como sistema, cuando se registre un reporte de daño técnico, debo generar automáticamente el cobro correspondiente al costo de reparación o reposición del recurso afectado.

**Why this priority:** Es el efecto directo y esperado de todo reporte de daño; sin esta generación automática, el daño quedaría registrado sin ninguna consecuencia económica para el estudiante responsable.

**Independent Test:** Puede probarse de forma independiente invocando el caso de uso con un reporte de daño ya registrado y un recurso con costo de reposición configurado, verificando que se genera un cobro con el monto correcto.

**Acceptance Scenarios:**

**Scenario: Cobro generado con costo de reposición configurado**
- **Given** un reporte de daño técnico registrado sobre un recurso cuyo costo de reposición está configurado en el catálogo.
- **When** el caso de uso *Reportar novedad técnica* dispara la generación del cobro (`«include»`).
- **Then** el sistema genera un cobro en estado "Pendiente", con el monto correspondiente al costo de reposición del recurso, vinculado al estudiante, al recurso y al reporte de daño.

**Scenario: Intento de generar cobro para un recurso sin costo configurado**
- **Given** un reporte de daño sobre un recurso cuyo costo de reposición no está configurado en el catálogo.
- **When** el sistema intenta generar el cobro.
- **Then** el sistema rechaza la generación automática e informa que se requiere definir manualmente el monto antes de continuar.

### User Story 2 - Gestión y aprobación del cobro (Priority: P2)
Como dirección universitaria, quiero poder revisar, ajustar, exonerar o aprobar un cobro generado, para tener control sobre casos particulares antes de que se le cobre al estudiante.

**Why this priority:** No todo daño reportado debe traducirse en un cobro definitivo sin revisión (por ejemplo, desgaste normal o error del sistema); la dirección universitaria necesita poder intervenir antes de la notificación final al estudiante.

**Independent Test:** Puede probarse con un cobro en estado "Pendiente" que la dirección universitaria revisa y decide aprobar, ajustar el monto, o exonerar, verificando que el estado y el monto se actualizan correctamente.

**Acceptance Scenarios:**

**Scenario: Aprobación de un cobro**
- **Given** un cobro en estado "Pendiente".
- **When** la dirección universitaria lo aprueba.
- **Then** el estado del cobro cambia a "Aprobado" y queda disponible para ser notificado y pagado por el estudiante.

**Scenario: Exoneración de un cobro**
- **Given** un cobro en estado "Pendiente".
- **When** la dirección universitaria decide exonerarlo, indicando un motivo.
- **Then** el estado del cobro cambia a "Exonerado" y el estudiante no debe pagar ningún monto.

### User Story 3 - Consulta y pago del cobro por el estudiante (Priority: P2)
Como estudiante, quiero poder consultar el estado de mis cobros pendientes y pagarlos a través del Módulo 2, para resolver mi situación con los recursos utilizados.

**Why this priority:** Es el cierre natural del flujo de cobro; el estudiante necesita visibilidad y un canal de pago concreto.

**Independent Test:** Puede probarse con un cobro en estado "Aprobado" que el estudiante consulta y paga a través del Módulo 2, verificando que el estado cambia a "Pagado".

**Acceptance Scenarios:**

**Scenario: Consulta de cobros pendientes**
- **Given** un estudiante con al menos un cobro en estado "Aprobado".
- **When** consulta sus cobros a través del Módulo 2.
- **Then** el sistema muestra el detalle del cobro (motivo, monto, recurso asociado, fecha).

**Scenario: Pago de un cobro**
- **Given** un cobro en estado "Aprobado".
- **When** el estudiante realiza el pago a través del Módulo 2.
- **Then** el sistema actualiza el estado del cobro a "Pagado" y lo registra con fecha de pago.

### User Story 4 - Consulta de tarjetas de cobro (Priority: P1)
Como estudiante, quiero ver los cobros pendientes asociados al deterioro o rotura de materiales, para estar al tanto de mis deudas económicas.

**Why this priority:** La deuda debe ser visible con suficiente detalle para que el estudiante pueda identificarla y resolverla.

**Independent Test:** Puede probarse con dos cobros aprobados, verificando que cada tarjeta muestra su referencia, ítem y monto en moneda local.

**Acceptance Scenarios:**

**Scenario: Visualización del detalle del cobro**
- **Given** un estudiante con uno o más cobros pendientes.
- **When** consulta la sección de cobros de su perfil.
- **Then** el sistema muestra una tarjeta por cobro con referencia única, nombre del ítem dañado, monto en CLP y estado.

**Scenario: Sin cobros pendientes**
- **Given** un estudiante sin cobros pendientes por daños.
- **When** consulta la sección de cobros.
- **Then** el sistema muestra "No tienes deudas o cobros pendientes.".

### User Story 5 - Pago o apelación desde el perfil (Priority: P1)
Como estudiante, quiero pagar un cobro directamente desde la plataforma o apelar la penalización si considero que no fui responsable del daño.

**Why this priority:** El perfil debe ofrecer una acción para saldar la deuda y otra para solicitar revisión administrativa.

**Independent Test:** Puede probarse seleccionando "Pagar" y "Apelar" desde una tarjeta de cobro, verificando que cada acción inicia su flujo correspondiente.

**Acceptance Scenarios:**

**Scenario: Inicio del pago**
- **Given** un cobro aprobado y pendiente de pago.
- **When** el estudiante selecciona "Pagar".
- **Then** el sistema redirige a la pasarela de pago y conserva la referencia del cobro.

**Scenario: Inicio de la apelación**
- **Given** un cobro pendiente o aprobado que no tiene una apelación en revisión.
- **When** el estudiante selecciona "Apelar".
- **Then** el sistema abre un formulario o modal de reclamo para enviar la justificación y la evidencia a administración.

**Scenario: Apelación en revisión**
- **Given** un cobro con una apelación en estado "Pendiente".
- **When** el estudiante consulta la tarjeta.
- **Then** el sistema muestra la etiqueta "Apelación Pendiente" y mantiene visible la acción de pago mientras la apelación esté en revisión.

## Edge Cases

- **Recurso sin costo de reposición configurado:** el sistema debe rechazar la generación automática e informar que se requiere definir manualmente el monto.
- **Cobro ya generado para el mismo reporte de daño:** el sistema debe impedir la generación de un cobro duplicado para el mismo reporte.
- **Estudiante apela un cobro ya aprobado:** el sistema debe permitir registrar la apelación, cambiando el estado a "Apelado" y suspendiendo temporalmente la exigencia de pago hasta su resolución.
- **Apelación pendiente:** el sistema debe mostrar la etiqueta "Apelación Pendiente" y evitar una segunda apelación activa para el mismo cobro.
- **Error al generar el cobro:** el sistema debe informar que el reporte de daño se registró pero el cobro no pudo generarse, marcando la inconsistencia para revisión.
- **Intento de pago sobre un cobro exonerado o ya pagado:** el sistema debe rechazar el intento e informar el estado actual del cobro.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe generar automáticamente un cobro cuando el caso de uso *Reportar novedad técnica* dispare la generación (`«include»`).
- **FR-002:** El sistema debe calcular el monto del cobro a partir del costo de reposición configurado para el recurso afectado.
- **FR-003:** El sistema debe rechazar la generación automática e informar cuando el recurso no tenga un costo de reposición configurado.
- **FR-004:** El sistema debe vincular el cobro generado con el estudiante, el recurso y el reporte de daño correspondientes.
- **FR-005:** El sistema debe permitir a la dirección universitaria aprobar, ajustar el monto, o exonerar un cobro en estado "Pendiente".
- **FR-006:** El sistema debe permitir al estudiante consultar el estado y detalle de sus cobros a través del Módulo 2.
- **FR-007:** El sistema debe permitir al estudiante pagar un cobro en estado "Aprobado" a través del Módulo 2.
- **FR-008:** El sistema debe permitir al estudiante apelar un cobro, cambiando su estado a "Apelado" y suspendiendo la exigencia de pago hasta su resolución.
- **FR-009:** El sistema debe impedir la generación de un cobro duplicado para el mismo reporte de daño.
- **FR-010:** El sistema debe rechazar intentos de pago sobre cobros en estado "Exonerado" o "Pagado".
- **FR-011:** La interfaz debe mostrar cada cobro pendiente en una tarjeta individual con referencia, ítem dañado, monto en CLP y estado.
- **FR-012:** La interfaz debe mostrar "No tienes deudas o cobros pendientes." cuando el estudiante no tenga cobros activos.
- **FR-013:** El botón "Pagar" debe redirigir a la pasarela de pago conservando la referencia única del cobro.
- **FR-014:** El botón "Apelar" debe abrir un formulario o modal de reclamo para revisión administrativa.
- **FR-015:** El sistema debe mostrar "Apelación Pendiente" en la tarjeta mientras la apelación esté en revisión y mantener habilitado el pago.

## Key Entities
- **Cobro:** Representa el monto adeudado por daño o reposición. Contiene motivo, monto, recurso, estudiante, reporte de daño asociado, y estado (Pendiente, Aprobado, Exonerado, Apelado, Pagado).
- **Catalogo_Costos_Reposicion:** Configuración del costo de reposición o reparación por tipo de recurso.
- **Reporte_Daño:** Origen del cobro generado.
- **Estudiante:** Responsable del cobro, con visibilidad y capacidad de pago a través del Módulo 2.
- **Dirección universitaria:** Actor con capacidad de aprobar, ajustar o exonerar el cobro.
- **Apelacion_Cobro:** Solicitud de revisión administrativa vinculada a un cobro, con estado, justificación y evidencia.

## Integración con Módulos Externos

| Módulo | Tipo de relación | Justificación |
|---|---|---|
| Módulo 2 | **Proactivo (REST)** | La consulta del estado del cobro y el pago requieren una respuesta inmediata y consistente (el estudiante necesita ver el resultado de su pago al instante, y no se puede dejar en un estado ambiguo). Se implementan como llamadas síncronas de solicitud/respuesta. |

## Success Criteria

### Measurable Outcomes
- **SC-001:** El 100% de los reportes de daño sobre recursos con costo configurado deben generar un cobro automáticamente.
- **SC-002:** El 100% de los recursos sin costo configurado deben rechazar la generación automática e informar el motivo.
- **SC-003:** El 0% de los reportes de daño deben generar más de un cobro asociado.
- **SC-004:** El 100% de los cobros en estado "Aprobado" deben quedar disponibles para consulta y pago por el estudiante en menos de 5 segundos.
- **SC-005:** El 100% de los intentos de pago sobre cobros "Exonerado" o "Pagado" deben ser rechazados.
