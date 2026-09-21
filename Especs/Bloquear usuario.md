# Feature Specification: Bloquear Usuario

**Created:** 2026-09-04

## User Scenarios & Testing

### User Story 1 - Bloqueo automático por score de confianza en 0 (Priority: P1)
Como sistema, cuando el score de confianza de un estudiante llegue a 0 puntos, debo bloquearlo automáticamente para impedir que realice nuevos préstamos.

**Why this priority:** Es el mecanismo de control final del módulo de sanciones: sin él, un estudiante con historial de incumplimientos podría seguir solicitando recursos sin límite.

**Independent Test:** Puede probarse de forma independiente invocando el caso de uso con un estudiante cuyo score acaba de llegar a 0 y verificando que su estado cambia a "Bloqueado por score" y que no puede iniciar un nuevo préstamo.

**Acceptance Scenarios:**

**Scenario: Bloqueo automático al llegar a 0**
- **Given** el caso de uso *Actualizar score de confianza* dispara el bloqueo (`«extend»`) porque el score de un estudiante llegó a 0.
- **When** el sistema procesa el bloqueo.
- **Then** el estado del estudiante cambia a "Bloqueado por score" y se rechaza cualquier intento posterior de nuevo préstamo.

**Scenario: Intento de préstamo con usuario bloqueado**
- **Given** un estudiante en estado "Bloqueado por score".
- **When** intenta solicitar un nuevo préstamo.
- **Then** el sistema rechaza la solicitud e informa que el estudiante se encuentra bloqueado por score de confianza insuficiente.

### User Story 2 - Desbloqueo por recuperación de score o intervención administrativa (Priority: P2)
Como administrador, necesito poder desbloquear a un estudiante cuando su score se recupere por encima de 0 o cuando decida habilitarlo manualmente.

**Why this priority:** Da una vía de salida al bloqueo automático; sin ella, el bloqueo sería permanente e irreversible.

**Independent Test:** Puede probarse con un estudiante bloqueado cuyo score sube por encima de 0 mediante el flujo de recuperación, verificando que su estado vuelve a "Activo".

**Acceptance Scenarios:**

**Scenario: Desbloqueo por recuperación de score**
- **Given** un estudiante en estado "Bloqueado por score" cuyo score sube de 0 a 5 puntos mediante el flujo de *Recuperar score de confianza*.
- **When** el sistema detecta que el score es mayor a 0.
- **Then** el estado del estudiante vuelve a "Activo".

**Scenario: Desbloqueo manual por administrador**
- **Given** un estudiante en estado "Bloqueado por score".
- **When** un administrador ejecuta el flujo de habilitación manual.
- **Then** el estado del estudiante vuelve a "Activo", quedando registrada la intervención administrativa en el historial.

### User Story 3 - Visualización del estado bloqueado en el perfil (Priority: P1)
Como estudiante, quiero ver un banner prominente cuando mi cuenta esté bloqueada, para entender inmediatamente por qué no puedo realizar nuevos préstamos.

**Why this priority:** El bloqueo debe ser visible antes de que el estudiante intente una nueva solicitud y debe explicar su causa.

**Independent Test:** Puede probarse con un estudiante cuyo score sea 0, verificando que su perfil muestra el banner rojo, el score actual y el motivo de la sanción.

**Acceptance Scenarios:**

**Scenario: Banner de bloqueo por sanción completa**
- **Given** un estudiante con score de confianza igual a 0.
- **When** accede a su perfil.
- **Then** el sistema muestra un banner rojo con el texto "ESTADO: BLOQUEADO POR SANCIÓN COMPLETA. Su score actual es 0 / 50.".

**Scenario: Motivo de sanción reciente**
- **Given** un estudiante bloqueado con una sanción registrada recientemente.
- **When** consulta el encabezado de su perfil.
- **Then** el sistema muestra una notificación flotante o badge con el motivo y la variación, por ejemplo "Sanción: -5 pts por retraso".

### User Story 4 - Acceso al reglamento de préstamos (Priority: P2)
Como estudiante, quiero acceder rápidamente al reglamento oficial de préstamos desde mi perfil, para conocer las reglas y normativas del sistema.

**Why this priority:** El reglamento es la referencia institucional para interpretar sanciones, bloqueos y condiciones de préstamo.

**Independent Test:** Puede probarse seleccionando el enlace del reglamento desde un perfil bloqueado o activo y verificando que el documento se abre correctamente.

**Acceptance Scenarios:**

**Scenario: Consulta del reglamento**
- **Given** un estudiante autenticado en su perfil.
- **When** selecciona "Ver reglamento de préstamos".
- **Then** el sistema abre el reglamento institucional en una vista o modal disponible para consulta.

**Scenario: Reglamento no disponible**
- **Given** que el documento institucional no está disponible temporalmente.
- **When** el estudiante selecciona "Ver reglamento de préstamos".
- **Then** el sistema informa la indisponibilidad y permite volver al perfil sin perder el contexto.

## Edge Cases

- **Estudiante bloqueado tanto por score como por daño técnico:** ambos bloqueos coexisten de forma independiente; el estudiante permanece bloqueado hasta que se resuelvan ambas condiciones.
- **Recuperación de score que no llega a superar 0 (por ejemplo, de 0 a 0 si la operación no aplica):** el sistema debe mantener el bloqueo activo.
- **Intento de bloqueo duplicado (el estudiante ya está bloqueado por score):** el sistema debe ignorar el nuevo disparo sin generar un registro adicional inconsistente.
- **Error al persistir el cambio de estado del estudiante:** el sistema debe informar que el bloqueo no pudo aplicarse y evitar que el estudiante quede en un estado ambiguo (ni bloqueado ni activo).

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe bloquear automáticamente a un estudiante cuando el caso de uso *Actualizar score de confianza* dispare el bloqueo por score de confianza en 0.
- **FR-002:** El sistema debe rechazar cualquier intento de nuevo préstamo o check-out de un estudiante en estado "Bloqueado por score".
- **FR-003:** El sistema debe restaurar el estado "Activo" del estudiante cuando su score se recupere por encima de 0 puntos.
- **FR-004:** El sistema debe permitir a los administradores desbloquear manualmente a un estudiante, registrando la intervención en el historial.
- **FR-005:** El sistema debe mantener el bloqueo por daño técnico de forma independiente del bloqueo por score, de modo que ambos puedan coexistir.
- **FR-006:** El sistema debe impedir el registro de un bloqueo duplicado cuando el estudiante ya se encuentre bloqueado por score.
- **FR-007:** El perfil del estudiante debe mostrar un banner rojo cuando el score sea 0 o exista una sanción que bloquee nuevos préstamos.
- **FR-008:** El banner debe mostrar el estado, el score actual sobre 50 y el motivo de la sanción más reciente cuando exista.
- **FR-009:** El perfil debe ofrecer el enlace o botón "Ver reglamento de préstamos".
- **FR-010:** El sistema debe mostrar un estado visual habilitado cuando el score sea mayor que 0 y no existan bloqueos vigentes.

## Key Entities
- **Estudiante:** Contiene su estado actual (Activo, Bloqueado por score, Bloqueado por daño, Bloqueado por ambos).
- **Historial_Bloqueos:** Registro de cada bloqueo y desbloqueo aplicado, con fecha, motivo (score / daño / manual) y usuario responsable en caso de intervención administrativa.
- **Score_Confianza:** Origen del bloqueo automático cuando alcanza 0 puntos.
- **Reglamento_Prestamos:** Documento institucional consultable desde el perfil del estudiante.

## Success Criteria

### Measurable Outcomes
- **SC-001:** El 100% de los estudiantes cuyo score llegue a 0 deben quedar bloqueados automáticamente antes de poder realizar un nuevo préstamo.
- **SC-002:** El 100% de los intentos de préstamo de un estudiante bloqueado deben ser rechazados.
- **SC-003:** El 100% de las recuperaciones de score por encima de 0 deben restaurar el estado "Activo" del estudiante, salvo que exista un bloqueo por daño técnico vigente.
- **SC-004:** El sistema debe aplicar el bloqueo automático en menos de 1 segundo desde que el score llega a 0.
