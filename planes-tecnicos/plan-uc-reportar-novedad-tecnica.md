# Implementation Plan: Reportar novedad técnica — daño (UC8)

**Date**: 2026-10-05
**Spec**: [reportar-novedad-tecnica.md](./reportar-novedad-tecnica.md)
**Plan general**: [plan.md](./plan.md)
**Módulo**: Módulo 3 — Sanciones y Cumplimiento
**Planes previos**: plan-uc-realizar-check-out.md — UC6 lo invoca (`«extend»`) cuando el estudiante indica un daño al hacer check-out; aporta la `Usage` local
**Planes relacionados**: generar-cobro.md — UC4 lo dispara obligatoriamente (`«include»`); [plan-uc3-bloquear-usuario.md](./plan-uc3-bloquear-usuario.md) — expone `BlockForDamagePort`, pero la spec de UC8 **no** pide bloquear (ver NC-05)

## Summary

UC8 deja **constancia de un daño** en un recurso universitario y pone en marcha sus consecuencias: cambia el estado del recurso en el Módulo 1 para que nadie más lo reciba y dispara la generación del cobro por daño o reposición (UC4). Puede originarse de dos formas: el estudiante lo reporta **durante el check-out** (P1, `«extend»` de UC6) o la **dirección universitaria** lo reporta de forma independiente, por ejemplo tras una revisión de inventario (P2).

UC8 **no calcula el monto del cobro** (UC4), **no bloquea al estudiante** (UC3) y **no produce eventos Kafka**: su única integración externa es una llamada REST **síncrona** al Módulo 1.

**Enfoque técnico:**

1. Expone un **puerto de entrada** (`ReportDamagePort`) con dos operaciones de registro: `reportDuringCheckOut` (lo llama UC6 en memoria) y `reportIndependent` (lo llama el controlador REST de dirección). Ambas convergen en la misma lógica; solo cambia cómo se identifica la utilización.
2. Toda novedad se **vincula a una utilización** (`usage_id`) y, por tanto, a un estudiante. Sin utilización no hay a quién cobrar, y es lo que permite impedir duplicados con una clave única (FR-003, FR-007).
3. El registro tiene **tres pasos**, porque no existe una transacción que abarque MySQL y el REST de M1 (`plan.md` §Module Communication):
   - **T1 (transacción local)**: validar + `INSERT damage_report` (`resource_sync_status = PENDING`) + disparar UC4 (`GenerateChargePort`). Todo atómico: si UC4 falla, no queda un reporte sin cobro ni un registro incompleto.
   - **Llamada a M1 (fuera de transacción)**, ejecutada *después del commit* mediante un listener `AFTER_COMMIT`: `ResourceModuleClient.changeStatus(...)`.
   - **T2 (transacción local corta)**: marcar el reporte `SYNCED`, o `FAILED` + `needs_review` si M1 falla.
4. Si M1 no confirma, **el reporte y el cobro se conservan** y se marca la inconsistencia para revisión (Edge Case de la spec, `plan.md`). Un job de reintento y un endpoint manual reenvían el cambio de estado; la llamada es idempotente.
5. La **evidencia** (fotografías) es opcional y se adjunta en una operación aparte, para no complicar el flujo de check-out con subidas de archivos (FR-004).
6. El UC tiene **4 user stories**: reporte durante el check-out (US1), reporte independiente (US2), manejo de rechazos y de la falla de M1 (US3) y evidencia + consulta + pantalla (US4). La spec declara US1 y US2; US3 y US4 se derivan de los Edge Cases y de FR-004.

## Technical Context

**Language/Version**: Java 21 (backend); JavaScript con React + Vite (frontend)
**Primary Dependencies**: Spring Boot 4.x (Web, Data JPA, Validation, Security OAuth2 Resource Server, Spring Scheduling), `RestClient` para M1, Flyway, springdoc-openapi. **Ninguna nueva** respecto a `plan.md`. **No usa Spring for Apache Kafka**: UC8 no publica ni consume Kafka.
**Storage**: MySQL 8. UC8 **escribe** `damage_report` y `damage_evidence`. **Lee** `usage` (de UC6) para identificar la utilización. Los archivos de evidencia se guardan en un volumen de disco detrás de `EvidenceStoragePort` (NC-04).
**Testing**: JUnit 5 + Mockito (dominio y casos de uso), `@WebMvcTest` (controladores), Testcontainers MySQL (persistencia, unicidad, transacciones), WireMock (M1: éxito, error, timeout), ArchUnit (regla de dependencias entre capas).
**Target Platform**: Servidor Linux con JVM 21; navegador web para el frontend.
**Project Type**: Web: backend y frontend separados (React + Vite).
**Performance Goals**: registrar el reporte en **menos de 2 segundos** desde que se recibe la información (SC-004). T1 es local y corta; la llamada a M1 tiene un *timeout* de 1,5 s para no exceder el presupuesto.
**Constraints**:
- Un solo reporte de daño por utilización, garantizado por clave única en BD (FR-007, SC-003).
- Descripción obligatoria (FR-008); la evidencia es opcional (FR-004).
- Todo reporte registrado con éxito dispara UC4 (FR-005, SC-001): por eso UC4 se llama dentro de T1.
- El cambio de estado del recurso exige **confirmación síncrona** de M1; no se asume que cambió si M1 no respondió con éxito (`plan.md`).
- Si el reporte no puede registrarse, no debe quedar un registro incompleto: T1 es atómica.
- La zona horaria es `America/Bogota`.
**Scale/Scope**: Mismo orden que M2 (miles de estudiantes y recursos). Una sección de daño dentro de la pantalla de check-out, una pantalla de reporte para dirección y un panel de revisión de inconsistencias.

## Integración con otros módulos

### Lo que M3 consume de M1 y M2 (adoptado tal cual)

**UC8 no consume eventos de M1 ni de M2.** Su disparador es una persona (estudiante en el check-out, o dirección) y, internamente, UC6.

**Contrato REST de M1:** la spec y `plan.md` fijan *que* M3 debe llamar a M1 de forma síncrona para cambiar el estado, pero **el endpoint concreto de M1 no está en los documentos disponibles**. La sección Contratos §4 propone una forma mínima, marcada como **NC-01**, para contrastarla con el plan de M1 antes de implementar. No se redefine nada de M1: se adapta M3 a lo que M1 publique.

### Lo que M3 produce para M1 y M2 (creado por M3)

**UC8 no produce eventos Kafka.** El cobro generado y su notificación al estudiante son responsabilidad de UC4 (que decide si publica a M2). El único efecto hacia otro módulo es la llamada REST a M1.

### Endpoints que M3 expone para M2 y M1

**UC8 no expone endpoints a M1 ni a M2.**

### Endpoints que M3 expone para su propio frontend

| Endpoint | Método | Para qué | Rol |
|---|---|---|---|
| *(dentro de `POST` de check-out de UC6)* | POST | El estudiante indica el daño al hacer check-out; UC6 llama a `reportDuringCheckOut` | `ESTUDIANTE` o `MONITOR` |
| `POST /api/v1/admin/damage-reports` | POST | Dirección reporta un daño de forma independiente | `ADMIN` o `DIRECCION_PROGRAMA` |
| `GET /api/v1/damage-reports/{id}` | GET | Consultar un reporte (el estudiante solo si es el titular de la utilización) | `ESTUDIANTE`, `MONITOR`, `ADMIN` o `DIRECCION_PROGRAMA` |
| `GET /api/v1/admin/damage-reports` | GET | Listar reportes; filtrar por `needsReview` o `resourceId` | `ADMIN` o `DIRECCION_PROGRAMA` |
| `POST /api/v1/damage-reports/{id}/evidence` | POST (multipart) | Adjuntar fotografías a un reporte | quien reportó, o `ADMIN` |
| `GET /api/v1/damage-reports/{id}/evidence/{evidenceId}` | GET | Descargar una evidencia | `ADMIN`, `DIRECCION_PROGRAMA` o el titular |
| `POST /api/v1/admin/damage-reports/{id}/retry-resource-sync` | POST | Reintentar el cambio de estado en M1 | `ADMIN` |

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan.md                                    # Plan general del M3
├── reportar-novedad-tecnica.md                 # Spec de este plan
├── realizar-checkout.md                       # «extend» que lo invoca
├── generar-cobro.md                            # «include» que dispara
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
│   │   │   └── AdminDamageReportController.java       # POST, GET list, retry-resource-sync
│   │   └── dto/
│   │       ├── DamageReportRequest.java
│   │       ├── DamageReportResponse.java
│   │       ├── DamageReportListResponse.java
│   │       └── DamageEvidenceResponse.java
│   │
│   ├── business/
│   │   ├── service/
│   │   │   ├── DamageReportService.java               # implementa ReportDamagePort
│   │   │   ├── DamageReportRegistrar.java             # @Transactional: validar + guardar + UC4 (T1)
│   │   │   ├── ResourceStatusSynchronizer.java        # llamada a M1 + marcado (T2)
│   │   │   └── DamageEvidenceService.java             # implementa AttachDamageEvidencePort
│   │   └── event/
│   │       └── ResourceSyncListener.java              # @TransactionalEventListener(AFTER_COMMIT)
│   │
│   ├── domain/
│   │   ├── model/
│   │   │   ├── DamageReport.java                      # recurso, utilización, descripción, estado de sincronización
│   │   │   ├── DamageEvidence.java                    # metadatos de una fotografía
│   │   │   ├── DamageSource.java                      # enum: CHECK_OUT, INDEPENDENT
│   │   │   ├── ResourceTargetStatus.java              # enum: MAINTENANCE, OUT_OF_SERVICE
│   │   │   ├── ResourceSyncStatus.java                # enum: PENDING, SYNCED, FAILED
│   │   │   └── UsageView.java                         # vista mínima de la utilización (de UC6)
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   ├── ReportDamagePort.java              # puerto de entrada
│   │   │   │   └── AttachDamageEvidencePort.java
│   │   │   └── out/
│   │   │       ├── DamageReportRepository.java
│   │   │       ├── DamageEvidenceRepository.java
│   │   │       ├── UsageLookupPort.java               # utilización por id / última por recurso (de UC6)
│   │   │       ├── ResourceModuleClient.java          # REST síncrono a M1
│   │   │       ├── GenerateChargePort.java            # salida hacia UC4
│   │   │       └── EvidenceStoragePort.java           # guardar/leer archivos
│   │   ├── event/
│   │   │   └── DamageReportRegisteredEvent.java       # evento de dominio interno (dispara la sincronización)
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
│       │       └── rest/
│       │           ├── ResourceModuleRestClient.java  # implementa ResourceModuleClient (timeout 1,5 s)
│       │           └── dto/
│       │               └── ResourceStatusUpdateRequest.java
│       ├── storage/
│       │   └── LocalEvidenceStorage.java              # implementa EvidenceStoragePort sobre un volumen
│       ├── scheduler/
│       │   └── ResourceSyncRetryScheduler.java        # @Scheduled: reintenta los FAILED
│       └── config/
│           ├── SecurityConfig.java
│           ├── RestClientConfig.java                  # timeouts de M1
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
    │   └── ResourceModuleClientContractTest.java       # WireMock contra el contrato de M1
    ├── integration/
    │   ├── DamageReportIT.java                         # persistencia, unicidad, T1 atómica con UC4
    │   ├── DamageResourceSyncIT.java                   # AFTER_COMMIT, M1 éxito/fallo/timeout, reintento
    │   └── DamageEvidenceIT.java
    └── unit/
        ├── DamageReportServiceTest.java
        ├── DamageReportRegistrarTest.java
        └── ResourceStatusSynchronizerTest.java

frontend/
└── src/
    ├── pages/
    │   └── DamageReportPage.jsx                        # reporte independiente + consulta + revisión (dirección)
    ├── components/
    │   ├── DamageReportForm.jsx                        # descripción + evidencia; reusado dentro de CheckOutPage
    │   ├── DamageEvidenceUploader.jsx                  # selector de fotos con vista previa
    │   ├── ResourceSyncBadge.jsx                       # PENDING / SYNCED / FAILED
    │   └── DamageReviewPanel.jsx                       # reportes con needsReview + botón reintentar
    └── services/
        └── damageReportApi.js                          # cliente HTTP
```

**Structure Decision**: se respeta la estructura de capas de `plan.md` (presentation / business / domain / infrastructure) con el paquete raíz `com.university.sanctions`. Siguiendo los planes los demás UC, los contratos viven como puertos en `domain/port/in` y `domain/port/out`; `ResourceModuleClient` (que `plan.md` ubica en `business/integration`) se declara aquí como puerto de salida para ser consistente con los demás planes, y su implementación REST vive en `infrastructure/integration/module1/rest`, como indica `plan.md`. La llamada a M1 se separa de la transacción de registro mediante un evento de dominio interno (`DamageReportRegisteredEvent`) escuchado después del commit. El frontend reutiliza `DamageReportForm.jsx` dentro de la pantalla de check-out de UC6.

## Decisiones de diseño de este caso de uso

**Dónde queda cada FR.**

| FR | Qué pide | Dónde se implementa |
|---|---|---|
| FR-001 | Reportar daño como parte del check-out (`«extend»`) | `ReportDamagePort.reportDuringCheckOut(...)` — lo llama UC6 |
| FR-002 | Reporte independiente por dirección universitaria | `AdminDamageReportController.POST` → `ReportDamagePort.reportIndependent(...)` |
| FR-003 | Identificar recurso, utilización y estudiante | `UsageLookupPort` + `DamageReportRegistrar` (regla de vinculación, ver decisión 3) |
| FR-004 | Registrar descripción y, si existe, evidencia | Descripción en T1; evidencia con `AttachDamageEvidencePort` (`DamageEvidenceService`) |
| FR-005 | Disparar obligatoriamente Generar cobro | `DamageReportRegistrar` llama a `GenerateChargePort.generate(...)` dentro de T1 |
| FR-006 | Actualizar el estado del recurso en M1 | `ResourceSyncListener` → `ResourceStatusSynchronizer` → `ResourceModuleClient.changeStatus(...)` |
| FR-007 | Impedir reporte duplicado por utilización | Clave única en `damage_report.usage_id` + verificación previa |
| FR-008 | Informar recurso no identificado o descripción faltante | `ResourceNotIdentifiedException`, `MissingDescriptionException` → `ProblemDetail` |

**Decisiones justificadas.**

1. **El cobro se dispara dentro de la transacción de registro; el cambio de estado en M1, después.** SC-001 exige que el 100 % de los reportes registrados disparen el cobro, y "evitar dejar un registro incompleto" exige atomicidad. Meter `GenerateChargePort` en T1 da ambas cosas: o hay reporte *y* cobro, o ninguno. En cambio M1 es REST y no puede participar de la transacción MySQL; por eso va después del commit y su fallo **no** revierte el reporte (Edge Case "Error al cambiar el estado del recurso").

2. **La llamada a M1 va en un listener `AFTER_COMMIT`, no dentro de `@Transactional`.** Mantener una transacción de BD abierta mientras se espera una respuesta HTTP bloquea conexiones y, si M1 se demora, alarga el bloqueo de filas. El listener se ejecuta de forma síncrona en el mismo hilo (no `@Async`), así la respuesta HTTP incluye el resultado real de M1 (`resourceSyncStatus`), que es lo que `plan.md` pide: confirmar antes de considerar actualizado el recurso.

3. **Regla de vinculación a una utilización.** Es el punto más delicado de la spec:
   - *Durante el check-out*: la utilización es la del check-out en curso; UC6 pasa `usageId`.
   - *Independiente, con `usageId` explícito*: se valida que la utilización exista y que corresponda al `resourceId` indicado.
   - *Independiente, sin `usageId`*: se usa la **última utilización cerrada** del recurso, siempre que el recurso **no esté actualmente en uso por otra persona**.
   - *Recurso actualmente en uso (posible reasignación)*: no se puede saber con certeza si el daño ocurrió en la utilización anterior o en la actual. La spec permite "vincular a la última utilización identificable antes de la reasignación, o rechazarlo si no puede determinarse con certeza". Se elige la opción **conservadora**: rechazar con `USAGE_NOT_DETERMINABLE` salvo que dirección indique `usageId`. Un cobro a la persona equivocada es peor que pedir un dato más.
   - *Recurso sin ninguna utilización*: se rechaza con `USAGE_NOT_DETERMINABLE` (sin estudiante no hay a quién cobrar; NC-03).

4. **`usage_id` obligatorio y `UNIQUE`.** Aunque la spec dice que la utilización y el estudiante se identifican "cuando aplique" (FR-003), el cobro (FR-005) siempre necesita un estudiante. Con `usage_id NOT NULL UNIQUE`, FR-007 y SC-003 se garantizan en la base de datos incluso con dos reportes simultáneos. La verificación previa solo da el mensaje claro; la clave única es la garantía real.

5. **Un daño durante el check-out aborta el check-out si es inválido.** Si el estudiante marca "tiene daño" pero no escribe la descripción, `MissingDescriptionException` se propaga a UC6 y el check-out no se cierra a medias, para que el estudiante pueda corregir (NC-02 para confirmarlo con UC6).

6. **Estado destino configurable, no inventado por el estudiante.** La spec dice "En mantenimiento" o "Fuera de servicio". El estudiante no tiene criterio técnico para elegir, así que en el check-out se usa el valor por defecto `sanctions.damage.default-resource-status = MAINTENANCE`. Dirección puede indicar `OUT_OF_SERVICE` en el reporte independiente.

7. **Idempotencia de la llamada a M1 con `Idempotency-Key = reportId`.** El reintento (manual o por job) puede llegar a M1 más de una vez; con la misma clave, M1 debe tratar la repetición como un no-op. Cambiar a `MAINTENANCE` un recurso que ya está en `MAINTENANCE` es seguro de todos modos (NC-01).

8. **Reintento automático además del manual.** Mientras M1 no confirme, el recurso puede seguir apareciendo como disponible y ser asignado a otro estudiante — justo lo que SC-002 busca evitar. Por eso, a diferencia de UC7, aquí sí hay un job (`ResourceSyncRetryScheduler`, cada 1 minuto, máximo `sanctions.damage.sync.max-attempts = 10`) que reenvía los `FAILED`. Tras agotar los intentos queda `needs_review = true` para intervención humana. Sigue existiendo el endpoint manual.

9. **La evidencia se sube aparte del reporte.** Enviar archivos dentro del flujo de check-out de UC6 obligaría a convertir ese endpoint en `multipart` y alargaría el camino crítico. Con `POST /damage-reports/{id}/evidence`, el reporte se registra rápido (SC-004) y las fotos llegan después. Límites propuestos: máximo 5 archivos por reporte, 5 MB cada uno, solo `image/jpeg` y `image/png` (NC-04).

10. **UC8 no bloquea al estudiante (por ahora).** El plan de UC3 dice que UC8 llama a `BlockForDamagePort.block` "cuando hay daño grave", pero la spec de UC8 no define "daño grave", ni gravedad, ni bloqueo. Para no inventar reglas, **no se implementa** esa llamada y se registra como **NC-05**. Cuando se decida, es una línea más en `DamageReportRegistrar` (y un campo `severity` en el reporte).

11. **UC8 no publica a Kafka.** Ni `plan.md` ni la spec definen una notificación del reporte de daño; la integración declarada es solo REST con M1. Publicar eventos que nadie consume es ruido. El cobro se notifica desde UC4.

12. **El reporte no se elimina ni se edita.** Es evidencia para un cobro. Solo cambian los campos de seguimiento de la sincronización y se pueden añadir evidencias.

## Contratos

Se aplican las convenciones comunes de M2 (camelCase, MAYUSCULA_CON_GUION_BAJO, nunca null, ISO-8601 con `-05:00`, RFC 9457). M3 las adopta para sus propios contratos.

### 1. Puerto de entrada — `ReportDamagePort`

```java
public interface ReportDamagePort {

    /**
     * Lo llama Realizar check-out (UC6) cuando el estudiante indica un daño.
     * Se ejecuta dentro de la transacción del check-out.
     * @throws MissingDescriptionException      falta la descripción del daño
     * @throws ResourceNotIdentifiedException   el recurso no pudo identificarse
     * @throws DuplicateDamageReportException   ya existe un reporte para la utilización
     */
    DamageReportResult reportDuringCheckOut(ReportDamageDuringCheckOutCommand command);

    /**
     * Lo llama el controlador de dirección universitaria.
     * @throws UsageNotDeterminableException    no se puede vincular con certeza a una utilización
     * (más las excepciones anteriores)
     */
    DamageReportResult reportIndependent(ReportDamageIndependentCommand command);

    /** Reenvía el cambio de estado del recurso a M1 para un reporte en FAILED. */
    DamageReportResult retryResourceSync(String reportId, String adminCode);
}

public record ReportDamageDuringCheckOutCommand(
    String usageId,
    String description,
    String reporterCode          // el estudiante, del JWT
) {}

public record ReportDamageIndependentCommand(
    String resourceId,
    String usageId,              // opcional
    String description,
    ResourceTargetStatus targetStatus, // opcional; por defecto MAINTENANCE
    String reporterCode          // dirección, del JWT
) {}

public record DamageReportResult(
    String reportId,
    String resourceId,
    String usageId,
    String studentCode,
    String chargeId,
    ResourceSyncStatus resourceSyncStatus,
    boolean needsReview,
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

`GenerateChargePort` — lo implementa **UC4**; UC8 lo llama dentro de T1:

```java
public interface GenerateChargePort {
    ChargeRef generate(GenerateChargeCommand command);
}

public record GenerateChargeCommand(
    String studentCode,
    String resourceId,
    String usageId,
    String reason,            // "DAMAGE"
    String sourceEventId      // = damageReportId, para idempotencia en UC4
) {}
```

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

`ResourceModuleClient` — REST síncrono a M1:

```java
public interface ResourceModuleClient {
    /** Devuelve normalmente si M1 confirmó; lanza ResourceModuleException si falló o expiró. */
    void changeStatus(String resourceId, ResourceTargetStatus status, String idempotencyKey, String reason);
}
```

### 4. Contrato REST con M1 (propuesta — **NC-01**)

Debe contrastarse con el plan de M1; si M1 publica otra forma, solo cambia `ResourceModuleRestClient`.

```http
PUT /api/v1/resources/{resourceId}/status
Idempotency-Key: 7d1c3e52-9a04-4f67-b1c8-3e5a2d9f6b10
Authorization: Bearer <token de servicio de M3>
```

```json
{
  "status": "MAINTENANCE",
  "reason": "Daño reportado en M3",
  "sourceReportId": "7d1c3e52-9a04-4f67-b1c8-3e5a2d9f6b10"
}
```

Respuesta esperada: `200 OK` (o `204`) = confirmado. Cualquier `4xx/5xx`, error de conexión o *timeout* (1,5 s) = **no confirmado** → `FAILED`. Un `404` de M1 (recurso inexistente) se registra con `failure_reason = RESOURCE_NOT_FOUND_IN_M1` y pasa directo a `needs_review` sin reintentos automáticos.

### 5. Endpoints REST

#### `POST /api/v1/admin/damage-reports`

Rol `ADMIN` o `DIRECCION_PROGRAMA`. El reportante sale del JWT.

```json
{
  "resourceId": "ACT-004512",
  "usageId": "usg-20260930-0187",
  "description": "La pantalla del portátil presenta una fisura en la esquina inferior derecha.",
  "targetStatus": "MAINTENANCE"
}
```

`usageId` y `targetStatus` son opcionales.

Respuesta `201 Created` (M1 confirmó):

```json
{
  "id": "5a8f1c2e-3b74-4d09-9e16-7c2b0a4d8f35",
  "resourceId": "ACT-004512",
  "usageId": "usg-20260930-0187",
  "studentCode": "2023123456",
  "source": "INDEPENDENT",
  "description": "La pantalla del portátil presenta una fisura en la esquina inferior derecha.",
  "targetStatus": "MAINTENANCE",
  "reportedBy": "direccion@unimagdalena.edu.co",
  "reportedAt": "2026-10-05T11:15:00-05:00",
  "chargeId": "chg-10231",
  "resourceSyncStatus": "SYNCED",
  "needsReview": false,
  "evidence": [],
  "message": "Daño registrado. El recurso pasó a mantenimiento y se generó el cobro correspondiente."
}
```

Respuesta `201 Created` (reporte registrado, M1 no confirmó — Edge Case de la spec):

```json
{
  "id": "5a8f1c2e-3b74-4d09-9e16-7c2b0a4d8f35",
  "resourceId": "ACT-004512",
  "usageId": "usg-20260930-0187",
  "studentCode": "2023123456",
  "source": "INDEPENDENT",
  "targetStatus": "MAINTENANCE",
  "reportedAt": "2026-10-05T11:15:00-05:00",
  "chargeId": "chg-10231",
  "resourceSyncStatus": "FAILED",
  "needsReview": true,
  "evidence": [],
  "message": "El daño quedó registrado, pero el estado del recurso no pudo actualizarse. Quedó marcado para revisión."
}
```

> Un fallo de M1 **no es un error HTTP**: el reporte y el cobro sí existen, por eso `201` con `resourceSyncStatus: FAILED`.

#### Reporte durante el check-out

No tiene endpoint propio: el cuerpo del check-out de UC6 incluye un bloque opcional que UC6 delega a `reportDuringCheckOut(...)`. Este plan define la forma de ese bloque; UC6 debe incluirla en su contrato:

```json
{
  "damage": {
    "hasDamage": true,
    "description": "La tecla Enter quedó suelta."
  }
}
```

Y la respuesta del check-out incorpora `damageReport` con `id`, `chargeId` y `resourceSyncStatus`.

#### `GET /api/v1/damage-reports/{id}`

Devuelve el mismo objeto del `201`. Un estudiante solo puede ver el reporte de su propia utilización (`403` si no).

#### `GET /api/v1/admin/damage-reports?needsReview=true&resourceId=ACT-004512&page=1&pageSize=20`

```json
{
  "reports": [ { "id": "5a8f1c2e-...", "resourceId": "ACT-004512", "studentCode": "2023123456", "resourceSyncStatus": "FAILED", "needsReview": true, "reportedAt": "2026-10-05T11:15:00-05:00" } ],
  "pagination": { "page": 1, "pageSize": 20, "totalPages": 1, "total": 1 }
}
```

#### `POST /api/v1/damage-reports/{id}/evidence`

`multipart/form-data` con un campo `file`. Respuesta `201 Created`:

```json
{
  "id": "evd-0001",
  "reportId": "5a8f1c2e-3b74-4d09-9e16-7c2b0a4d8f35",
  "fileName": "pantalla.jpg",
  "contentType": "image/jpeg",
  "sizeBytes": 1843201,
  "uploadedAt": "2026-10-05T11:17:00-05:00"
}
```

#### `POST /api/v1/admin/damage-reports/{id}/retry-resource-sync`

Rol `ADMIN`. Sin cuerpo. `200 OK` con el reporte actualizado (`SYNCED` o, si M1 vuelve a fallar, `FAILED`). `409 RESOURCE_ALREADY_SYNCED` si ya estaba sincronizado.

### 6. Errores

| Código HTTP | `code` | Cuándo |
|---|---|---|
| 400 | `MISSING_DESCRIPTION` | Falta la descripción del daño (FR-008) |
| 400 | `INVALID_EVIDENCE` | Tipo no permitido, archivo > 5 MB o más de 5 archivos |
| 404 | `RESOURCE_NOT_IDENTIFIED` | No fue posible identificar el recurso asociado (FR-008) |
| 404 | `DAMAGE_REPORT_NOT_FOUND` | El reporte consultado no existe |
| 409 | `DUPLICATE_DAMAGE_REPORT` | Ya existe un reporte para la utilización (FR-007) |
| 409 | `USAGE_NOT_DETERMINABLE` | No se puede vincular con certeza a una utilización (recurso reasignado o sin historial) |
| 409 | `RESOURCE_ALREADY_SYNCED` | Reintento sobre un reporte ya sincronizado |
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

### 7. Tablas

#### `damage_report`

| Columna | Tipo | Nota |
|---|---|---|
| `id` | varchar(36) PK | UUID |
| `resource_id` | varchar(50) NOT NULL | |
| `usage_id` | varchar(50) NOT NULL | **UNIQUE** (FR-007) |
| `student_code` | varchar(20) NOT NULL | Titular de la utilización |
| `source` | varchar(20) NOT NULL | `CHECK_OUT`, `INDEPENDENT` |
| `description` | text NOT NULL | |
| `target_status` | varchar(20) NOT NULL | `MAINTENANCE`, `OUT_OF_SERVICE` |
| `reported_by` | varchar(100) NOT NULL | Del JWT |
| `reported_at` | timestamp NOT NULL | |
| `charge_id` | varchar(50), nulo | Id del cobro devuelto por UC4 |
| `resource_sync_status` | varchar(20) NOT NULL | `PENDING`, `SYNCED`, `FAILED` |
| `sync_attempts` | int NOT NULL DEFAULT 0 | |
| `last_sync_error` | varchar(255), nulo | |
| `last_sync_attempt_at` | timestamp, nulo | |
| `synced_at` | timestamp, nulo | |
| `needs_review` | boolean NOT NULL DEFAULT false | |
| `created_at` | timestamp NOT NULL | |
| `updated_at` | timestamp NOT NULL | |

```sql
CREATE UNIQUE INDEX damage_report_usage ON damage_report (usage_id);
CREATE INDEX damage_report_resource ON damage_report (resource_id, reported_at DESC);
CREATE INDEX damage_report_student ON damage_report (student_code, reported_at DESC);
CREATE INDEX damage_report_sync ON damage_report (resource_sync_status, last_sync_attempt_at);
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

La clave única en `usage_id` es lo que hace cumplir FR-007 y SC-003 a nivel de base de datos.

### 8. Tipos del frontend

```js
export const ResourceSyncStatus = {
  PENDING: "PENDING",
  SYNCED: "SYNCED",
  FAILED: "FAILED",
};

export const ResourceTargetStatus = {
  MAINTENANCE: "MAINTENANCE",
  OUT_OF_SERVICE: "OUT_OF_SERVICE",
};

export const reportDamage = async ({ resourceId, usageId, description, targetStatus }) => {
  const response = await fetch("/api/v1/admin/damage-reports", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ resourceId, usageId, description, targetStatus }),
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

export const getDamageReports = async ({ needsReview, resourceId, page = 1 } = {}) => {
  const params = new URLSearchParams({ page });
  if (needsReview !== undefined) params.set("needsReview", needsReview);
  if (resourceId) params.set("resourceId", resourceId);
  const response = await fetch(`/api/v1/admin/damage-reports?${params}`);
  if (!response.ok) throw await response.json();
  return response.json();
};

export const retryResourceSync = async (reportId) => {
  const response = await fetch(`/api/v1/admin/damage-reports/${reportId}/retry-resource-sync`, {
    method: "POST",
  });
  if (!response.ok) throw await response.json();
  return response.json();
};
```

### 9. Fixtures compartidos

```text
backend/src/test/resources/contracts/
├── api-damage-report-201-synced.json
├── api-damage-report-201-sync-failed.json
├── api-damage-reports-list-200.json
├── api-damage-evidence-201.json
├── api-damage-report-400-missing-description.json
├── api-damage-report-404-resource.json
├── api-damage-report-409-duplicate.json
├── api-damage-report-409-usage-not-determinable.json
├── m1-resource-status-200.json
└── m1-resource-status-500.json
```

## Phase 1: Setup

- [ ] T001 Añadir a `application.yml`: `sanctions.damage.default-resource-status=MAINTENANCE`, `sanctions.damage.sync.max-attempts=10`, `sanctions.damage.sync.retry-interval=PT1M`, `sanctions.damage.evidence.max-files=5`, `sanctions.damage.evidence.max-size-bytes=5242880`, `sanctions.damage.evidence.storage-path=/data/evidence`, `sanctions.integration.module1.base-url`, `sanctions.integration.module1.timeout=PT1.5S`
- [ ] T002 [P] Extender `SanctionsProperties` con esos parámetros y validarlos al arrancar
- [ ] T003 [P] Añadir un volumen `evidence` al `docker-compose.yml` y configurar `RestClientConfig` con los timeouts de M1
- [ ] T004 [P] Habilitar `@Scheduled` en `SchedulingConfig` (si no existe ya de UC1)

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: dónde guardar el reporte y la evidencia, y los puertos hacia UC6, UC4 y M1.

- [ ] T005 Escribir `V12__damage_report.sql` con la tabla `damage_report` y sus índices de Contratos §7 (incluida la clave única en `usage_id`)
- [ ] T006 Escribir `V13__damage_evidence.sql` con la tabla `damage_evidence` y su índice
- [ ] T007 [P] Crear el modelo de dominio: `DamageReport`, `DamageEvidence`, `DamageSource`, `ResourceTargetStatus`, `ResourceSyncStatus`, `UsageView` en `domain/model/`
- [ ] T008 [P] Definir `ReportDamagePort` y `AttachDamageEvidencePort` en `domain/port/in/`
- [ ] T009 [P] Definir `DamageReportRepository`, `DamageEvidenceRepository`, `UsageLookupPort`, `ResourceModuleClient`, `GenerateChargePort` y `EvidenceStoragePort` en `domain/port/out/`
- [ ] T010 [P] Crear `DamageReportRegisteredEvent` en `domain/event/` y las excepciones de `domain/error/`
- [ ] T011 [P] Implementar `DamageReportEntity`, `DamageEvidenceEntity`, sus repositorios Spring Data, adaptadores y mappers
- [ ] T012 [P] Implementar el adaptador de `UsageLookupPort` sobre la tabla `usage` de UC6 (**verificar columnas con el plan de UC6**)
- [ ] T013 [P] Implementar `ResourceModuleRestClient` (REST a M1, `Idempotency-Key`, timeout, mapeo de errores a `ResourceModuleException`) — **depende de NC-01**
- [ ] T014 [P] Implementar `LocalEvidenceStorage` sobre el volumen
- [ ] T015 Coordinar con UC4 la firma de `GenerateChargePort`; mientras UC4 no exista, usar un doble de prueba que devuelva un `chargeId` simulado

**Checkpoint**: existe dónde guardar el reporte, cómo ubicar la utilización, cómo hablar con M1 y cómo disparar el cobro

## Phase 3: User Story 1 — Reporte de daño durante el check-out (Priority: P1)

**Goal**: cuando el estudiante indica un daño al hacer check-out, el sistema registra el reporte vinculado a la utilización, dispara el cobro y cambia el estado del recurso en M1.

**Independent Test**: simular un check-out con `damage.hasDamage = true` y descripción; comprobar que (a) hay una fila en `damage_report` con `source = CHECK_OUT`, (b) `GenerateChargePort` se invocó una vez, (c) M1 recibió el cambio a `MAINTENANCE` y el reporte quedó `SYNCED`.

### Tests for User Story 1

- [ ] T016 [P] [US1] Pruebas en `DamageReportServiceTest.java` (unit, con puertos falsos): `reportDuringCheckOut` registra con el estudiante de la utilización, llama a `GenerateChargePort` con `sourceEventId = reportId` y publica `DamageReportRegisteredEvent`
- [ ] T017 [P] [US1] Prueba `DamageReportIT.java` con Testcontainers MySQL: reporte + cobro en la misma transacción; si `GenerateChargePort` lanza excepción → rollback completo y **ninguna** fila en `damage_report`
- [ ] T018 [P] [US1] Prueba `DamageResourceSyncIT.java` con WireMock: M1 responde 200 → `SYNCED` y `synced_at`; el listener `AFTER_COMMIT` solo se ejecuta si T1 hizo commit (si UC6 revierte el check-out, M1 **no** es llamado)
- [ ] T019 [P] [US1] Prueba de contrato `ResourceModuleClientContractTest.java`: cabecera `Idempotency-Key`, cuerpo y manejo de 200/204 según el contrato acordado con M1

### Implementation for User Story 1

- [ ] T020 [US1] Implementar `DamageReportRegistrar.register(...)` (`@Transactional`): validar, guardar con `PENDING`, llamar a `GenerateChargePort`, guardar `charge_id` y publicar `DamageReportRegisteredEvent` (T1)
- [ ] T021 [US1] Implementar `ResourceStatusSynchronizer.sync(...)`: llamar a `ResourceModuleClient.changeStatus(...)` y marcar `SYNCED` (T2)
- [ ] T022 [US1] Implementar `ResourceSyncListener` con `@TransactionalEventListener(phase = AFTER_COMMIT)` (síncrono, sin `@Async`)
- [ ] T023 [US1] Implementar `DamageReportService.reportDuringCheckOut(...)`
- [ ] T024 [US1] Coordinar con UC6: añadir el bloque `damage` al contrato del check-out y llamar a `reportDuringCheckOut` desde `CheckOutService`; incluir `damageReport` en la respuesta del check-out

**Checkpoint**: el camino P1 funciona de punta a punta, con cobro y cambio de estado del recurso

## Phase 4: User Story 2 — Reporte de daño fuera del flujo de check-out (Priority: P2)

**Goal**: dirección universitaria reporta un daño sobre un recurso ya devuelto, y el sistema lo vincula a la última utilización identificable.

**Independent Test**: sobre un recurso devuelto sin daño reportado, `POST /api/v1/admin/damage-reports` con `resourceId` y descripción; comprobar `source = INDEPENDENT`, vinculación a la última utilización cerrada, cobro disparado y estado actualizado en M1.

### Tests for User Story 2

- [ ] T025 [P] [US2] Pruebas en `DamageReportRegistrarTest.java`: vinculación a la última utilización cerrada; `usageId` explícito que no pertenece al recurso → `RESOURCE_NOT_IDENTIFIED`/`USAGE_NOT_DETERMINABLE`; recurso **en uso por otra persona** sin `usageId` → `USAGE_NOT_DETERMINABLE`; recurso en uso **con** `usageId` explícito → acepta; recurso sin historial → `USAGE_NOT_DETERMINABLE`
- [ ] T026 [P] [US2] Prueba `AdminDamageReportControllerTest.java` con `@WebMvcTest`, contra los fixtures: `POST` con rol `DIRECCION_PROGRAMA` → 201; con rol `ESTUDIANTE` → 403; sin sesión → 401
- [ ] T027 [P] [US2] Prueba `DamageReportIT.java` (extensión): `targetStatus = OUT_OF_SERVICE` se envía a M1; por defecto se envía `MAINTENANCE`

### Implementation for User Story 2

- [ ] T028 [US2] Completar `DamageReportRegistrar` con la regla de vinculación de la decisión 3 (usar `UsageLookupPort.findLastClosedByResource` / `findActiveByResource`)
- [ ] T029 [US2] Implementar `DamageReportService.reportIndependent(...)`
- [ ] T030 [US2] Implementar `AdminDamageReportController` con `POST /api/v1/admin/damage-reports` y los DTOs `DamageReportRequest`/`DamageReportResponse`; tomar el reportante del JWT

**Checkpoint**: dirección puede reportar daños detectados fuera del check-out

## Phase 5: User Story 3 — Rechazos y falla de M1 (Priority: P1)

**Goal**: el sistema rechaza los reportes inválidos con el motivo exacto, impide duplicados y, si M1 no confirma, conserva el reporte, marca la inconsistencia y lo reintenta.

**Independent Test**: cubrir cada Edge Case de la spec — recurso no identificado, duplicado, sin descripción, recurso reasignado, error al registrar, error de M1 — y verificar código, `code` y estado final de los datos.

### Tests for User Story 3

- [ ] T031 [P] [US3] Pruebas en `DamageReportRegistrarTest.java`: descripción vacía o solo espacios → `MISSING_DESCRIPTION`; recurso inexistente → `RESOURCE_NOT_IDENTIFIED`; duplicado → `DUPLICATE_DAMAGE_REPORT`; un rechazo **no** invoca a `GenerateChargePort` ni a M1
- [ ] T032 [P] [US3] Prueba `DamageReportIT.java` (extensión): dos reportes concurrentes sobre la misma utilización → una fila, un solo `201`, el otro `409` (clave única); un rechazo no deja filas en `damage_report`
- [ ] T033 [P] [US3] Pruebas en `ResourceStatusSynchronizerTest.java` y `DamageResourceSyncIT.java` (WireMock): M1 responde 500 / expira → `FAILED`, `needs_review = true`, `sync_attempts` incrementado, **reporte y cobro intactos**; M1 responde 404 → `needs_review` sin reintentos automáticos
- [ ] T034 [P] [US3] Prueba del `ResourceSyncRetryScheduler`: reintenta solo los `FAILED` con intentos < máximo; tras el éxito pasa a `SYNCED` y `needs_review = false`; al agotar intentos queda en `needs_review = true`; con reloj inyectado
- [ ] T035 [P] [US3] Pruebas de `retry-resource-sync` en `AdminDamageReportControllerTest.java`: `ADMIN` → 200; ya sincronizado → 409; rol insuficiente → 403

### Implementation for User Story 3

- [ ] T036 [US3] Completar `DamageReportRegistrar` con las validaciones: recurso identificable → descripción no vacía → utilización determinable → no duplicado
- [ ] T037 [US3] Traducir la violación de la clave única `usage_id` a `DuplicateDamageReportException` en `DamageReportRepositoryAdapter`
- [ ] T038 [US3] Completar `ResourceStatusSynchronizer` con la rama de fallo (`FAILED`, `last_sync_error`, `needs_review`) sin lanzar excepción hacia el controlador
- [ ] T039 [US3] Implementar `ResourceSyncRetryScheduler` y `DamageReportService.retryResourceSync(...)` con el endpoint `POST .../retry-resource-sync`
- [ ] T040 [US3] Mapear cada excepción de dominio a su `ProblemDetail` (`@RestControllerAdvice`) según Contratos §6

**Checkpoint**: ningún reporte inválido se registra y una falla de M1 no pierde datos ni se queda sin atender

## Phase 6: User Story 4 — Evidencia, consulta y pantallas (Priority: P3)

**Goal**: adjuntar fotografías, consultar reportes y operar todo desde la interfaz.

### Tests for User Story 4

- [ ] T041 [P] [US4] Prueba `DamageEvidenceIT.java`: subir una imagen válida la guarda y registra sus metadatos; tipo no permitido, > 5 MB o el sexto archivo → `INVALID_EVIDENCE`; no se puede adjuntar a un reporte inexistente (`404`)
- [ ] T042 [P] [US4] Prueba `DamageReportControllerTest.java`: `GET /{id}` por el titular → 200, por otro estudiante → 403; descarga de evidencia con autorización correcta
- [ ] T043 [P] [US4] Pruebas de `DamageReportForm` y `DamageReviewPanel` (React Testing Library): la descripción vacía muestra el error del backend; "Reintentar" solo aparece con `FAILED`; la evidencia es opcional

### Implementation for User Story 4

- [ ] T044 [US4] Implementar `DamageEvidenceService` y los endpoints de subida y descarga en `DamageReportController`
- [ ] T045 [US4] Implementar `GET /api/v1/damage-reports/{id}` y `GET /api/v1/admin/damage-reports` con paginación y reglas de visibilidad por rol
- [ ] T046 [P] [US4] Frontend: `damageReportApi.js`
- [ ] T047 [P] [US4] Frontend: `DamageReportForm.jsx` y `DamageEvidenceUploader.jsx` (reutilizables en `CheckOutPage` de UC6)
- [ ] T048 [P] [US4] Frontend: `ResourceSyncBadge.jsx` y `DamageReviewPanel.jsx`
- [ ] T049 [US4] Frontend: `DamageReportPage.jsx` que integra el formulario de reporte independiente, la lista y el panel de revisión

**Checkpoint**: el flujo es utilizable de punta a punta desde la interfaz

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T050 [P] Verificar SC-001 (100 % de los reportes registrados disparan Generar cobro)
- [ ] T051 [P] Verificar SC-002 (100 % de los recursos con daño quedan distintos de "Disponible" justo después del reporte, **cuando M1 confirma**; medir cuánto dura la ventana con M1 caído — ver NC-06)
- [ ] T052 [P] Verificar SC-003 (0 % de reportes duplicados por utilización, incluso con concurrencia)
- [ ] T053 [P] Verificar SC-004 (registro < 2 s, incluida la llamada a M1)
- [ ] T054 [P] Registrar en logs cada reporte (recurso, utilización, estudiante, reportante) y cada intento de sincronización con M1 y su resultado
- [ ] T055 [P] Prueba ArchUnit: `domain` y `business` no importan JPA, `RestClient` ni clases de `infrastructure`
- [ ] T056 [P] Documentar en el README cómo reportar un daño, cómo configurar el volumen de evidencia y cómo reintentar la sincronización con M1
- [ ] T057 Llevar a `pendientes-clarificacion.md` los NEEDS CLARIFICATION abiertos en este plan (NC-01 a NC-06)

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: depende de `plan.md`
- **Foundational (Phase 2)**: depende de Setup — BLOCKS las user stories
- **User Story 1 (Phase 3)**: depende de Foundational y de UC6 (T024) y UC4 (T015, al menos con doble de prueba)
- **User Story 2 (Phase 4)**: depende de US1 (comparte `DamageReportRegistrar`)
- **User Story 3 (Phase 5)**: depende de US1 (completa registrar y sincronizar); los tests de T031 pueden escribirse en paralelo con US2
- **User Story 4 (Phase 6)**: depende de US1; la evidencia (T044) es independiente de US2 y US3
- **Polish (Phase 7)**: depende de todas las US

### Dependencias con otros casos de uso del M3

- **Realizar check-out (UC6)**: invoca `reportDuringCheckOut` y aporta `usage` y `UsageLookupPort`. Hay que acordar el bloque `damage` del contrato del check-out (T024) y qué pasa con el check-out si el reporte es inválido (NC-02).
- **Generar cobro (UC4)**: UC8 lo dispara obligatoriamente dentro de T1. Hay que acordar `GenerateChargePort` y su idempotencia por `sourceEventId` (T015).
- **Bloquear usuario (UC3)**: su plan espera que UC8 llame a `BlockForDamagePort.block` en daño grave; **no se implementa** hasta resolver NC-05.

### Dependencias con otros módulos

- **Módulo 1**: REST síncrono para cambiar el estado del recurso (NC-01 sobre el contrato exacto).
- **Módulo 2**: sin integración directa en este UC.

### Parallel Opportunities

- En Foundational: T007 a T014
- En US1: T016 a T019 (tests)
- En US2: T025, T026, T027 (tests)
- En US3: T031 a T035 (tests)
- En US4: T041, T042, T043 (tests) y T046 a T048 (frontend)
- En Polish: T050 a T056

## NEEDS CLARIFICATION abiertos

| Id | Pendiente | Supuesto con el que avanza el plan |
|---|---|---|
| NC-01 | **Contrato REST exacto de M1** para cambiar el estado de un recurso (ruta, método, valores de estado, idempotencia, token de servicio). No figura en los documentos disponibles | `PUT /api/v1/resources/{id}/status` con `Idempotency-Key` (Contratos §4). **Bloquea T013** hasta contrastar con el plan de M1 |
| NC-02 | **Qué hace el check-out si el reporte de daño es inválido** (por ejemplo, falta la descripción) | El check-out completo se rechaza para que el estudiante corrija; no se cierra a medias |
| NC-03 | **Qué pasa con un daño detectado en un recurso sin ninguna utilización** (nadie a quien cobrar) | Se rechaza con `USAGE_NOT_DETERMINABLE` |
| NC-04 | **Dónde y cómo se guardan las fotografías** (volumen local, almacenamiento de objetos), formatos y límites | Volumen de disco detrás de `EvidenceStoragePort`; máx. 5 archivos, 5 MB, JPEG/PNG |
| NC-05 | **Si un daño debe bloquear al estudiante** y cómo se define "daño grave" (el plan de UC3 lo asume; la spec de UC8 no lo menciona) | UC8 no bloquea; se añade `severity` y la llamada a `BlockForDamagePort` cuando se defina |
| NC-06 | **Qué hacer mientras M1 no confirma**: el recurso podría reasignarse antes de que M1 refleje el daño, lo que contradice SC-002 | Reintento automático cada minuto + `needs_review`; SC-002 se cumple "cuando M1 confirma". Si no es aceptable, hay que acordar con M1 un bloqueo preventivo |

## Notes

- La numeración T0XX es propia de este plan
- [P] tasks = different files, no dependencies
- [US1] a [US4] = trazabilidad a la user story
- La sección Contratos es la única fuente del JSON de UC8
- Decisiones adoptadas de M2: convenciones (camelCase, MAYUSCULA_CON_GUION_BAJO, nunca null, ISO-8601 con -05:00, RFC 9457), nombres de tablas en snake_case
- Decisiones propias de M3: el cobro se dispara dentro de la transacción de registro y M1 después del commit; el fallo de M1 no revierte el reporte ni el cobro; `usage_id` obligatorio y único; vinculación conservadora ante reasignación; evidencia subida aparte; reintento automático y manual; UC8 no publica a Kafka ni bloquea al estudiante
- Lo que M3 consume de M1: se adopta tal cual cuando M1 publique su contrato (NC-01); no se redefine
- Lo que M3 produce para M1/M2: UC8 no produce eventos; solo la llamada REST a M1
