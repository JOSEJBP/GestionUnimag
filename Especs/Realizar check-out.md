# Feature Specification: Realizar check-out

**Created**: 31/08/2026

## User Scenarios & Testing

### User Story 1 - Realizar check-out de un recurso (Priority: P1)

Como **estudiante**, quiero realizar el check-out de un recurso que tengo actualmente en uso, para finalizar su utilización y dejar registrada la devolución.

**Why this priority**: Es la funcionalidad principal del caso de uso, ya que permite finalizar formalmente la utilización de un recurso y registrar la devolución realizada por el estudiante.

**Independent Test**: Puede probarse de forma independiente con un estudiante que tenga un recurso actualmente en uso. Al realizar el check-out, el sistema debe registrar la fecha de devolución, finalizar la utilización del recurso y dar inicio al flujo de penalización correspondiente.

**Acceptance Scenarios**:

1. **Scenario**: Check-out exitoso

   * **Given** el estudiante tiene un recurso actualmente en uso.
   * **When** el estudiante realiza el check-out del recurso.
   * **Then** el sistema registra el check-out con la fecha de devolución, finaliza la utilización del recurso e inicializa el flujo de penalización correspondiente.

### User Story 2 - Intentar realizar check-out sin utilización activa (Priority: P2)

Como **estudiante**, quiero recibir información cuando intento realizar un check-out sin tener un recurso actualmente en uso, para conocer que no existe una utilización pendiente de finalizar.

**Why this priority**: Representa un escenario de fallo de la funcionalidad principal y evita que el sistema registre un check-out cuando no existe una utilización activa asociada al estudiante.

**Independent Test**: Puede probarse de forma independiente con un estudiante que no tenga ningún recurso actualmente en uso e intentando realizar un check-out.

**Acceptance Scenarios**:

1. **Scenario**: Check-out rechazado por falta de utilización activa

   * **Given** el estudiante no tiene ningún recurso actualmente en uso.
   * **When** el estudiante intenta realizar un check-out.
   * **Then** el sistema rechaza la operación e informa que no existe una utilización activa para realizar el check-out.

### Edge Cases

* **Estudiante sin utilización activa:** El sistema debe rechazar el check-out e informar que no existe una utilización activa asociada al estudiante.

* **Utilización ya finalizada:** El sistema debe impedir que se registre nuevamente el check-out e informar que la utilización ya fue finalizada.

* **Recurso no identificado:** El sistema debe impedir el registro del check-out e informar que no fue posible identificar el recurso asociado a la utilización.

* **Error al registrar el check-out:** El sistema debe informar que el check-out no pudo registrarse y evitar que se almacene un registro incompleto.

* **Error al finalizar la utilización:** El sistema debe informar que no fue posible completar el check-out y evitar que la utilización quede en un estado inconsistente.

* **Error al registrar la fecha de devolución:** El sistema debe impedir la finalización del check-out e informar que no fue posible registrar correctamente la fecha de devolución.

* **Intentos simultáneos de check-out:** El sistema debe permitir únicamente un registro exitoso de check-out para la misma utilización y rechazar los intentos duplicados.

## Requirements

### Functional Requirements

* **FR-001**: El sistema debe permitir al estudiante realizar el check-out de un recurso que tenga actualmente en uso.

* **FR-002**: El sistema debe identificar la utilización activa asociada al estudiante.

* **FR-003**: El sistema debe identificar el recurso asociado a la utilización activa.

* **FR-004**: El sistema debe registrar el check-out realizado por el estudiante.

* **FR-005**: El sistema debe registrar la fecha en que el estudiante realiza el check-out.

* **FR-006**: El sistema debe finalizar la utilización del recurso cuando el check-out sea registrado correctamente.

* **FR-007**: El sistema debe inicializar el flujo de penalización una vez registrado correctamente el check-out.

* **FR-008**: El sistema debe impedir que una utilización finalizada sea registrada nuevamente mediante un check-out.

* **FR-009**: El sistema debe informar al estudiante cuando el check-out haya sido registrado correctamente.

* **FR-010**: El sistema debe informar al estudiante cuando no exista una utilización activa que pueda ser finalizada mediante un check-out.

* **FR-011**: El sistema debe mantener asociado el check-out con el estudiante, el recurso y la utilización correspondiente.

### Key Entities

* **Estudiante**: Representa a la persona que tiene bajo su responsabilidad un recurso y realiza el check-out.

* **Recurso**: Representa el activo universitario que se encuentra en uso por parte del estudiante y cuya utilización será finalizada mediante el check-out.

* **Utilización**: Representa la relación entre el estudiante y el recurso durante el período en que este se encuentra bajo su responsabilidad.

* **Check-out**: Representa la operación mediante la cual se registra la devolución y se finaliza la utilización del recurso. Contiene la fecha en que se realizó y se relaciona con el estudiante, el recurso y la utilización correspondiente.

## Success Criteria

### Measurable Outcomes

* **SC-001**: El 100% de los check-outs realizados por estudiantes con una utilización activa deben quedar registrados con su fecha de devolución correspondiente.

* **SC-002**: El 100% de los check-outs registrados correctamente deben finalizar la utilización correspondiente.

* **SC-003**: El 100% de los check-outs registrados correctamente deben quedar asociados al estudiante, recurso y utilización correspondientes.

* **SC-004**: El 100% de los check-outs registrados correctamente deben inicializar el flujo de penalización correspondiente.

* **SC-005**: El 100% de los intentos de realizar check-out sin una utilización activa deben ser rechazados.

* **SC-006**: El sistema debe impedir el 100% de los intentos de registrar nuevamente un check-out para una utilización que ya haya sido finalizada.
