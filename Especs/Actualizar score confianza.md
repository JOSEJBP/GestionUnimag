# Feature Specification: Actualizar Score de Confianza

**Created:** 2026-09-03
**Revisado:** 2026-09-04 — se adoptaron los incrementos variables (+2 check-out, +5 recuperación, -5 reducción) en lugar de la regla de "solo múltiplos de 5"; se eliminó por ahora el flujo de "confirmar asistencia" (+3) al no existir como caso de uso en el diagrama; el bloqueo/desbloqueo efectivo se delega al caso de uso *Bloquear usuario* (`«extend»`); la notificación de cada cambio se delega a *Notificar sanción* (`«include»`), que ahora cubre cualquier cambio de score, no solo sanciones.

## User Scenarios & Testing

### User Story 1 - Reducción automática del score por infracción (Priority: P1)
Como sistema, cuando se ejecuta el flujo de *Calcular penalización* y la regla configurada indica que afecta el score de confianza (`afecta_score_confianza = TRUE`), debo reducir automáticamente el score del estudiante en 5 puntos y registrar este cambio en su historial.

**Why this priority:** Esta es la funcionalidad principal del caso de uso y la que más se ejecutará. Sin esta reducción automática, el score de confianza no cumpliría su propósito de reflejar el comportamiento del estudiante. Es crítica para el MVP.

**Independent Test:** Puede probarse de forma independiente simulando un estudiante con score de 50 puntos, disparando el flujo con una reducción de 5 puntos, y verificando que el score queda en 45 y se registra en el historial.

**Acceptance Scenarios:**

**Scenario: Reducción de score por primera infracción**
- **Given** un estudiante con score de confianza de 50 puntos.
- **When** el sistema recibe una solicitud de tipo "REDUCCION" desde *Calcular penalización*.
- **Then** el sistema reduce el score a 45 puntos, registra el cambio en el historial con motivo "Infracción por retraso", y dispara (`«include»`) *Notificar sanción*.

**Scenario: Reducción de score cuando queda en 0**
- **Given** un estudiante con score de confianza de 5 puntos.
- **When** el sistema recibe una solicitud de reducción de 5 puntos.
- **Then** el sistema reduce el score a 0 puntos, registra el cambio, y dispara (`«extend»`) *Bloquear usuario*.

**Scenario: Reducción de score cuando la regla NO lo permite**
- **Given** una regla de penalización con `afecta_score_confianza = FALSE`.
- **When** se calcula una penalización pero la regla indica que NO afecta el score.
- **Then** el sistema NO ejecuta este caso de uso (el `«extend»` desde *Calcular penalización* no se dispara).

### User Story 2 - Aumento automático del score por check-out a tiempo (Priority: P1)
Como sistema, cuando un estudiante realiza un check-out exitoso (dentro del tiempo límite establecido), debo aumentar automáticamente su score de confianza en 2 puntos, registrando este cambio en su historial.

**Why this priority:** El refuerzo positivo es tan importante como las penalizaciones y permite a los estudiantes recuperar su score demostrando buen comportamiento.

**Independent Test:** Puede probarse de forma independiente simulando un estudiante con score de 40 puntos que realiza un check-out exitoso, y verificando que el score aumenta a 42 puntos y se registra en el historial.

**Acceptance Scenarios:**

**Scenario: Aumento de score por entrega a tiempo**
- **Given** un estudiante con score de confianza de 40 puntos y un préstamo activo.
- **When** realiza check-out dentro del tiempo límite establecido.
- **Then** el sistema aumenta el score en 2 puntos (queda en 42), registra el cambio en el historial con motivo "Entrega a tiempo", y dispara (`«include»`) *Notificar sanción*.

**Scenario: Aumento de score cuando ya está en el máximo (50)**
- **Given** un estudiante con score de confianza de 50 puntos.
- **When** realiza un check-out exitoso.
- **Then** el sistema NO aumenta el score (ya está en el máximo), pero registra en el historial "Entrega a tiempo - Score ya en máximo".

### User Story 3 - Recuperación por período sin infracciones (Priority: P2)
Como sistema, cuando un estudiante completa 30 días consecutivos sin infracciones (retrasos o inasistencias), debo aumentar automáticamente su score en 5 puntos y notificarlo.

**Why this priority:** Es una forma adicional de recuperación que incentiva el comportamiento consistente a largo plazo; importante pero no tan urgente como el aumento por check-out.

**Independent Test:** Puede probarse simulando un estudiante con 30 días sin infracciones y verificando que el sistema aplica automáticamente +5 puntos.

**Acceptance Scenarios:**

**Scenario: Recuperación automática después de 30 días sin infracciones**
- **Given** un estudiante con score de confianza de 30 puntos y 30 días consecutivos sin infracciones.
- **When** el sistema ejecuta el proceso batch diario de revisión.
- **Then** el sistema aumenta el score a 35 puntos, registra el cambio con motivo "30 días sin infracciones", y dispara (`«include»`) *Notificar sanción*.

**Scenario: No hay recuperación porque aún no han pasado 30 días**
- **Given** un estudiante con score de 30 puntos y solo 20 días sin infracciones.
- **When** el sistema ejecuta el proceso batch diario.
- **Then** el sistema NO aplica recuperación y no realiza ninguna acción.

**Scenario: Recuperación cuando el score ya está en máximo**
- **Given** un estudiante con score de 50 puntos y 30 días sin infracciones.
- **When** el sistema ejecuta el proceso batch diario.
- **Then** el sistema NO aumenta el score (ya está en máximo), pero registra el cumplimiento del período.

### User Story 4 - Disparo del bloqueo/desbloqueo según el score (Priority: P1)
Como sistema, cuando una actualización deje el score en 0, debo disparar el bloqueo del estudiante; cuando una actualización lo deje por encima de 0 estando bloqueado por score, debo disparar su desbloqueo. El bloqueo/desbloqueo en sí lo ejecuta el caso de uso *Bloquear usuario*.

**Why this priority:** Es la consecuencia más grave de una reducción de score y debe dispararse siempre que corresponda; sin embargo, el efecto (bloquear/desbloquear) es responsabilidad de otro caso de uso, para no duplicar esa lógica aquí.

**Independent Test:** Puede probarse simulando un estudiante con score de 5 puntos, aplicando una reducción de 5 puntos, y verificando que el score queda en 0 y se dispara (`«extend»`) *Bloquear usuario*.

**Acceptance Scenarios:**

**Scenario: Score llega a 0 tras una reducción**
- **Given** un estudiante con score de confianza de 5 puntos.
- **When** el sistema recibe una solicitud de reducción de 5 puntos.
- **Then** el score queda en 0 y el sistema dispara (`«extend»`) *Bloquear usuario*.

**Scenario: Score sube por encima de 0 tras una recuperación**
- **Given** un estudiante bloqueado con score de 0 puntos.
- **When** realiza un check-out exitoso (+2 puntos) o completa un período de recuperación (+5 puntos).
- **Then** el score sube por encima de 0 y el sistema dispara (`«extend»`) *Bloquear usuario* para que gestione el desbloqueo.

### User Story 5 - Consulta y visualización del score (Priority: P3)
Como estudiante y administrador, necesito poder consultar el score de confianza actual y su historial de cambios, para entender mi situación y tomar acciones correctivas.

**Why this priority:** La consulta es importante para la transparencia, pero no es crítica para el funcionamiento del sistema.

**Independent Test:** Puede probarse consultando el score de un estudiante y verificando que muestra el valor actual y el historial de cambios.

**Acceptance Scenarios:**

**Scenario: Estudiante consulta su score actual**
- **Given** un estudiante autenticado en el sistema.
- **When** accede a su perfil y consulta su score de confianza.
- **Then** el sistema muestra su score actual y el historial de los últimos cambios (ej. "10/09/2026: +2 puntos por entrega a tiempo").

**Scenario: Administrador consulta score de un estudiante**
- **Given** un administrador autenticado en el sistema.
- **When** busca a un estudiante y consulta su score de confianza.
- **Then** el sistema muestra el score actual, el historial completo de cambios, y el estado del estudiante (Activo/Bloqueado).

## Edge Cases

- **Score en 5 y se aplica una reducción de 5 puntos:** el score queda en 0 y se dispara el bloqueo (`«extend»` a *Bloquear usuario*).
- **Score en 49 y se aplica un aumento de 2 puntos:** el score queda en 50 (máximo); nunca puede superarlo.
- **Score en 48 y se aplica un aumento de 5 puntos (recuperación):** el score queda en 50 (máximo); nunca puede superarlo.
- **Estudiante bloqueado por score y con una sanción activa por retraso:** ambas situaciones coexisten; su resolución es responsabilidad de *Bloquear usuario*, no de este caso de uso.
- **Reducción de score sobre un estudiante ya bloqueado por daño técnico:** la reducción se aplica igual para mantener el historial actualizado, aunque el estudiante ya estuviera bloqueado por otro motivo.
- **Múltiples check-outs exitosos el mismo día:** cada uno otorga +2 puntos, siempre respetando el máximo de 50.
- **Intento de reducción que llevaría el score por debajo de 0:** el sistema establece el score en 0 (nunca negativo) y dispara el bloqueo correspondiente.
- **Solicitud de actualización sin un evento de origen válido:** el sistema debe rechazarla e informar que no fue posible identificar el origen del cambio.

## Requirements

### Functional Requirements

**Actualizaciones del score:**
- **FR-SC-001:** El sistema debe reducir el score de confianza del estudiante en 5 puntos cuando se ejecute *Calcular penalización* y la regla configurada tenga `afecta_score_confianza = TRUE`.
- **FR-SC-002:** El sistema debe aumentar el score de confianza del estudiante en 2 puntos cuando realice un check-out exitoso (dentro del tiempo límite establecido).
- **FR-SC-003:** El sistema debe aumentar el score de confianza del estudiante en 5 puntos cuando complete el período configurado (30 días por defecto) sin infracciones.

**Límites:**
- **FR-SC-004:** El sistema debe garantizar que el score de confianza nunca sea negativo; el valor mínimo es 0.
- **FR-SC-005:** El sistema debe garantizar que el score de confianza nunca supere los 50 puntos; el valor máximo es 50.

**Disparo de bloqueo/desbloqueo y notificación:**
- **FR-SC-006:** El sistema debe disparar (`«extend»`) el caso de uso *Bloquear usuario* cada vez que una actualización de score resulte en 0 puntos.
- **FR-SC-007:** El sistema debe disparar (`«extend»`) el caso de uso *Bloquear usuario* cada vez que una actualización de score resulte en un valor mayor a 0 para un estudiante previamente bloqueado por score.
- **FR-SC-008:** El sistema debe disparar (`«include»`) el caso de uso *Notificar sanción* cada vez que se actualice el score de confianza, sea por reducción, aumento por check-out, o recuperación por período.

**Historial:**
- **FR-SC-009:** El sistema debe registrar en el historial de cambios de score cada actualización, incluyendo: fecha, tipo de cambio (REDUCCION, AUMENTO_CHECKOUT, RECUPERACION_PERIODO), puntos, motivo, y score resultante.

**Consultas y administración:**
- **FR-SC-010:** El sistema debe permitir a los estudiantes consultar su score de confianza actual y su historial de cambios.
- **FR-SC-011:** El sistema debe permitir a los administradores consultar el score de confianza y el historial de cambios de cualquier estudiante.

**Configuración:**
- **FR-SC-012:** El sistema debe permitir al administrador configurar el período sin infracciones requerido para la recuperación (valor por defecto: 30 días) en `Configuracion_Sistema`.
- **FR-SC-013:** El sistema debe permitir al administrador configurar los puntos otorgados por check-out exitoso (valor por defecto: 2 puntos) en `Configuracion_Sistema`.
- **FR-SC-014:** El sistema debe disparar este caso de uso como un `«extend»` condicional desde *Calcular penalización*, solo cuando `afecta_score_confianza = TRUE`.

## Key Entities
- **Score_Confianza:** Nivel de confianza actual del estudiante. Inicia en 50 puntos, se mueve entre 0 y 50 mediante incrementos/decrementos variables (-5, +2, +5). Incluye valor actual y fecha de última actualización.
- **Historial_Score:** Registro de cada cambio: estudiante, fecha, tipo de cambio (REDUCCION, AUMENTO_CHECKOUT, RECUPERACION_PERIODO), puntos, motivo, score antes y después del cambio.
- **Estudiante:** Titular del score (su estado de bloqueo y el detalle de bloqueos viven en *Bloquear usuario*).
- **Configuracion_Sistema:** Parámetros globales: período sin infracciones para recuperación (30 días), puntos por check-out exitoso (2 puntos), score máximo (50), score mínimo (0).

## Success Criteria

### Measurable Outcomes
- **SC-001:** El sistema debe actualizar el score de confianza en menos de 1 segundo desde que recibe la solicitud de origen (penalización, check-out o recuperación).
- **SC-002:** El score de confianza debe iniciar en 50 puntos para todos los estudiantes nuevos y actualizarse con una precisión del 100% en los cálculos.
- **SC-003:** El score de confianza debe mantenerse siempre entre 0 y 50 puntos, sin excepción.
- **SC-004:** El 100% de las actualizaciones que resulten en score = 0 deben disparar *Bloquear usuario*.
- **SC-005:** El 100% de las actualizaciones que resulten en score > 0 para un estudiante bloqueado por score deben disparar *Bloquear usuario* para su desbloqueo.
- **SC-006:** El 100% de las actualizaciones de score deben disparar *Notificar sanción*.
- **SC-007:** El 100% de los cambios de score deben quedar registrados en el historial con todos los campos obligatorios.
- **SC-008:** Los administradores deben poder configurar los parámetros del score (período de recuperación, puntos por check-out) en menos de 5 minutos.
