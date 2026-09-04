# Feature Specification: Notificar Sanción

**Created:** 2026-09-04
**Revisado:** 2026-09-04 — se amplió el alcance de este caso de uso: ya no notifica solo penalizaciones y bloqueos, sino **cualquier** cambio de score de confianza (reducciones, aumentos por check-out, recuperación por período) y el bloqueo/desbloqueo asociado. El disparador principal pasa a ser *Actualizar score de confianza* (`«include»`), en lugar de *Calcular penalización* directamente.

## User Scenarios & Testing

### User Story 1 - Notificación de reducción de score (Priority: P2)
Como sistema, después de que *Actualizar score de confianza* registre una reducción, debo notificar al estudiante a través del Módulo 1 o 2, informándole el motivo, los puntos restados y su nuevo score.

**Why this priority:** La notificación es esencial para que el estudiante conozca su situación y pueda tomar acciones correctivas, pero puede implementarse después de que el cálculo base de la sanción esté funcionando.

**Independent Test:** Puede probarse de forma independiente invocando el caso de uso con los datos de una reducción ya registrada y verificando que el estudiante recibe la notificación con todos los detalles.

**Acceptance Scenarios:**

**Scenario: Notificación de reducción por infracción**
- **Given** un estudiante cuyo score se redujo de 50 a 45 puntos por "Infracción por retraso".
- **When** el sistema dispara la notificación tras registrar el cambio.
- **Then** el estudiante recibe una notificación con: "Motivo: Infracción por retraso. Cambio: -5 puntos. Score de confianza: 45 puntos".

**Scenario: Notificación de sanción con días de suspensión**
- **Given** una penalización que además de reducir el score aplica días de suspensión (según *Calcular penalización*).
- **When** el sistema dispara la notificación.
- **Then** el estudiante recibe una notificación con el motivo, los días de suspensión y el nuevo score, en un solo mensaje.

### User Story 2 - Notificación de aumento de score (Priority: P2)
Como sistema, después de que *Actualizar score de confianza* registre un aumento (por check-out a tiempo o por período sin infracciones), debo notificar al estudiante con el motivo, los puntos sumados y su nuevo score.

**Why this priority:** El refuerzo positivo también debe comunicarse para que el estudiante entienda que su buen comportamiento tiene un efecto visible en su score.

**Independent Test:** Puede probarse invocando el caso de uso con los datos de un aumento ya registrado y verificando que el estudiante recibe la notificación correspondiente.

**Acceptance Scenarios:**

**Scenario: Notificación de aumento por check-out a tiempo**
- **Given** un estudiante cuyo score aumentó de 40 a 42 puntos por "Entrega a tiempo".
- **When** el sistema dispara la notificación.
- **Then** el estudiante recibe una notificación con: "Motivo: Entrega a tiempo. Cambio: +2 puntos. Score de confianza: 42 puntos".

**Scenario: Notificación de recuperación por período sin infracciones**
- **Given** un estudiante cuyo score aumentó de 30 a 35 puntos por "30 días sin infracciones".
- **When** el sistema dispara la notificación.
- **Then** el estudiante recibe una notificación con: "Motivo: 30 días sin infracciones. Cambio: +5 puntos. Score de confianza: 35 puntos".

### User Story 3 - Notificación de bloqueo y desbloqueo (Priority: P2)
Como sistema, cuando *Bloquear usuario* aplique o levante un bloqueo por score, debo notificar al estudiante con el motivo y, en caso de bloqueo, las condiciones para su reactivación.

**Why this priority:** Sin esta notificación, el estudiante se entera del bloqueo solo al intentar un nuevo préstamo, lo cual empeora la experiencia y genera reclamos evitables.

**Independent Test:** Puede probarse simulando un bloqueo o desbloqueo automático y verificando que el estudiante recibe la notificación correspondiente.

**Acceptance Scenarios:**

**Scenario: Notificación de bloqueo automático**
- **Given** *Bloquear usuario* bloquea a un estudiante por score en 0.
- **When** el sistema dispara la notificación.
- **Then** el estudiante recibe una notificación indicando el bloqueo, el motivo (score de confianza en 0) y las condiciones para recuperar su score.

**Scenario: Notificación de desbloqueo**
- **Given** *Bloquear usuario* desbloquea a un estudiante cuyo score subió por encima de 0.
- **When** el sistema dispara la notificación.
- **Then** el estudiante recibe una notificación indicando que su cuenta fue reactivada y su score actual.

## Edge Cases

- **Módulo 1 y Módulo 2 no disponibles simultáneamente:** el sistema debe intentar el envío por el canal disponible y, si ninguno responde, encolar la notificación para reintento.
- **Notificación duplicada para el mismo cambio de score:** el sistema debe evitar enviar dos veces la misma notificación ante reintentos automáticos.
- **Datos incompletos para construir el mensaje (por ejemplo, falta el score resultante):** el sistema debe registrar el error y no enviar una notificación con información incompleta o inconsistente.
- **Varios cambios de score en un corto período (ej. reducción y luego bloqueo inmediato):** el sistema puede agrupar la información en un solo mensaje o enviar mensajes separados, siempre evitando que el estudiante reciba información contradictoria entre ellos.
- **Error de entrega en ambos módulos:** el sistema debe registrar el fallo de notificación sin bloquear ni revertir el cambio de score, la sanción o el bloqueo ya aplicados.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe notificar al estudiante, a través del Módulo 1 o 2, cada vez que *Actualizar score de confianza* registre un cambio de score (reducción, aumento por check-out, o recuperación por período).
- **FR-002:** El sistema debe notificar al estudiante cuando *Bloquear usuario* aplique un bloqueo automático por score en 0.
- **FR-003:** El sistema debe notificar al estudiante cuando *Bloquear usuario* levante un bloqueo por recuperación de score.
- **FR-004:** La notificación de una reducción debe incluir: motivo, puntos restados, y score de confianza resultante; si la reducción proviene de una penalización con días de suspensión, debe incluir también esos días.
- **FR-005:** La notificación de un aumento debe incluir: motivo, puntos sumados, y score de confianza resultante.
- **FR-006:** La notificación de bloqueo debe incluir: motivo del bloqueo y condiciones para su reactivación.
- **FR-007:** La notificación de desbloqueo debe incluir: motivo y score de confianza actual.
- **FR-008:** El sistema debe reintentar el envío de la notificación por el canal alternativo (Módulo 1 o 2) si el canal principal no está disponible.
- **FR-009:** El sistema debe evitar el envío duplicado de una misma notificación ante reintentos.
- **FR-010:** El sistema debe registrar el fallo de envío sin afectar el estado del cambio de score, la sanción o el bloqueo asociados.

## Key Entities
- **Notificacion:** Representa el mensaje enviado al estudiante. Contiene tipo (Reducción / Aumento / Bloqueo / Desbloqueo), canal utilizado (Módulo 1 / Módulo 2), contenido, fecha de envío y estado (Enviada, Fallida, Reintentando).
- **Estudiante:** Destinatario de la notificación.
- **Historial_Score:** Fuente de datos para construir el contenido de las notificaciones de reducción y aumento.
- **Historial_Bloqueos:** Fuente de datos para construir el contenido de las notificaciones de bloqueo y desbloqueo.

## Success Criteria

### Measurable Outcomes
- **SC-001:** El sistema debe notificar al estudiante en menos de 5 segundos después de registrado el cambio de score, la sanción o el bloqueo.
- **SC-002:** El 100% de las notificaciones enviadas deben incluir todos los campos obligatorios según su tipo.
- **SC-003:** El 0% de las notificaciones deben duplicarse ante reintentos automáticos.
- **SC-004:** El 100% de los fallos de entrega deben quedar registrados sin afectar el estado del evento que las originó.
