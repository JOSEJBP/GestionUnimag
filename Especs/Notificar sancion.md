# Feature Specification: Notificar Sanción

**Created:** 2026-09-04
**Nota:** este caso de uso no existía como burbuja independiente en el diagrama original; se agregó porque la notificación es un flujo con su propio disparador (`«include»` desde *Calcular penalización*) y su propio destino (Módulo 1 o Módulo 2), y mezclarlo dentro de *Calcular penalización* generaba requisitos duplicados entre specs.

## User Scenarios & Testing

### User Story 1 - Notificación de sanción al estudiante (Priority: P2)
Como sistema, después de que *Calcular penalización* registre correctamente una sanción, debo notificar al estudiante a través del Módulo 1 o 2, informándole el motivo, los días de suspensión y su nuevo score de confianza.

**Why this priority:** La notificación es esencial para que el estudiante conozca su situación y pueda tomar acciones correctivas, pero puede implementarse después de que el cálculo base de la sanción esté funcionando.

**Independent Test:** Puede probarse de forma independiente invocando el caso de uso con los datos de una sanción ya calculada y verificando que el estudiante recibe la notificación con todos los detalles, sin depender de cómo se calculó la sanción.

**Acceptance Scenarios:**

**Scenario: Notificación de primera infracción**
- **Given** un estudiante recibe su primera sanción por retraso (1 día de suspensión, score resultante 45).
- **When** el sistema dispara la notificación tras registrar la penalización.
- **Then** el estudiante recibe una notificación con: "Motivo: Retraso en entrega. Sanción: 1 día de suspensión. Score de confianza: 45 puntos (-5)".

**Scenario: Notificación de sanción grave**
- **Given** un estudiante recibe su quinta sanción (7 días de suspensión, score resultante 25).
- **When** el sistema dispara la notificación.
- **Then** el estudiante recibe una notificación con: "Motivo: Retraso en entrega. Sanción: 7 días de suspensión. Score de confianza: 25 puntos (-5). Esta es su quinta infracción en 30 días".

### User Story 2 - Notificación de bloqueo (Priority: P2)
Como sistema, cuando el score de un estudiante llegue a 0 y se dispare el bloqueo, debo notificarle que ha sido bloqueado y las condiciones para su reactivación.

**Why this priority:** Sin esta notificación, el estudiante se entera del bloqueo solo al intentar un nuevo préstamo, lo cual empeora la experiencia y genera reclamos evitables.

**Independent Test:** Puede probarse simulando un bloqueo automático y verificando que el estudiante recibe una notificación indicando el bloqueo y cómo recuperar su score.

**Acceptance Scenarios:**

**Scenario: Notificación de bloqueo automático**
- **Given** el caso de uso *Bloquear usuario* bloquea a un estudiante por score en 0.
- **When** el sistema dispara la notificación correspondiente.
- **Then** el estudiante recibe una notificación indicando que está bloqueado, el motivo (score de confianza en 0) y las condiciones para recuperar su score.

## Edge Cases

- **Módulo 1 y Módulo 2 no disponibles simultáneamente:** el sistema debe intentar el envío por el canal disponible y, si ninguno responde, encolar la notificación para reintento.
- **Notificación duplicada para la misma sanción:** el sistema debe evitar enviar dos veces la misma notificación ante reintentos automáticos.
- **Datos incompletos para construir el mensaje (por ejemplo, falta el score resultante):** el sistema debe registrar el error y no enviar una notificación con información incompleta o inconsistente.
- **Error de entrega en ambos módulos:** el sistema debe registrar el fallo de notificación sin bloquear ni revertir la sanción ya calculada.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe notificar al estudiante, a través del Módulo 1 o 2, cuando *Calcular penalización* registre correctamente una sanción.
- **FR-002:** El sistema debe notificar al estudiante cuando *Bloquear usuario* aplique un bloqueo automático por score en 0.
- **FR-003:** La notificación de sanción debe incluir: motivo, días de suspensión y score de confianza resultante.
- **FR-004:** La notificación de bloqueo debe incluir: motivo del bloqueo y condiciones para su reactivación.
- **FR-005:** El sistema debe reintentar el envío de la notificación por el canal alternativo (Módulo 1 o 2) si el canal principal no está disponible.
- **FR-006:** El sistema debe evitar el envío duplicado de una misma notificación ante reintentos.
- **FR-007:** El sistema debe registrar el fallo de envío sin afectar el estado de la sanción o del bloqueo ya aplicados.

## Key Entities
- **Notificacion:** Representa el mensaje enviado al estudiante. Contiene tipo (Sanción / Bloqueo), canal utilizado (Módulo 1 / Módulo 2), contenido, fecha de envío y estado (Enviada, Fallida, Reintentando).
- **Estudiante:** Destinatario de la notificación.
- **Historial_Sanciones / Score_Confianza:** Fuentes de datos para construir el contenido de la notificación.

## Success Criteria

### Measurable Outcomes
- **SC-001:** El sistema debe notificar al estudiante en menos de 5 segundos después de registrada la sanción o el bloqueo.
- **SC-002:** El 100% de las notificaciones enviadas deben incluir todos los campos obligatorios según el tipo (sanción o bloqueo).
- **SC-003:** El 0% de las notificaciones deben duplicarse ante reintentos automáticos.
- **SC-004:** El 100% de los fallos de entrega deben quedar registrados sin afectar el estado de la sanción o el bloqueo asociados.
