# Feature Specification: Calcular Penalización

**Created:** 2026-09-03
**Revisado:** 2026-09-04 — se separó la lógica de score de confianza (→ *Actualizar score de confianza*), bloqueo (→ *Bloquear usuario*) y notificación (→ *Notificar sanción*), que ahora son casos de uso independientes en el diagrama.

## User Scenarios & Testing

### User Story 1 - Sanción progresiva por retraso en check-out (Priority: P1)
Como sistema, cuando un estudiante realiza el check-out de un equipo y excede el tiempo límite de uso, debo contar sus infracciones previas en los últimos 30 días y determinar la sanción correspondiente según la escala progresiva configurada (1 día para primera infracción, 2 días para segunda, etc.).

**Why this priority:** El retraso en la entrega de equipos es la infracción más común y tiene mayor impacto en la disponibilidad de recursos. La sanción progresiva es el corazón del módulo de sanciones y debe funcionar siempre desde el MVP.

**Independent Test:** Puede probarse de forma independiente simulando un estudiante con 0 infracciones previas que realiza check-out con 30 minutos de retraso, y verificando que el sistema determina 1 día de suspensión y dispara la actualización del score de confianza.

**Acceptance Scenarios:**

**Scenario: Primera infracción por retraso (sin antecedentes)**
- **Given** un estudiante con 0 infracciones en los últimos 30 días.
- **When** realiza check-out con 30 minutos de retraso (supera el umbral de 15 minutos).
- **Then** el sistema determina 1 día de suspensión, registra la infracción en el historial y dispara la actualización del score de confianza.

**Scenario: Quinta infracción por retraso (sanción máxima configurada)**
- **Given** un estudiante con 4 infracciones previas en los últimos 30 días.
- **When** realiza check-out con 45 minutos de retraso.
- **Then** el sistema determina 7 días de suspensión, registra la infracción y dispara la actualización del score de confianza.

**Scenario: Infracción que no supera el umbral mínimo**
- **Given** un estudiante con 0 infracciones previas.
- **When** realiza check-out con 10 minutos de retraso (umbral mínimo configurado en 15 minutos).
- **Then** el sistema NO calcula penalización porque el retraso no supera el umbral mínimo.

### User Story 2 - Sanción progresiva por inasistencia (Priority: P1)
Como sistema, cuando un profesor o administrador reporta que un estudiante no asistió a su reserva, debo contar sus infracciones previas en los últimos 30 días y aplicar obligatoriamente la sanción progresiva correspondiente.

**Why this priority:** La inasistencia es una falta grave que afecta la disponibilidad de recursos y la planificación. Es igual de crítica que el retraso y debe estar disponible desde el MVP.

**Independent Test:** Puede probarse simulando el reporte de inasistencia para un estudiante con 2 infracciones previas y verificando que el sistema determina 3 días de suspensión y dispara la actualización del score.

**Acceptance Scenarios:**

**Scenario: Primera inasistencia**
- **Given** un estudiante con 0 infracciones previas.
- **When** se ejecuta el flujo de *Reportar no asistencia*.
- **Then** el sistema aplica obligatoriamente 1 día de suspensión, registra la infracción y dispara la actualización del score.

**Scenario: Más de 5 inasistencias (sanción máxima)**
- **Given** un estudiante con 5 infracciones previas en los últimos 30 días.
- **When** se reporta una nueva inasistencia.
- **Then** el sistema aplica 30 días de suspensión, registra la infracción y dispara la actualización del score.

### User Story 3 - Configuración de escalas progresivas (Priority: P3)
Como administrador, necesito poder modificar los rangos de la escala progresiva (días de suspensión) a través de una interfaz de administración, sin necesidad de modificar el código fuente.

**Why this priority:** Importante para la mantenibilidad del sistema, pero no crítica para el MVP; se puede comenzar con valores predefinidos y ajustarlos después.

**Independent Test:** Accediendo a la interfaz de administración, modificando la escala (ej. cambiar "5 infracciones = 7 días" a "5 infracciones = 10 días") y verificando que las siguientes sanciones usan el nuevo valor.

**Acceptance Scenarios:**

**Scenario: Modificación de días de suspensión**
- **Given** un administrador accede a la configuración de escalas.
- **When** cambia la escala para 2 infracciones previas de 3 días a 5 días.
- **Then** el sistema guarda el cambio y las siguientes sanciones con 2 infracciones previas aplican 5 días.

**Scenario: Adición de nuevo rango en la escala**
- **Given** un administrador accede a la configuración de escalas.
- **When** agrega un nuevo rango "6 infracciones = 45 días".
- **Then** el sistema guarda el nuevo rango y lo aplica cuando un estudiante alcanza 6 infracciones.

## Edge Cases

- **Infracciones de distinto tipo (retraso + inasistencia):** se acumulan todas en los últimos 30 días, independientemente del tipo, para el conteo de infracciones previas.
- **Infracciones que superan el último rango configurado:** el sistema aplica el rango más alto definido en la escala.
- **Reinicio del contador:** ocurre automáticamente tras 30 días sin infracciones (período configurable).
- **No existe una regla de penalización activa (`Sancion_Reglas`) para el evento recibido:** el sistema debe rechazar el cálculo e informar que no fue posible determinar la sanción aplicable.
- **No existe un rango configurado en `Sancion_Escala_Progresiva`:** el sistema debe rechazar el cálculo e informar que no fue posible determinar los días de suspensión.
- **Intento de calcular penalización duplicada para el mismo evento:** el sistema debe rechazar el segundo cálculo y evitar duplicar la infracción en el historial.
- **Error al registrar la infracción en el historial:** el sistema debe informar que la penalización no pudo registrarse y evitar dejar un registro incompleto.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe calcular una penalización cuando el flujo de *Realizar check-out* (`«extend»`) indique que el tiempo de uso superó el umbral mínimo configurado en `Sancion_Reglas`.
- **FR-002:** El sistema debe calcular obligatoriamente una penalización siempre que se ejecute el flujo de *Reportar no asistencia* (`«include»`).
- **FR-003:** El sistema debe determinar los días de suspensión consultando `Sancion_Escala_Progresiva`, según el número de infracciones previas del estudiante.
- **FR-004:** El sistema debe registrar la infracción en `Historial_Sanciones`, vinculándola con el estudiante, la regla y la escala aplicadas.
- **FR-005:** El sistema debe registrar el motivo exacto de la penalización (Retraso en entrega, Inasistencia, etc.) tomado de `Sancion_Reglas`.
- **FR-006:** El sistema debe disparar (`«include»`) el caso de uso *Actualizar score de confianza* cuando `Sancion_Reglas.afecta_score_confianza = TRUE`.
- **FR-007:** El sistema debe disparar (`«include»`) el caso de uso *Notificar sanción* una vez registrada correctamente la penalización.
- **FR-008:** El sistema debe impedir el cálculo duplicado de una penalización para el mismo evento (check-out o reporte de inasistencia).
- **FR-009:** El sistema debe informar cuando un check-out no supere el umbral mínimo y por lo tanto no genere penalización.
- **FR-010:** El sistema debe rechazar el cálculo e informar el motivo cuando no exista una regla activa en `Sancion_Reglas` o un rango aplicable en `Sancion_Escala_Progresiva`.
- **FR-011:** El sistema debe aplicar el rango más alto definido en la escala cuando las infracciones previas superen el último rango configurado.
- **FR-012:** El sistema debe permitir a los administradores gestionar (crear, modificar, eliminar) los rangos de `Sancion_Escala_Progresiva`, ajustando únicamente los días de suspensión.
- **FR-013:** El sistema debe contar las infracciones previas del estudiante considerando solo el período configurado (30 días por defecto).
- **FR-014:** El sistema debe acumular todas las infracciones del estudiante (retrasos e inasistencias) dentro del período configurado para determinar la escala aplicable.

## Key Entities
- **Sancion_Reglas:** Configuración base de cada tipo de penalización (motivo, unidad de cálculo, umbral mínimo, indicador `afecta_score_confianza`, estado activo/inactivo).
- **Sancion_Escala_Progresiva:** Progresión de sanciones según el historial de infracciones (infracciones_previas → días de suspensión).
- **Historial_Sanciones:** Registro de cada penalización aplicada (regla y escala aplicadas, estudiante, fecha, días de suspensión, motivo, estado: Pendiente/Cumplida/Apelada). *No almacena el detalle del score; referencia el evento para trazabilidad.*
- **Estudiante:** Referencia al infractor (su score e historial completo viven en el caso de uso *Actualizar score de confianza*).
- **Configuracion_Sistema:** Parámetros globales, incluyendo el período (días) para contar infracciones previas.

## Success Criteria

### Measurable Outcomes
- **SC-001:** El sistema debe calcular cualquier penalización en menos de 2 segundos desde que se dispara el flujo (check-out o inasistencia), incluyendo la consulta al historial y la búsqueda en la escala progresiva.
- **SC-002:** El 100% de las sanciones deben aplicar correctamente la escala progresiva basada en el conteo de infracciones previas en el período configurado.
- **SC-003:** El 100% de las penalizaciones calculadas correctamente deben disparar el caso de uso *Actualizar score de confianza* y *Notificar sanción*.
- **SC-004:** El 0% de los eventos ya penalizados deben generar un nuevo cálculo duplicado.
- **SC-005:** El 100% de los intentos sin regla o escala configurada deben ser rechazados, informando el motivo específico.
- **SC-006:** Los administradores deben poder modificar los rangos de la escala progresiva en menos de 5 minutos, viendo los cambios reflejados en la siguiente sanción calculada.
