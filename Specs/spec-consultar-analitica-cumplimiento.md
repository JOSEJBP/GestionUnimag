# Feature Specification: Consultar Analítica de Cumplimiento

**Created:** 2026-10-17
**Updated:** 2026-10-22 (el personal administrativo ve datos generales y el detalle por estudiante)

## User Scenarios & Testing

### User Story 1 - Consultar indicadores generales de cumplimiento (Priority: P1)
Como personal administrativo, quiero ver indicadores de cumplimiento del uso de recursos y espacios en un período, para controlar la operación y detectar problemas.

**Why this priority:** Es el caso de uso que respalda la "Analítica" del módulo y los reportes de cumplimiento y control administrativo que pide el proyecto.

**Independent Test:** Con datos de un semestre (check-outs, daños, sanciones y cobros), abrir el panel y verificar que los indicadores coinciden con los datos de origen.

**Acceptance Scenarios:**

**Scenario: Indicadores del período**
- **Given** datos de uso registrados en el período seleccionado (por defecto, el semestre vigente).
- **When** el personal administrativo abre la analítica.
- **Then** el sistema muestra: porcentaje de check-outs a tiempo y con retraso, cantidad de préstamos con descuento por retraso y de préstamos perdidos, cantidad de sanciones por inasistencia, cantidad de reportes de daño, y cantidad y monto total de cobros por estado (Pendiente, Pagado, Anulado). Son datos generales de todos los estudiantes juntos.

**Scenario: Filtros**
- **Given** el panel de indicadores abierto.
- **When** el personal administrativo filtra por rango de fechas, tipo de recurso o espacio, o facultad.
- **Then** el sistema recalcula los indicadores con el filtro aplicado.

**Scenario: Período sin datos**
- **Given** un período sin registros de uso.
- **When** el personal administrativo lo consulta.
- **Then** el sistema muestra un estado vacío informativo, sin indicadores en cero engañosos.

### User Story 2 - Identificar recursos y espacios críticos (Priority: P2)
Como personal administrativo, quiero ver rankings de los recursos con más daños y los salones con más inasistencias, para priorizar mantenimiento y ajustar políticas.

**Why this priority:** Convierte los indicadores en decisiones accionables, pero requiere que los indicadores generales ya funcionen.

**Independent Test:** Con varios recursos con distinta cantidad de daños, verificar que el ranking los ordena correctamente.

**Acceptance Scenarios:**

**Scenario: Ranking de recursos con más daños**
- **Given** reportes de daño en el período.
- **When** el personal administrativo abre el ranking de daños.
- **Then** el sistema muestra los recursos o tipos de recurso ordenados por cantidad de reportes.

**Scenario: Ranking de salones con más inasistencias**
- **Given** inasistencias confirmadas en el período.
- **When** el personal administrativo abre el ranking de inasistencias.
- **Then** el sistema muestra los espacios ordenados por cantidad de inasistencias confirmadas.

### User Story 3 - Exportar un reporte (Priority: P3)
Como personal administrativo, quiero exportar los indicadores del período a un archivo, para compartirlos o archivarlos.

**Why this priority:** Es una comodidad útil para reportes formales, pero no es necesaria para el análisis.

**Independent Test:** Exportar el panel de un período y verificar que el archivo contiene los mismos indicadores y filtros.

**Acceptance Scenarios:**

**Scenario: Exportación**
- **Given** un panel con filtros aplicados.
- **When** el personal administrativo exporta el reporte.
- **Then** el sistema genera un archivo (CSV o PDF) con los indicadores, el período y los filtros usados.

### User Story 4 - Consultar el detalle de un estudiante (Priority: P2)
Como personal administrativo, quiero buscar a un estudiante por su código institucional y ver su situación completa, para revisar su caso, decidir un desbloqueo o gestionar sus cobros.

**Why this priority:** Los indicadores generales no permiten resolver casos concretos; el personal necesita el detalle por estudiante para actuar, pero depende de que los datos de M3 ya estén consolidados.

**Independent Test:** Con un estudiante con check-outs, una inasistencia, un daño y un cobro, buscarlo por código y verificar que el detalle coincide con los datos de origen.

**Acceptance Scenarios:**

**Scenario: Detalle de un estudiante**
- **Given** un estudiante con historial en M3.
- **When** el personal administrativo lo busca por su código institucional.
- **Then** el sistema muestra su score actual y su historial, si está bloqueado, sus sanciones (vigentes y pasadas), sus check-outs (a tiempo y con retraso), sus préstamos vencidos, sus inasistencias, sus reportes de daño y sus cobros con su estado.

**Scenario: Estudiante sin historial**
- **Given** un código institucional sin registros en M3.
- **When** el personal administrativo lo busca.
- **Then** el sistema muestra el score inicial (50/50) y un estado vacío informativo.

**Scenario: Código inexistente o con formato inválido**
- **Given** un código con formato inválido.
- **When** el personal administrativo lo busca.
- **Then** el sistema rechaza la búsqueda e informa el error.

## Edge Cases

- **Rango de fechas inválido (fin anterior al inicio):** el sistema debe rechazar la consulta e informar el error.
- **Gran volumen de datos:** el sistema debe agregar y paginar para mantener el tiempo de respuesta.
- **Usuario sin permisos:** el sistema debe rechazar el acceso a quien no sea personal administrativo autorizado.
- **Datos descriptivos faltantes (facultad o ubicación de un recurso):** el sistema debe agrupar esos registros en "Sin clasificar" y no excluirlos de los totales. La ficha de reserva de M2 trae el tipo y la ubicación del recurso, pero no la facultad ni el programa del titular, así que el filtro por facultad depende de un dato que hoy ningún módulo entrega a M3. [POR DEFINIR con M1 y M2: de dónde sale la facultad.]
- **Privacidad:** los indicadores generales y los rankings no identifican a estudiantes; el detalle nominal solo está disponible en la consulta por estudiante, para el personal administrativo autorizado. No se registra quién consulta el detalle de cada estudiante.
- **Datos aún no consolidados (eventos pendientes de procesar):** el sistema debe indicar la fecha y hora de la última actualización de los datos.
- **Falla al exportar:** el sistema debe informar el error sin afectar la consulta en pantalla.

## Requirements

### Functional Requirements
- **FR-001:** El sistema debe mostrar el porcentaje de check-outs a tiempo y con retraso en el período seleccionado.
- **FR-002:** El sistema debe mostrar la cantidad de sanciones por inasistencia aplicadas, la cantidad de préstamos con descuento por retraso y la cantidad de préstamos declarados perdidos.
- **FR-003:** El sistema debe mostrar la cantidad de reportes de daño y la cantidad y monto total de cobros por estado (Pendiente, Pagado, Anulado).
- **FR-004:** El sistema debe permitir filtrar por rango de fechas, tipo de recurso o espacio, y facultad, con el semestre vigente como período por defecto.
- **FR-005:** El sistema debe mostrar el ranking de recursos o tipos de recurso con más reportes de daño.
- **FR-006:** El sistema debe mostrar el ranking de espacios con más inasistencias confirmadas.
- **FR-007:** El sistema debe permitir exportar los indicadores y filtros aplicados en CSV o PDF.
- **FR-008:** El sistema debe restringir esta consulta al personal administrativo autorizado.
- **FR-009:** El sistema debe mostrar los indicadores generales y los rankings sin identificar individualmente a estudiantes.
- **FR-010:** El sistema debe indicar la fecha y hora de la última actualización de los datos.
- **FR-011:** El sistema debe construir los indicadores únicamente a partir de los datos de M3: check-outs, reportes de daño, sanciones, confirmaciones de inasistencia y cobros.
- **FR-012:** El sistema debe mostrar un estado vacío informativo cuando no existan datos para el período y los filtros elegidos.
- **FR-013:** El sistema debe permitir al personal administrativo buscar a un estudiante por su código institucional y ver su detalle: score e historial, bloqueo, sanciones, check-outs, préstamos vencidos, inasistencias, reportes de daño y cobros con su estado.
- **FR-014:** El sistema debe restringir el detalle por estudiante al personal administrativo autorizado.

## Key Entities
- **Check_out, Reporte_Daño, Sancion, Confirmacion_Inasistencia, Cobro:** Fuentes de datos de los indicadores.
- **Indicador:** Valor agregado calculado para un período y unos filtros (cantidad, porcentaje o monto).
- **Recurso / Espacio:** Dimensión de análisis, con tipo, ubicación y facultad tomados de las instantáneas guardadas al registrar cada evento.
- **Estudiante (consultado):** Sujeto del detalle nominal; el personal lo busca por código institucional.
- **Personal administrativo:** Único actor autorizado.

## Integración con Módulos Externos

| Módulo | Tipo de relación | Justificación |
|---|---|---|
| Módulos 1 y 2 | **Sin integración directa** | La analítica consulta solo datos de M3. Los datos descriptivos de recursos y espacios (tipo, ubicación) provienen de las instantáneas que M3 guarda a partir de las reservas que informa M2 y de los eventos que publica a M1 en los demás casos de uso. |

## Success Criteria

### Measurable Outcomes
- **SC-001:** El panel debe cargar los indicadores de un semestre en menos de 5 segundos.
- **SC-002:** El 100% de los indicadores deben coincidir con los datos de origen de M3 para el mismo período y filtros.
- **SC-003:** El 0% de las vistas de indicadores generales y rankings deben exponer datos que identifiquen individualmente a un estudiante.
- **SC-004:** El 100% de los intentos de usuarios sin permisos deben ser rechazados.
- **SC-005:** La exportación de un período debe generarse en menos de 10 segundos y contener los mismos indicadores que el panel.
- **SC-006:** El detalle de un estudiante debe cargar en menos de 3 segundos y coincidir con los datos de origen de M3.
