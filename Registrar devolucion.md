
# Feature Specification: Registrar devolución

**Created**: 31/08/2026

## User Scenarios & Testing

### User Story 1 - Registrar devolución de un recurso (Priority: P1)

Como **estudiante**, quiero realizar la devolución de un recurso que tengo actualmente en uso, para finalizar su utilización y registrar el check-out correspondiente.

**Why this priority**: Es la funcionalidad principal del caso de uso, ya que permite finalizar formalmente la utilización de un recurso y dejar constancia de su devolución.

**Independent Test**: Puede probarse de forma independiente con un estudiante que tenga un recurso en uso. Al realizar la devolución, debe quedar registrada la fecha, finalizar la utilización y registrarse el check-out.

**Acceptance Scenarios**:

1. **Scenario**: Devolución exitosa

   * **Given** el estudiante tiene un recurso actualmente en uso.
   * **When** el estudiante realiza la devolución del recurso.
   * **Then** el sistema registra la fecha de devolución, registra el check-out, finaliza la utilización del recurso e inicializa el flujo de penalización correspondiente.

---

### User Story 2 - Intentar devolver un recurso sin utilización activa (Priority: P2)

Como **estudiante**, quiero recibir información cuando intento realizar una devolución sin tener un recurso en uso, para conocer que no existe una devolución pendiente.

**Why this priority**: Representa un escenario de fallo de la historia principal y evita que el sistema registre devoluciones que no corresponden a una utilización activa.

**Independent Test**: Puede probarse de forma independiente con un estudiante que no tenga ningún recurso actualmente en uso e intentando realizar una devolución.

**Acceptance Scenarios**:

1. **Scenario**: Devolución sin utilización activa

   * **Given** el estudiante no tiene ningún recurso actualmente en uso.
   * **When** intenta realizar una devolución.
   * **Then** el sistema rechaza la operación e informa que no existe una utilización activa para realizar la devolución.

---

## Edge Cases

* **Estudiante sin utilización activa:** El sistema debe rechazar la devolución e informar que no existe una utilización activa asociada al estudiante.

* **Utilización ya finalizada:** El sistema debe impedir que se registre nuevamente la devolución e informar que la utilización ya fue finalizada.

* **Recurso no identificado:** El sistema debe impedir el registro de la devolución e informar que no fue posible identificar el recurso asociado a la utilización.

* **Error al registrar la devolución:** El sistema debe informar que la devolución no pudo registrarse y evitar que quede almacenado un registro incompleto.

* **Error al finalizar la utilización:** El sistema debe informar que no fue posible completar la devolución y evitar que la utilización quede en un estado inconsistente.

* **Error al registrar la fecha:** El sistema debe impedir la finalización de la devolución hasta que la fecha pueda registrarse correctamente.

* **Intentos simultáneos de devolución:** El sistema debe permitir únicamente un registro exitoso de devolución para la misma utilización y rechazar los registros duplicados.

---

## Requirements

### Functional Requirements

* **FR-001**: El sistema debe permitir al estudiante realizar la devolución de un recurso que tenga actualmente en uso.
* **FR-002**: El sistema debe identificar la utilización activa asociada al estudiante.
* **FR-003**: El sistema debe mostrar al estudiante la información del recurso asociado a la utilización activa.
* **FR-004**: El sistema debe registrar la devolución realizada por el estudiante.
* **FR-005**: El sistema debe registrar la fecha en que el estudiante realiza la devolución.
* **FR-006**: El sistema debe registrar el check-out como parte del proceso de devolución.
* **FR-007**: El sistema debe finalizar la utilización del recurso cuando la devolución sea registrada correctamente.
* **FR-008**: El sistema debe inicializar el flujo de penalización una vez registrado correctamente el check-out.
* **FR-009**: El sistema debe impedir que una utilización finalizada sea registrada nuevamente como devolución.
* **FR-010**: El sistema debe informar al estudiante cuando la devolución haya sido registrada correctamente.
* **FR-011**: El sistema debe informar al estudiante cuando no exista una utilización activa que pueda ser devuelta.
* **FR-012**: El sistema debe mantener asociada la devolución con el estudiante y el recurso correspondiente.

### Key Entities

* **Estudiante**: Representa a la persona que tiene bajo su responsabilidad un recurso y realiza la devolución.

* **Recurso**: Representa el activo universitario que se encuentra en uso por parte del estudiante y que posteriormente es devuelto.

* **Utilización**: Representa la relación activa entre el estudiante y el recurso durante el período en que este se encuentra bajo su responsabilidad.

* **Devolución**: Representa el registro de la entrega realizada por el estudiante, incluyendo la fecha en que fue realizada y su relación con el estudiante, el recurso y la utilización correspondiente.

---

## Success Criteria

### Measurable Outcomes

* **SC-001**: El 100% de las devoluciones realizadas por estudiantes con una utilización activa deben quedar registradas con su fecha correspondiente.

* **SC-002**: El 100% de las devoluciones registradas correctamente deben generar el check-out correspondiente.

* **SC-003**: El 100% de las devoluciones registradas correctamente deben finalizar la utilización asociada al recurso.

* **SC-004**: El 100% de los check-outs registrados correctamente deben inicializar el flujo de penalización correspondiente.

* **SC-005**: El 100% de los intentos de devolución realizados sin una utilización activa deben ser rechazados.

* **SC-006**: El sistema debe impedir el 100% de los intentos de registrar nuevamente una utilización que ya haya sido finalizada.
