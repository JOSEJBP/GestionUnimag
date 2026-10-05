# Implementation Plan: Reportar novedad técnica — daño (UC8)

**Date**: 2026-10-05
**Spec**: [reportar-novedad-tecnica.md](../Especs/reportar-novedad-tecnica.md)
**Plan general**: [plan.md](../Especs/plan.md)
**Módulo**: Módulo 3 — Sanciones y Cumplimiento
**Planes previos**: plan-uc-realizar-check-out.md — UC6 lo invoca (`«extend»`) cuando el estudiante indica un daño al hacer check-out; aporta la `Usage` local
**Planes relacionados**: [plan-uc3-bloquear-usuario.md](./plan-uc3-bloquear-usuario.md) — expone `BlockForDamagePort`, pero la spec de UC8 **no** pide bloquear (ver NC-05)
**Contratos Kafka compartidos**: [plan-integracion-kafka.md](./plan-integracion-kafka.md) — consolida las integraciones de M3 con M1 y M2; el evento M1 `DamageReport` y sus campos aún por confirmar

## Summary

UC8 registra un daño y lo comunica a Módulo 1 para que el dueño del inventario actualice el estado del recurso según su contrato. Puede originarse de dos formas: el estudiante lo reporta **durante el check-out** (P1, `«extend»` de UC6) o la **dirección universitaria** lo reporta de forma independiente, por ejemplo tras una revisión de inventario (P2).

UC8 **no bloquea al estudiante** (UC3) y comunica el daño mediante el evento Kafka que define M1 (`DamageReport`). El contrato disponible no define una confirmación de consumo de vuelta a M3; M3 solo puede informar el estado de publicación, no asegurar que M1 ya actualizó el recurso.

**Enfoque técnico:**

1. Expone un **puerto de entrada** (`ReportDamagePort`) con dos operaciones de registro: `reportDuringCheckOut` (lo llama UC6 en memoria) y `reportIndependent` (lo llama el controlador REST de dirección). Ambas convergen en la misma lógica; solo cambia cómo se identifica la utilización.
2. Cuando el daño corresponde a una utilización identificable, el reporte conserva esa relación para trazabilidad y deduplicación (FR-003, FR-006).
3. El registro local del reporte y la inserción del evento en el outbox son atómicos. El publicador Kafka compartido administra los reintentos de entrega.
4. El evento `DamageReport` informa a M1 del daño; M1 decide y actualiza el estado del recurso. La entrega al broker no se interpreta como confirmación de que M1 ya aplicó el cambio.
5. La **evidencia** (fotografías) es opcional y se adjunta en una operación aparte, para no complicar el flujo de check-out con subidas de archivos (FR-004).
6. El UC tiene **4 user stories**: reporte durante el check-out (US1), reporte independiente (US2), manejo de rechazos y de fallas de publicación (US3) y evidencia + consulta + pantalla (US4). La spec declara US1 y US2; US3 y US4 se derivan de los Edge Cases y de FR-004.

## Technical Context

**Language/Version**: Java 21 (backend); JavaScript con React + Vite (frontend)
**Primary Dependencies**: Spring Boot 4.x (Web, Data JPA, Validation, Security OAuth2 Resource Server, Spring for Apache Kafka, Flyway, springdoc-openapi). Reutiliza el outbox y la configuración Kafka de M3.
**Storage**: MySQL 8 para `damage_report`, `damage_evidence` y los metadatos de evidencia; UC8 **lee** `usage` (de UC6) para identificar la utilización. Guardar los bytes en almacenamiento persistente separado, tras `EvidenceStoragePort`, es una propuesta pendiente de concretar en NC-04; MySQL no almacena los archivos.
**Testing**: JUnit 5 + Mockito (dominio y casos de uso), `@WebMvcTest` (controladores), Testcontainers MySQL y Kafka (persistencia, unicidad, outbox/publicación), ArchUnit (regla de dependencias entre capas).
**Target Platform**: Servidor Linux con JVM 21; navegador web para el frontend.
**Project Type**: Web: backend y frontend separados (React + Vite).
**Performance Goals**: registrar el reporte en **menos de 2 segundos** desde que se recibe la información (SC-004). El flujo de registro solo persiste el reporte y el evento en el outbox; no espera a que M1 procese el mensaje.
**Constraints**:
- Un solo reporte de daño por utilización, garantizado por clave única en BD (FR-006, SC-003).
- Descripción obligatoria (FR-007); la evidencia es opcional (FR-004).
- El cambio de estado del recurso es responsabilidad de M1; publicar el evento no equivale a confirmar la transición.
- Si el reporte no puede registrarse, no debe quedar un registro incompleto: T1 es atómica.
- La zona horaria es `America/Bogota`.
**Scale/Scope**: Mismo orden que M2 (miles de estudiantes y recursos). Una sección de daño dentro de la pantalla de check-out y una pantalla de reporte/consulta para dirección.

## Integración con otros módulos

### Lo que M3 consume de M1 y M2 (adoptado tal cual)

**UC8 no consume eventos de M1 ni de M2.** Su disparador es una persona (estudiante en el check-out, o dirección) y, internamente, UC6.

M3 no consulta ni recibe el costo del recurso desde M1.

### Lo que M3 produce para M1 y M2 (creado por M3)

M3 publica `DamageReport` para M1 conforme a los nombres del borrador de M1; el topic, la clave Kafka y varios campos requieren confirmación. Ver [plan-integracion-kafka.md](./plan-integracion-kafka.md). M1 consume el evento y decide/aplica la actualización del recurso. La publicación al broker no confirma su consumo ni la aplicación del cambio.

### Endpoints que M3 expone para M2 y M1

**UC8 no expone endpoints a M1 ni a M2.**

### Endpoints que M3 expone para su propio frontend

| Endpoint | Método | Para qué | Rol |
|---|---|---|---|
| *(dentro de `POST` de check-out de UC6)* | POST | El estudiante indica el daño al hacer check-out; UC6 llama a `reportDuringCheckOut` | `ESTUDIANTE` o `MONITOR` |
| `POST /api/v1/admin/damage-reports` | POST | Dirección reporta un daño de forma independiente | `ADMIN` o `DIRECCION_PROGRAMA` |
| `GET /api/v1/damage-reports/{id}` | GET | Consultar un reporte (el estudiante solo si es el titular de la utilización) | `ESTUDIANTE`, `MONITOR`, `ADMIN` o `DIRECCION_PROGRAMA` |
| `GET /api/v1/admin/damage-reports` | GET | Listar reportes; filtrar por `publicationStatus` o `resourceId` | `ADMIN` o `DIRECCION_PROGRAMA` |
| `POST /api/v1/damage-reports/{id}/evidence` | POST (multipart) | Adjuntar fotografías a un reporte | quien reportó, o `ADMIN` |
| `GET /api/v1/damage-reports/{id}/evidence/{evidenceId}` | GET | Descargar una evidencia | `ADMIN`, `DIRECCION_PROGRAMA` o el titular |

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan.md                                    # Plan general del M3
├── plan-integracion-kafka.md                   # Contratos Kafka M3↔M1/M2 y asuntos por confirmar
├── reportar-novedad-tecnica.md                 # Spec de este plan
├── realizar-checkout.md                       # «extend» que lo invoca
└── ...                                         # resto de specs
```

### Source Code (repository root)

Solo los archivos que crea o toca este plan. La organización completa está en `plan.md`.

```text
backend/
├── src/main/java/com/university/sanctions/
│   ├── presentation/
│   │   ├── controller/
│   │   │   ├── DamageReportController.java            # GET {id}, evidencia (subir/descargar)
│   │   │   └── AdminDamageReportController.java       # POST y GET list
│   │   └── dto/
│   │       ├── DamageReportRequest.java
│   │       ├── DamageReportResponse.java
│   │       ├── DamageReportListResponse.java
│   │       └── DamageEvidenceResponse.java
│   │
│   ├── business/
│   │   ├── service/
│   │   │   ├── DamageReportService.java               # implementa ReportDamagePort
│   │   │   ├── DamageReportRegistrar.java             # @Transactional: validar + guardar + outbox
│   │   │   └── DamageEvidenceService.java             # implementa AttachDamageEvidencePort
│   │
│   ├── domain/
│   │   ├── model/
│   │   │   ├── DamageReport.java                      # recurso, utilización, descripción y publicación
│   │   │   ├── DamageEvidence.java                    # metadatos de una fotografía
│   │   │   ├── DamageSource.java                      # enum: CHECK_OUT, INDEPENDENT
│   │   │   ├── DamagePublicationStatus.java           # enum: PENDING, PUBLISHED, FAILED
│   │   │   └── UsageView.java                         # vista mínima de la utilización (de UC6)
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   ├── ReportDamagePort.java              # puerto de entrada
│   │   │   │   └── AttachDamageEvidencePort.java
│   │   │   └── out/
│   │   │       ├── DamageReportRepository.java
│   │   │       ├── DamageEvidenceRepository.java
│   │   │       ├── UsageLookupPort.java               # utilización por id / última por recurso (de UC6)
│   │   │       ├── DamageReportPublisherPort.java     # publicación del evento a M1
│   │   │       └── EvidenceStoragePort.java           # guardar/leer archivos
│   │   ├── event/
│   │   │   ├── DamageReportRegisteredEvent.java       # evento interno para persistir en outbox
│   │   │   └── DamageReportRegistrationRejected.java  # rechazo conocido comunicado al llamador UC6
│   │   └── error/
│   │       ├── ResourceNotIdentifiedException.java
│   │       ├── MissingDescriptionException.java
│   │       ├── DuplicateDamageReportException.java
│   │       ├── UsageNotDeterminableException.java
│   │       ├── DamageReportNotFoundException.java
│   │       └── InvalidEvidenceException.java
│   │
│   └── infrastructure/
│       ├── persistence/
│       │   └── jpa/
│       │       ├── entity/
│       │       │   ├── DamageReportEntity.java
│       │       │   └── DamageEvidenceEntity.java
│       │       ├── repository/
│       │       │   ├── JpaDamageReportRepository.java
│       │       │   └── JpaDamageEvidenceRepository.java
│       │       ├── adapter/
│       │       │   ├── DamageReportRepositoryAdapter.java
│       │       │   └── DamageEvidenceRepositoryAdapter.java
│       │       └── mapper/
│       │           ├── DamageReportMapper.java
│       │           └── DamageEvidenceMapper.java
│       ├── integration/
│       │   └── module1/
│       │       └── kafka/
│       │           └── DamageReportEventMapper.java  # mapea al esquema acordado con M1
│       ├── messaging/
│       │   ├── kafka/
│       │   │   └── DamageReportPublisher.java        # publica el evento pendiente del outbox
│       │   └── outbox/                                # outbox compartido de M3
│       ├── storage/
│       │   └── LocalEvidenceStorage.java              # implementa EvidenceStoragePort sobre un volumen
│       └── config/
│           ├── SecurityConfig.java
│           └── UseCasesConfig.java
│
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       ├── V12__damage_report.sql
│       └── V13__damage_evidence.sql
│
└── src/test/java/com/university/sanctions/
    ├── contract/
    │   ├── AdminDamageReportControllerTest.java
    │   ├── DamageReportControllerTest.java
    │   └── DamageReportEventContractTest.java          # payload/topic/key según M1
    ├── integration/
    │   ├── DamageReportIT.java                         # persistencia, unicidad y outbox atómico
    │   ├── DamageReportPublishIT.java                  # publicación desde outbox a Kafka
    │   └── DamageEvidenceIT.java
    └── unit/
        ├── DamageReportServiceTest.java
        ├── DamageReportRegistrarTest.java
        └── DamageReportPublisherTest.java

frontend/
└── src/
    ├── pages/
    │   └── DamageReportPage.jsx                        # reporte independiente + consulta (dirección)
    ├── components/
    │   ├── DamageReportForm.jsx                        # descripción + evidencia; reusado dentro de CheckOutPage
    │   ├── DamageEvidenceUploader.jsx                  # selector de fotos con vista previa
    │   └── DamagePublicationBadge.jsx                  # estado de entrega al broker
    └── services/
        └── damageReportApi.js                          # cliente HTTP
```

**Structure Decision**: se respeta la estructura de capas de `plan.md` con el paquete raíz `com.university.sanctions`. Los puertos viven en `domain/port`; el adaptador Kafka de M1 y la publicación mediante outbox viven en infraestructura. Reporte y outbox se guardan en la misma transacción. El frontend reutiliza `DamageReportForm.jsx` dentro de la pantalla de check-out de UC6.

## Decisiones de diseño de este caso de uso

**Dónde queda cada FR.**

| FR | Qué pide | Dónde se implementa |
|---|---|---|
| FR-001 | Reportar daño como parte del check-out (`«extend»`) | `ReportDamagePort.reportDuringCheckOut(...)` — lo llama UC6 |
| FR-002 | Reporte independiente por dirección universitaria | `AdminDamageReportController.POST` → `ReportDamagePort.reportIndependent(...)` |
| FR-003 | Identificar recurso, utilización y estudiante | `UsageLookupPort` + `DamageReportRegistrar` (regla de vinculación, ver decisión 3) |
| FR-004 | Registrar descripción y, si existe, evidencia | Descripción en T1; evidencia con `AttachDamageEvidencePort` (`DamageEvidenceService`) |
| FR-005 | Comunicar el reporte a M1 para que actualice el recurso | Outbox + `DamageReportPublisher` publica el evento M1 |
| FR-006 | Impedir reporte duplicado por utilización | Clave única en `damage_report.usage_id` + verificación previa |
| FR-007 | Informar recurso no identificado o descripción faltante | `ResourceNotIdentifiedException`, `MissingDescriptionException` → `ProblemDetail` |

**Decisiones justificadas.**

1. **El reporte local y el evento son atómicos.** UC8 no llama a M1 por REST. La transacción de M3 guarda el reporte y su evento en el outbox; una falla posterior de Kafka no pierde el reporte y se gestiona con los reintentos compartidos del publicador.

2. **M1 es responsable de la transición del inventario.** M3 publica el evento con el contrato de M1; el estado de publicación no representa un acuse de M1 ni se refleja como estado final del recurso.

3. **Regla de vinculación a una utilización.** Es el punto más delicado de la spec:
   - *Durante el check-out*: la utilización es la del check-out en curso; UC6 pasa `usageId`.
   - *Independiente, con `usageId` explícito*: se valida que la utilización exista y que corresponda al `resourceId` indicado.
   - *Independiente, sin `usageId`*: se usa la **última utilización cerrada** del recurso, siempre que el recurso **no esté actualmente en uso por otra persona**.
   - *Recurso actualmente en uso (posible reasignación)*: no se puede saber con certeza si el daño ocurrió en la utilización anterior o en la actual. La spec permite "vincular a la última utilización identificable antes de la reasignación, o rechazarlo si no puede determinarse con certeza". Se elige la opción **conservadora**: rechazar con `USAGE_NOT_DETERMINABLE` salvo que dirección indique `usageId`. No se atribuye un daño a una persona si no se puede establecer la utilización correcta.
   - *Recurso sin ninguna utilización*: se registra el daño asociado al recurso, con `usageId` y `studentCode` vacíos; no se atribuye el reporte a una persona (NC-03).

4. **Unicidad por utilización.** `usage_id` es opcional y único cuando existe, conforme a FR-006; los reportes no asociados a una utilización no se atribuyen a un estudiante.

5. **Un reporte de daño inválido no debe bloquear el check-out.** UC6 declara que el check-out se registra normalmente aunque el reporte de daño no pueda iniciarse. Para un rechazo de negocio conocido, UC8 no guarda reporte ni evento M1 y emite el evento interno `DamageReportRegistrationRejected`. El consumidor previsto en UC6 queda pendiente (NC-02); los fallos inesperados de infraestructura se exponen explícitamente.

6. **M3 no elige el estado de inventario.** El evento informa el daño según el contrato de M1; M1 determina y aplica la transición permitida.

7. **Idempotencia del evento.** Se conserva el mismo identificador al reintentar la publicación; el nombre canónico del campo debe confirmarse por la inconsistencia `eventoId`/`eventId` (NC-01). La intención documentada es usar el recurso como clave para preservar su orden; su nombre y tipo están pendientes.

8. **Reintento de publicación compartido.** Los errores de envío a Kafka se procesan por la infraestructura outbox común de M3. UC8 no define un scheduler, endpoint de reintento ni confirmación de consumo que el contrato de M1 no ofrece.

9. **La evidencia se sube aparte del reporte.** Enviar archivos dentro del flujo de check-out de UC6 obligaría a convertir ese endpoint en `multipart` y alargaría el camino crítico. Con `POST /damage-reports/{id}/evidence`, el reporte se registra rápido (SC-004) y las fotos llegan después. Límites propuestos: máximo 5 archivos por reporte, 5 MB cada uno, solo `image/jpeg` y `image/png` (NC-04).

10. **UC8 no bloquea al estudiante (por ahora).** El plan de UC3 dice que UC8 llama a `BlockForDamagePort.block` "cuando hay daño grave", pero la spec de UC8 no define "daño grave", ni gravedad, ni bloqueo. Para no inventar reglas, **no se implementa** esa llamada y se registra como **NC-05**.

11. **UC8 publica el evento Kafka de M1.** El evento se escribe en el outbox junto con el reporte y se envía al topic propuesto por M1, cuyo nombre final queda pendiente de NC-01.

12. **El reporte no se elimina ni se edita.** Es evidencia de la novedad comunicada a M1; se pueden añadir evidencias.

## Contratos

Se aplican las convenciones comunes de M2 (camelCase, MAYUSCULA_CON_GUION_BAJO, nunca null, ISO-8601 con `-05:00`, RFC 9457). M3 las adopta para sus propios contratos.

### 1. Puerto de entrada — `ReportDamagePort`

```java
public interface ReportDamagePort {

    /**
     * Lo llama Realizar check-out (UC6) cuando el estudiante indica un daño.
     * Un rechazo de negocio se devuelve como resultado y no impide completar UC6.
     */
    ReportDamageAttemptResult reportDuringCheckOut(ReportDamageDuringCheckOutCommand command);

    /**
     * Lo llama el controlador de dirección universitaria.
     * @throws UsageNotDeterminableException    no se puede vincular con certeza a una utilización
     * (más las excepciones anteriores)
     */
    DamageReportResult reportIndependent(ReportDamageIndependentCommand command);

}

public sealed interface ReportDamageAttemptResult
    permits DamageReportAccepted, DamageReportRejected {}

public record DamageReportAccepted(DamageReportResult report)
    implements ReportDamageAttemptResult {}

public record DamageReportRejected(DamageReportRegistrationRejected rejection)
    implements ReportDamageAttemptResult {}

public record ReportDamageDuringCheckOutCommand(
    String usageId,
    String description,
    String reporterCode          // el estudiante, del JWT
) {}

public record ReportDamageIndependentCommand(
    String resourceId,
    String usageId,              // opcional
    String description,
    String reporterCode          // dirección, del JWT
) {}

public record DamageReportResult(
    String reportId,
    String resourceId,
    String usageId,
    String studentCode,
    DamagePublicationStatus publicationStatus,
    Instant reportedAt
) {}
```

### 2. Puerto de entrada — `AttachDamageEvidencePort`

```java
public interface AttachDamageEvidencePort {
    DamageEvidenceView attach(String reportId, String fileName, String contentType,
                              long sizeBytes, InputStream content, String actorCode);
}
```

### 3. Puertos de salida

`UsageLookupPort` — se alimenta de la `usage` local de UC6:

```java
public interface UsageLookupPort {
    Optional<UsageView> findById(String usageId);
    Optional<UsageView> findLastClosedByResource(String resourceId);
    Optional<UsageView> findActiveByResource(String resourceId);
}

public record UsageView(String usageId, String resourceId, String studentCode,
                        Instant startedAt, Instant endedAt) {}  // endedAt nulo si sigue en curso
```

### 4. Contrato Kafka con M1

El contrato se centraliza en [plan-integracion-kafka.md](./plan-integracion-kafka.md). Allí se documentan los nombres usados en `KAFKA.md` y `PLAN (2).md`, el ejemplo de `DamageReport` y sus inconsistencias. No se fija un DTO final ni se cambia la nomenclatura del borrador hasta confirmar el esquema con M1. El evento se guarda en el outbox junto con el reporte; el publicador compartido lo entrega a Kafka y conserva/reintenta fallos de publicación. El acuse del broker no confirma consumo ni actualización del recurso por M1.

### 5. Contrato de rechazo del intento desde UC6

Si UC8 rechaza el intento por un motivo de negocio conocido, produce el evento interno `DamageReportRegistrationRejected`, definido con su esquema en [plan-integracion-kafka.md](./plan-integracion-kafka.md). No se publica a Kafka ni a M1. UC6 es el consumidor previsto, pero los otros planes aún no lo procesan. El rechazo no crea reporte ni evento `DamageReport`; los errores inesperados de infraestructura se propagan explícitamente.

### 6. Endpoints REST de M3

#### `POST /api/v1/admin/damage-reports`

Rol `ADMIN` o `DIRECCION_PROGRAMA`. El reportante sale del JWT.

```json
{
  "resourceId": "ACT-004512",
  "usageId": "usg-20260930-0187",
  "description": "La pantalla del portátil presenta una fisura en la esquina inferior derecha."
}
```

`usageId` es opcional cuando la novedad puede identificarse sin una utilización; las reglas de vinculación se validan antes del registro. `201 Created` confirma el registro local del reporte y del mensaje outbox, no el procesamiento de M1:

```json
{
  "id": "5a8f1c2e-3b74-4d09-9e16-7c2b0a4d8f35",
  "resourceId": "ACT-004512",
  "usageId": "usg-20260930-0187",
  "studentCode": "2023123456",
  "source": "INDEPENDENT",
  "description": "La pantalla del portátil presenta una fisura en la esquina inferior derecha.",
  "reportedBy": "direccion@unimagdalena.edu.co",
  "reportedAt": "2026-10-05T11:15:00-05:00",
  "publicationStatus": "PENDING",
  "evidence": []
}
```

#### Reporte durante el check-out

No tiene endpoint propio: UC6 incluye un bloque opcional `damage` en el check-out y delega su registro a `reportDuringCheckOut(...)`:

```json
{
  "damage": {
    "hasDamage": true,
    "description": "La tecla Enter quedó suelta."
  }
}
```

La respuesta del check-out puede incluir `damageReport` con su identificador y `publicationStatus`; no afirma que M1 ya actualizó el recurso.

#### Consulta y evidencia

- `GET /api/v1/damage-reports/{id}` devuelve el reporte; el estudiante solo puede consultar el de su utilización.
- `GET /api/v1/admin/damage-reports?publicationStatus=FAILED&resourceId=ACT-004512&page=1&pageSize=20` lista reportes con filtros y paginación. Los fallos se refieren a publicación, no a actualización del inventario por M1.
- `POST /api/v1/damage-reports/{id}/evidence` recibe `multipart/form-data` con un campo `file`; respuesta `201 Created` con metadatos de evidencia.
- `GET /api/v1/damage-reports/{id}/evidence/{evidenceId}` descarga evidencia autorizada.

No se define endpoint UC8 para reintentar ni para cambiar el estado en M1; los reintentos Kafka pertenecen al publicador/outbox compartido.

### 7. Errores

| Código HTTP | `code` | Cuándo |
|---|---|---|
| 400 | `MISSING_DESCRIPTION` | Falta la descripción del daño (FR-007) |
| 400 | `INVALID_EVIDENCE` | Tipo no permitido, archivo > 5 MB o más de 5 archivos |
| 404 | `RESOURCE_NOT_IDENTIFIED` | No fue posible identificar el recurso asociado (FR-007) |
| 404 | `DAMAGE_REPORT_NOT_FOUND` | El reporte consultado no existe |
| 409 | `DUPLICATE_DAMAGE_REPORT` | Ya existe un reporte para la utilización (FR-006) |
| 409 | `USAGE_NOT_DETERMINABLE` | No se puede vincular con certeza a una utilización (por ejemplo, recurso reasignado) |
| 500 | `PERSISTENCE_ERROR` | Fallo al registrar; no queda registro incompleto |

`ProblemDetail` de ejemplo:

```json
{
  "type": "https://sanctions.unimagdalena.edu.co/errors/usage-not-determinable",
  "title": "No se puede determinar la utilización",
  "status": 409,
  "detail": "El recurso ACT-004512 está en uso por otra persona; indique la utilización (usageId) a la que corresponde el daño.",
  "instance": "/api/v1/admin/damage-reports",
  "code": "USAGE_NOT_DETERMINABLE",
  "resourceId": "ACT-004512"
}
```

### 8. Tablas

#### `damage_report`

| Columna | Tipo | Nota |
|---|---|---|
| `id` | varchar(36) PK | UUID |
| `resource_id` | varchar(50) NOT NULL | |
| `usage_id` | varchar(50), nulo | **UNIQUE** cuando existe (FR-006) |
| `student_code` | varchar(20), nulo | Titular, cuando se identifica utilización |
| `source` | varchar(20) NOT NULL | `CHECK_OUT`, `INDEPENDENT` |
| `description` | text NOT NULL | |
| `reported_by` | varchar(100) NOT NULL | Del JWT |
| `reported_at` | timestamp NOT NULL | |
| `publication_status` | varchar(20) NOT NULL | `PENDING`, `PUBLISHED`, `FAILED` (publicación al broker solamente) |
| `created_at` | timestamp NOT NULL | |
| `updated_at` | timestamp NOT NULL | |

```sql
CREATE UNIQUE INDEX damage_report_usage ON damage_report (usage_id);
CREATE INDEX damage_report_resource ON damage_report (resource_id, reported_at DESC);
CREATE INDEX damage_report_student ON damage_report (student_code, reported_at DESC);
CREATE INDEX damage_report_publication ON damage_report (publication_status, reported_at);
```

#### `damage_evidence`

| Columna | Tipo | Nota |
|---|---|---|
| `id` | varchar(36) PK | |
| `report_id` | varchar(36) FK NOT NULL | |
| `file_name` | varchar(255) NOT NULL | |
| `content_type` | varchar(50) NOT NULL | |
| `size_bytes` | bigint NOT NULL | |
| `storage_key` | varchar(255) NOT NULL | Ruta relativa dentro del volumen |
| `uploaded_by` | varchar(100) NOT NULL | |
| `uploaded_at` | timestamp NOT NULL | |

```sql
CREATE INDEX damage_evidence_report ON damage_evidence (report_id);
```

La clave única opcional en `usage_id` evita duplicados para reportes vinculados. El criterio para evitar duplicados cuando no hay utilización asociada debe definirse junto con el esquema canónico de M1.

### 9. Tipos del frontend

```js
export const DamagePublicationStatus = {
  PENDING: "PENDING",
  PUBLISHED: "PUBLISHED",
  FAILED: "FAILED",
};

export const reportDamage = async ({ resourceId, usageId, description }) => {
  const response = await fetch("/api/v1/admin/damage-reports", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ resourceId, usageId, description }),
  });
  if (!response.ok) throw await response.json();
  return response.json();
};

export const uploadDamageEvidence = async (reportId, file) => {
  const body = new FormData();
  body.append("file", file);
  const response = await fetch(`/api/v1/damage-reports/${reportId}/evidence`, {
    method: "POST",
    body,
  });
  if (!response.ok) throw await response.json();
  return response.json();
};

export const getDamageReports = async ({ publicationStatus, resourceId, page = 1 } = {}) => {
  const params = new URLSearchParams({ page });
  if (publicationStatus) params.set("publicationStatus", publicationStatus);
  if (resourceId) params.set("resourceId", resourceId);
  const response = await fetch(`/api/v1/admin/damage-reports?${params}`);
  if (!response.ok) throw await response.json();
  return response.json();
};
```

### 10. Fixtures compartidos

```text
backend/src/test/resources/contracts/
├── api-damage-report-201-created.json
├── api-damage-report-201-created-pending.json
├── api-damage-reports-list-200.json
├── api-damage-evidence-201.json
├── api-damage-report-400-missing-description.json
├── api-damage-report-404-resource.json
├── api-damage-report-409-duplicate.json
├── api-damage-report-409-usage-not-determinable.json
├── m1-damage-report-event-v1.json
└── m1-damage-report-invalid-event.json
```

## Phase 1: Setup

- [ ] T001 Configurar las propiedades Kafka compartidas y los límites de evidencia. El topic propuesto `unimag.m3.uso-novedades.v1` y la clave de recurso quedan pendientes de NC-01; los valores de broker se toman de la configuración común de M3.
- [ ] T002 [P] Extender `SanctionsProperties` con topic y límites de evidencia; validar las propiedades al arrancar.
- [ ] T003 [P] Añadir un volumen `evidence` al `docker-compose.yml` y configurar la ruta de almacenamiento local.
- [ ] T004 Verificar que el publicador/outbox Kafka compartido está habilitado; UC8 no crea un scheduler de reintentos propio.

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: persistir el reporte y la evidencia, vincular utilización cuando aplique y guardar el evento M1 en outbox atómicamente.

- [ ] T005 Crear `V12__damage_report.sql` con reporte, estado de publicación y unicidad opcional por utilización; revisar el índice con los casos sin `usage_id`.
- [ ] T006 Crear `V13__damage_evidence.sql` con la tabla y sus índices.
- [ ] T007 [P] Crear modelos `DamageReport`, `DamageEvidence`, `DamageSource`, `DamagePublicationStatus` y `UsageView`; no crear un estado destino del recurso que pertenece a M1.
- [ ] T008 [P] Definir `ReportDamagePort` y `AttachDamageEvidencePort` en `domain/port/in/`.
- [ ] T009 [P] Definir `DamageReportRepository`, `DamageEvidenceRepository`, `UsageLookupPort`, `DamageReportPublisherPort` y `EvidenceStoragePort` en los puertos de salida.
- [ ] T010 [P] Crear `DamageReportRegisteredEvent` y las excepciones de dominio.
- [ ] T011 [P] Implementar entidades, repositorios Spring Data, adaptadores y mappers de reporte/evidencia.
- [ ] T012 [P] Implementar `UsageLookupPort` sobre la utilización local de UC6; admitir reportes sin utilización cuando no se pueda vincular una.
- [ ] T013 [P] Implementar el mapper y adaptador Kafka de `DamageReport` con los nombres confirmados por M1, documentados en [plan-integracion-kafka.md](./plan-integracion-kafka.md); no inferir ni corregir campos ambiguos antes de NC-01.
- [ ] T014 [P] Implementar `LocalEvidenceStorage` sobre el volumen.
- [ ] T015 Verificar/usar el outbox y el publicador compartidos de M3 para persistir y enviar el evento, incluidos sus reintentos de publicación.

**Checkpoint**: reporte, referencia opcional a utilización y evento outbox quedan persistidos en una única transacción; M3 no llama a M1 por REST.

## Phase 3: User Story 1 — Reporte de daño durante el check-out (Priority: P1)

**Goal**: cuando el estudiante indica un daño en el check-out, UC6 registra el reporte vinculado a esa utilización y deja el evento de M1 en el outbox.

**Independent Test**: simular un check-out con `damage.hasDamage = true` y descripción; comprobar que el reporte tiene `source = CHECK_OUT`, que el mensaje se guarda en outbox en la misma transacción y que se publica a Kafka bajo el topic y clave definidos por M1. No comprobar un acuse de procesamiento de M1 porque el contrato no lo define.

### Tests for User Story 1

- [ ] T016 [P] [US1] Pruebas unitarias: `reportDuringCheckOut` conserva `usageId`, recurso y estudiante, y produce el evento interno de reporte.
- [ ] T017 [P] [US1] Prueba MySQL: reporte y outbox son atómicos; si falla el registro del outbox, no queda reporte parcial.
- [ ] T018 [P] [US1] Prueba de integración: si la transacción completa de check-out se revierte, tampoco quedan reporte/outbox; si el subflujo de daño se rechaza por una regla de negocio, se emite `DamageReportRegistrationRejected` y el check-out puede completarse sin reporte.
- [ ] T019 [P] [US1] Prueba de contrato del payload/topic/key usando el esquema acordado con M1; el fixture no debe codificar las erratas del borrador recibido.

### Implementation for User Story 1

- [ ] T020 [US1] Implementar `DamageReportRegistrar.register(...)`: validar, guardar el reporte y guardar `DamageReport` en outbox dentro de T1.
- [ ] T021 [US1] Conectar el mensaje outbox con el publicador Kafka compartido, manteniendo el mismo identificador de evento en reintentos.
- [ ] T022 [US1] Implementar `DamageReportService.reportDuringCheckOut(...)`.
- [ ] T023 [US1] Coordinar con UC6: añadir el bloque opcional `damage`, delegar a `reportDuringCheckOut` y exponer el identificador del reporte sin afirmar que M1 ya aplicó el cambio; emitir `DamageReportRegistrationRejected` ante rechazos de negocio conocidos.

**Checkpoint**: el camino P1 guarda el reporte y publica el evento del contrato de M1; el resultado de M1 no se simula ni se espera sin contrato de acuse.

## Phase 4: User Story 2 — Reporte de daño fuera del flujo de check-out (Priority: P2)

**Goal**: dirección universitaria registra un daño detectado fuera del check-out; el reporte se vincula a una utilización cuando puede identificarse con certeza.

**Independent Test**: `POST /api/v1/admin/damage-reports` con recurso y descripción; comprobar `source = INDEPENDENT`, la vinculación opcional de utilización y un evento outbox publicado a M1.

### Tests for User Story 2

- [ ] T024 [P] [US2] Pruebas de vinculación: utilización explícita del recurso → acepta; utilización de otro recurso → rechaza; recurso en uso sin vínculo inequívoco → `USAGE_NOT_DETERMINABLE`; sin historial y sin `usageId` → registra sin atribuir estudiante.
- [ ] T025 [P] [US2] Prueba `AdminDamageReportControllerTest.java`: dirección → 201; estudiante sin permiso → 403; sin sesión → 401.
- [ ] T026 [P] [US2] Prueba de integración: reporte independiente crea exactamente un evento en outbox sin seleccionar el estado final del recurso.

### Implementation for User Story 2

- [ ] T027 [US2] Implementar la regla de vinculación opcional usando `UsageLookupPort`.
- [ ] T028 [US2] Implementar `DamageReportService.reportIndependent(...)`.
- [ ] T029 [US2] Implementar `AdminDamageReportController` y DTOs; tomar `reportedBy` del JWT.

**Checkpoint**: dirección puede informar daños sin convertir a M3 en autoridad del inventario.

## Phase 5: User Story 3 — Validaciones y fallas de publicación (Priority: P1)

**Goal**: rechazar reportes inválidos, impedir duplicados donde haya utilización asociada y conservar/reintentar eventos si Kafka no está disponible.

### Tests for User Story 3

- [ ] T030 [P] [US3] Validación de descripción vacía, recurso no identificado y utilización no determinable; una solicitud rechazada no crea reporte ni outbox.
- [ ] T031 [P] [US3] Prueba concurrente de unicidad por utilización; dos reportes no generan eventos duplicados para el mismo uso.
- [ ] T032 [P] [US3] Prueba outbox: broker no disponible deja el evento pendiente y el publicador compartido lo reintenta; distinguir estado de publicación de procesamiento de M1.
- [ ] T033 [P] [US3] Prueba de integración de idempotencia del consumidor M1 por identificador de evento, coordinada con M1 si existe entorno/contrato de prueba.

### Implementation for User Story 3

- [ ] T034 [US3] Mapear `ResourceNotIdentifiedException`, `MissingDescriptionException`, `DuplicateDamageReportException` y `UsageNotDeterminableException` a `ProblemDetail`.
- [ ] T035 [US3] Convertir colisiones de unicidad de `usage_id` a `DuplicateDamageReportException`.
- [ ] T036 [US3] Exponer `publicationStatus` solo como estado de entrega a Kafka; no modelar `SYNCED`, `needsReview` ni “estado M1 actualizado”.

**Checkpoint**: los errores locales son explícitos y los fallos de publicación no pierden el reporte ni implican una falsa confirmación de M1.

## Phase 6: User Story 4 — Evidencia, consulta y pantallas (Priority: P3)

- [ ] T037 [P] [US4] Prueba de evidencia: archivo válido se almacena con metadatos; tipo no permitido, tamaño >5 MB o más de 5 archivos → `INVALID_EVIDENCE`; reporte inexistente → 404.
- [ ] T038 [P] [US4] Prueba de consulta y autorización del reporte/evidencia.
- [ ] T039 [P] [US4] Pruebas React: formulario, evidencia opcional y estado de publicación sin lenguaje de “recurso sincronizado”.
- [ ] T040 [US4] Implementar carga/descarga de evidencia y consultas paginadas.
- [ ] T041 [P] [US4] Implementar `damageReportApi.js`, `DamageReportForm.jsx` y `DamageEvidenceUploader.jsx` reutilizable por UC6.
- [ ] T042 [US4] Implementar `DamageReportPage.jsx` y mostrar el estado de publicación al broker, sin afirmar que M1 actualizó el inventario ni añadir un botón UC8 de reintento manual.

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T043 [P] Verificar SC-001 mediante métricas de publicación/outbox: todos los reportes aceptados producen un mensaje destinado a M1.
- [ ] T044 [P] Verificar SC-002 con métricas/confirmación del lado de M1: M3 no recibe acuse de procesamiento según el contrato disponible.
- [ ] T045 [P] Verificar unicidad de reportes asociados a una utilización (SC-003).
- [ ] T046 [P] Verificar SC-004: registro local y outbox <2 s; no incluir espera de procesamiento M1.
- [ ] T047 [P] Registrar creación del reporte, `eventId`, estado del outbox y fallos de publicación; no registrar costos ni afirmar actualización de M1.
- [ ] T048 [P] Prueba ArchUnit: `domain` y `business` no importan JPA, Kafka ni clases de `infrastructure`.
- [ ] T049 Documentar en README cómo reportar el daño y adjuntar evidencia.
- [ ] T050 Registrar pendientes del esquema y del acuse de M1 para seguimiento intermodular.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: depende de `plan.md` y de la configuración Kafka común.
- **Foundational (Phase 2)**: requiere el esquema canónico del evento DamageReport de M1 para fijar el mapper; bloquea las historias que publican el evento.
- **User Story 1 (Phase 3)**: depende de UC6 para invocar el puerto durante el check-out.
- **User Story 2 (Phase 4)**: comparte registro, validaciones y publicación con US1.
- **User Story 3 (Phase 5)**: completa las reglas locales y el manejo outbox compartido.
- **User Story 4 (Phase 6)**: depende del modelo y repositorios, no de la confirmación de M1.
- **Polish (Phase 7)**: depende de las historias anteriores y de observabilidad de M1 para medir SC-002.

### Dependencias con otros casos de uso del M3

- **Realizar check-out (UC6)**: invoca `reportDuringCheckOut` y aporta `usageId`; el contrato `DamageReportRegistrationRejected` ya está definido, pero el consumidor de UC6 queda para una iteración posterior (NC-02).
- **Calcular penalización**: permanece interno a M3 y no forma parte de UC8.
- **Bloquear usuario**: UC8 no bloquea hasta que se defina gravedad y regla de negocio (NC-05).

### Dependencias con otros módulos

- **Módulo 1**: consume `DamageReport` desde el topic y con la clave documentados por su contrato; ver [plan-integracion-kafka.md](./plan-integracion-kafka.md). M1 decide y actualiza el recurso. Es necesario confirmar los nombres y el esquema final.
- **Módulo 2**: sin integración directa en este UC.

## NEEDS CLARIFICATION abiertos

| Id | Pendiente | Estado/supuesto de trabajo |
|---|---|---|
| NC-01 | Contrato canónico M1: topic marcado `@@@`, identificador `eventoId`/`eventId`, errata `recourseCategory`, key, tipos y campos requeridos de `DamageReport` | Ver [plan-integracion-kafka.md](./plan-integracion-kafka.md); el borrador contiene marcadores pendientes, erratas y JSON inválido. |
| NC-02 | Consumo por UC6 del rechazo del reporte de daño (por ejemplo, falta la descripción) | El contrato `DamageReportRegistrationRejected` está definido en [plan-integracion-kafka.md](./plan-integracion-kafka.md), pero UC6 aún no lo maneja; el check-out no se bloquea. |
| NC-03 | Reporte de daño sobre recurso sin utilización asociable | El reporte puede identificar el recurso sin `usageId` ni estudiante. La persistencia debe permitir ambas referencias nulas para este caso. El borrador M1 no aclara si `id_reservation_m2` y `student` son opcionales, ni define una clave de deduplicación para reportes sin utilización; confirmar esos puntos antes de cerrar el contrato y la restricción de unicidad. |
| NC-04 | Almacenamiento, formatos y límites de evidencias | Propuesta: hasta 5 imágenes JPEG/PNG de 5 MB cada una (máximo potencial de 25 MB por reporte); MySQL guarda metadatos y la clave de almacenamiento, no los bytes. El almacenamiento persistente separado y `EvidenceStoragePort` son propuestas de UC8, no infraestructura definida en `plan.md`. Falta acordar capacidad total, retención, limpieza, copias de respaldo y recuperación; validar también el máximo acumulado por reporte. |
| NC-05 | Si un daño debe bloquear al estudiante y cómo se define “daño grave” | No se implementa bloqueo hasta acordar regla y contrato con UC3. |
| NC-06 | Confirmación de procesamiento de M1 hacia M3 | No está definida en el contrato Kafka recibido; `PUBLISHED` solo confirma entrega al broker. No añadir ack/evento de retorno por suposición. |

## Notes

- La numeración T0XX es propia de este plan.
- [P] tasks = different files, no dependencies.
- [US1] a [US4] = trazabilidad a historias de usuario.
- La sección Contratos define el API de M3; el payload intermodular se centraliza en [plan-integracion-kafka.md](./plan-integracion-kafka.md) y queda sujeto a NC-01.
- M1 es autoridad de inventario y aplica su transición a partir del evento.
- UC8 publica eventos a M1 mediante outbox/Kafka; `publicationStatus` describe únicamente la entrega al broker, no el procesamiento de M1.
