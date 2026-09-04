# Feature Specification: Actualizar Score de Confianza

**Created:** 2026-09-04

## User Scenarios & Testing

### User Story 1 - Disminución del score por penalización (Priority: P1)
Como sistema, cuando el caso de uso *Calcular penalización* dispara la actualización (`«include»`), debo restar 5 puntos del score de confianza del estudiante y dejar registrado el nuevo valor.

**Why this priority:** Es el efecto directo y obligatorio de toda penalización que afecte el score; sin esta actualización, la escala progresiva de *Calcular penalización* no tendría impacto real sobre el estudiante.

**Independent Test:** Puede probarse de forma independiente invocando el caso de uso con un estudiante de score conocido y verificando que el resultado sea exactamente 5 puntos menos, sin depender del cálculo de la penalización en sí.

**Acceptance Scenarios:**

**Scenario: Actualización estándar tras una penalización**
- **Given** un estudiante con score de confianza de 45 puntos.
- **When** el caso de uso *Calcular penalización* dispara la actualización.
- **Then** el sistema resta 5 puntos, el score queda en 40, y se registra el cambio con fecha y motivo asociado.

**Scenario: Score inicial de un estudiante nuevo**
- **Given** un estudiante es registrado por primera vez en el sistema.
- **When** se consulta su score de confianza.
- **Then** el score debe ser 50 puntos.

**Scenario: Reducción progresiva del score**
- **Given** un estudiante con score de 50 puntos.
- **When** recibe 3 sanciones consecutivas.
- **Then** el score se reduce a 35 puntos (50 − 5 − 5 − 5).

### User Story 2 - Score llega a 0 y dispara bloqueo (Priority: P1)
Como sistema, cuando la resta de puntos deje el score de confianza en 0, debo disparar el caso de uso *Bloquear usuario*.

**Why this priority:** Es el mecanismo de control que impide que un estudiante con historial de incumplimientos siga solicitando recursos; es la razón de existir del score de confianza.

**Independent Test:** Puede probarse simulando un estudiante con score de 5 puntos que recibe una nueva sanción y verificando que el score queda en 0 y se dispara el caso de uso *Bloquear usuario*.

**Acceptance Scenarios:**

**Scenario: Score en 0 dispara bloqueo**
- **Given** un estudiante con score de 5 puntos y 9 infracciones previas.
- **When** recibe una nueva sanción que resta 5 puntos.
- **Then** el score queda en 0 y el sistema dispara (`«extend»`) el caso de uso *Bloquear usuario*.

### User Story 3 - Recuperación de score por buen comportamiento (Priority: P2)
Como administrador, necesito poder ejecutar el flujo de recuperación de score para que un estudiante con buen comportamiento recupere puntos.

**Why this priority:** Da una vía de salida al bloqueo automático y evita sanciones permanentes desproporcionadas; no es bloqueante para el cálculo base de penalizaciones.

**Independent Test:** Puede probarse creando un estudiante con score de 30 puntos y 30 días sin infracciones, ejecutando la recuperación y verificando que el score sube a 35.

**Acceptance Scenarios:**

**Scenario: Recuperación de puntos**
- **Given** un estudiante con score de 30 puntos que ha completado 30 días sin infracciones.
- **When** el administrador ejecuta el flujo de *Recuperar score de confianza*.
- **Then** el sistema aumenta 5 puntos del score (queda en 35), respetando siempre el incremento en múltiplos de 5.

## Edge Cases

- **Score no múltiplo de 5:** nunca debería ocurrir, ya que el score solo se modifica en incrementos/decrementos de 5 puntos; si se detecta, el sistema debe marcarlo como inconsistencia para revisión administrativa.
- **Estudiante bloqueado por daño técnico que además recibe una penalización de score:** ambas condiciones coexisten; el bloqueo por daño prevalece independientemente del resultado del score.
- **Intento de actualizar el score sin un evento de origen válido (penalización o recuperación):** el sistema debe rechazar la actualización e informar que no fue posible identificar el origen del cambio.
- **Error al persistir el nuevo score:** el sistema debe informar que la penalización se calculó pero no pudo reflejarse en el score, y marcar la inconsistencia para revisión, sin dejar el score en un estado intermedio.
- **Recuperación solicitada para un estudiante ya en 50 puntos (score máximo):** el sistema debe rechazar el incremento e informar que el estudiante ya tiene el score máximo.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe mantener un score de confianza inicial de 50 puntos para cada estudiante nuevo.
- **FR-002:** El sistema debe restar 5 puntos del score de confianza del estudiante cuando el caso de uso *Calcular penalización* dispare la actualización.
- **FR-003:** El sistema debe garantizar que el score de confianza siempre sea un múltiplo de 5.
- **FR-004:** El sistema debe registrar cada actualización de score (fecha, delta aplicado, score resultante, origen del cambio).
- **FR-005:** El sistema debe disparar (`«extend»`) el caso de uso *Bloquear usuario* cuando el score de confianza alcance 0 puntos.
- **FR-006:** El sistema debe permitir a los administradores ejecutar un flujo de *Recuperar score de confianza* que aumente 5 puntos por cada período de buen comportamiento configurado (por defecto, 30 días sin infracciones).
- **FR-007:** El sistema debe impedir que el score de confianza supere el máximo definido (50 puntos) o sea inferior al mínimo definido (0 puntos).
- **FR-008:** El sistema debe rechazar cualquier actualización de score que no provenga de un evento de origen válido (penalización o recuperación).
- **FR-009:** El sistema debe mantener un valor fijo de 5 puntos por actualización (incremento o decremento), no configurable por el administrador.

## Key Entities
- **Score_Confianza:** Nivel de confianza del estudiante. Inicia en 50 puntos y se actualiza siempre en múltiplos de 5 (± 5 por evento). Dispara bloqueos automáticos al llegar a 0.
- **Estudiante:** Titular del score, con su estado (Activo, Bloqueado por score, Bloqueado por daño, etc.).
- **Historial_Score:** Registro de cada cambio de score, con fecha, delta, score resultante y origen (penalización o recuperación).
- **Configuracion_Sistema:** Parámetro configurable para el período de buen comportamiento requerido para la recuperación (por defecto, 30 días).

## Success Criteria

### Measurable Outcomes
- **SC-001:** El score de confianza debe iniciar en 50 puntos para el 100% de los estudiantes nuevos.
- **SC-002:** El 100% de las actualizaciones de score deben restar o sumar exactamente 5 puntos, con precisión del 100% en los cálculos.
- **SC-003:** El score de confianza debe mantenerse siempre como múltiplo de 5, garantizando que nunca existan valores como 47, 38 o 22.
- **SC-004:** El sistema debe disparar el caso de uso *Bloquear usuario* en el 100% de los casos en que el score llegue a 0.
- **SC-005:** El sistema debe actualizar el score en menos de 1 segundo desde que recibe el evento de origen.
