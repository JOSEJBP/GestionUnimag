# Índice de planes: Módulo 3 — Control de Uso, Sanciones y Analítica

**Date**: 2026-10-23

El Módulo 3 se implementa con **seis planes, uno por caso de uso**, todos con la misma estructura (Summary, Technical Context, Module Communication, Project Structure, fases y tareas, Dependencies, Notes). Este índice define el orden, qué plan construye los fundamentos compartidos y deja en un solo lugar la vista completa de comunicación y estructura.

## Planes y orden de implementación

| Orden | Plan | Caso de uso | Depende de |
|---|---|---|---|
| 1 | `plan-registrar-checkout.md` | Registrar check-out | — (**construye el Setup y los fundamentos compartidos**) |
| 2 | `plan-reportar-novedad-tecnica.md` | Reportar novedad técnica (incluye cobros) | Plan 1 (fases 1 a 3) |
| 3 | `plan-confirmar-no-asistencia.md` | Confirmar no asistencia | Plan 1 (fases 1 y 2) |
| 4 | `plan-desbloquear-usuario.md` | Desbloquear usuario | Plan 1 (fases 1 y 2); mejor probado con el plan 3 |
| 5 | `plan-consultar-cumplimiento.md` | Consultar mi cumplimiento (incluye la consulta de M2) | Plan 1 (fases 1 y 2); se completa con los planes 1 a 4 |
| 6 | `plan-consultar-analitica-cumplimiento.md` | Consultar analítica de cumplimiento | Lee los datos de los cinco anteriores |

Los planes 2, 3 y 4 pueden avanzar en paralelo una vez terminado el plan 1. Conviene adelantar la consulta de M2 del plan 5 (sin ella, M2 no puede aplicar sanciones ni bloqueos); el plan 6 va al final porque lee los datos de todos.

La numeración de las tareas (T001, T002…) **reinicia en cada plan**. Cuando un plan necesita una tarea de otro, la cita como `plan-registrar-checkout T022`.

## Fundamentos compartidos (los construye el plan 1)

| Componente | Tareas en `plan-registrar-checkout` | Lo usan |
|---|---|---|
| Proyecto backend/frontend, linting, Docker (MySQL, Kafka), Flyway | T001–T005 | todos |
| Parámetros del negocio (`SystemConfiguration`) | T006 | todos |
| Copia local de reservas de M2 y sus consumidores | T008, T021 | check-out, inasistencia |
| Score de confianza e historial, `TrustScoreService` | T009, T022 | check-out, inasistencia, desbloqueo, cumplimiento |
| Sanciones, `SanctionService` | T010, T023 | inasistencia, desbloqueo, cumplimiento |
| Seguridad (JWT para el frontend y cadena sin JWT para M1 y M2) | T012 | todos |
| Bus de eventos de dominio | T016 | check-out, novedad, inasistencia |
| Outbox, inbox, DLT y configuración de Kafka | T017–T019 | check-out, novedad, inasistencia |
| `UsageEventsPublisher` (M1) y `ReservationModulePublisher` (M2) | T020 | check-out, novedad, inasistencia |

## Relaciones entre módulos (corregidas respecto a los planes anteriores)

- **M1** no es una API que M3 llame para cambiar el estado de un recurso: recibe `reserva.finalizada` y `reporte.danos` por Kafka. El `PATCH` de estado de M1 es solo para personas. M1 sí llama a M3 por REST para saber si hay novedades pendientes.
- **M2** es el módulo de **Reservas**, no de notificaciones ni de cobros. No expone HTTP a otros módulos: M3 recibe sus reservas por Kafka, le publica el check-out y la inasistencia, y M2 consulta a M3 las sanciones por REST.
- No se confirma de forma síncrona el cambio de estado de un recurso dañado: el reporte queda "Pendiente" y se resuelve cuando M1 informa por `estado.actualizado`.
- Kafka reemplaza a RabbitMQ en todo el módulo, y el outbox se complementa con un inbox por `eventId`.
- Son seis casos de uso, no ocho: el retraso se castiga con un descuento diario de score (no hay sanción por mora), el bloqueo se deriva de score = 0 y no hay notificaciones.

## Pendientes de contrato con M1 y M2

- Payloads de `reserva.finalizada` y `reporte.danos`, y si una devolución tardía sin daño también usa `reserva.finalizada`.
- Que `estado.actualizado` de M1 se emita también al salir de mantenimiento.
- Los tres tópicos "por acordar" de M2: acuse de no asistencia, acuse de check-out y `LoanDeclaredLost`.
- Qué hace M2 al recibir `NoShowReportVoided`; ventana y significado de `absenceCount` y `lateReturns`; texto de `reason`.
- De dónde sale la facultad para la analítica, y si M2 avisará a M3 el inicio de uso de una reserva.
- Si las llamadas REST entre módulos van sin autenticación también del lado de M1.

## Comunicación entre módulos (vista completa)

**Regla de oro**: M3 se comunica con M1 y M2 **únicamente por Kafka**. Las dos únicas llamadas REST entre módulos son **entrantes** a M3. Las llamadas entre módulos van sin autenticación (así lo asume M2; confirmar con M1), por lo que esas dos rutas tienen su propia cadena de seguridad, separada del JWT del frontend.

**Síncrono** significa que quien hace la solicitud espera la respuesta. **Asíncrono** significa que el sistema guarda/publica un mensaje y sigue trabajando sin esperar a que el otro módulo lo procese. En Kafka, el productor puede esperar la confirmación técnica de que el broker recibió el mensaje, pero no que el otro módulo lo procese.

### Lo que M3 publica (Kafka)

| Destino | Tópico | Evento | Cuándo | Estado del contrato |
|---|---|---|---|---|
| M1 | `unimag.m3.uso-novedades.v1` | `reserva.finalizada` | Check-out **sin** novedad técnica | Tópico definido; payload en blanco |
| M1 | `unimag.m3.uso-novedades.v1` | `reporte.danos` | Reporte de daño (en el check-out o independiente) | Tópico definido; payload en blanco |
| M2 | `module3.reservation.check-out.v1` | `CheckOutRegistered` | Cada check-out; en espacios lleva `verdict` | Propuesto por M2 |
| M2 | `module3.reservation.no-show.v1` | `NoShowReported`, `NoShowReportVoided` | Al confirmar o anular una inasistencia | Propuesto por M2 |

### Lo que M3 consume (Kafka)

| Origen | Tópico | Evento | Para qué | Estado del contrato |
|---|---|---|---|---|
| M2 | `module2.reservation.record.v1` | `ReservationRecordCreated` | Copia local de reservas y préstamos | Acordado en el plan general de M2 |
| M2 | `module2.reservation.record.v1` | `LoanDeclaredLost` | Expediente de pérdida y cobro | Por acordar |
| M2 | `module2.reservation.cancellation.v1` | `ReservationCancelled` | Excluir reservas canceladas | Acordado en el plan general de M2 |
| M2 | `module2.reservation.no-show-ack.v1` | `NoShowReportAcknowledged` | Crear la sanción al ser aceptado | Por acordar |
| M2 | `module2.reservation.check-out-ack.v1` | `CheckOutAcknowledged` | Conciliar el check-out | Por acordar |
| M1 | `unimag.m1.recursos-estados.v1` | `estado.actualizado` | Resolver novedades al salir de mantenimiento | Definido por M1 |

### Lo que M3 expone (REST entrante)

| Quién llama | Endpoint | Respuesta |
|---|---|---|
| M2 | `GET /api/v1/compliance/people/{code}` | `activeSanctions[{reason, startsOn, endsOn, scope}]`, `absenceCount`, `lateReturns`, `generatedAt`; el bloqueo (score 0) aparece como una sanción sin `endsOn` y con `scope` `TODO` |
| M1 | `GET /api/v1/novedades/pendientes?recursoId={id}` | `{ "recursoId", "tieneNovedadPendiente" }` |

### Lo que M3 no hace

- No llama por REST a M1 ni a M2. M1 cambia el estado del recurso escuchando los eventos de M3 (el `PATCH` de estado de M1 es solo para personas), y M2 conoce las sanciones preguntándole a M3.
- No publica eventos de sanción, bloqueo ni desbloqueo: M2 consulta en cada operación y no guarda copia.
- No envía notificaciones al estudiante (el caso de uso de notificación quedó descartado); el estudiante ve su estado en *Consultar mi cumplimiento*.
- No procesa pagos en línea: el recaudo ocurre fuera de M3 y el personal administrativo registra el pago o la anulación del cobro.
- No acepta ni envía datos de daño a M2: el daño va solo de M3 a M1.

### Garantías de entrega

- **Outbox** (`outbox_message`): el evento se guarda en la misma transacción MySQL del cambio de negocio y un publicador lo envía después, con reintentos y espera creciente. Si la aplicación cae entre guardar y publicar, el evento no se pierde.
- **Inbox** (`processed_event`): cada evento recibido se registra por `eventId` antes de procesarse; una reentrega se descarta. El *offset* se confirma después de guardar.
- **DLT** (`module3.dlt.<tópico-de-origen>`): eventos mal formados o sobre una reserva desconocida. Un rechazo de negocio de M2 llega como acuse, no al DLT.
- **Convenciones de cada módulo**: los eventos hacia M2 usan el sobre de M2 (`eventId`, `type`, `version`, `occurredAt`, `data`, clave `reservation_id`); los eventos hacia M1 usan las cabeceras de M1 (`X-Correlation-Id`, `X-Event-Type`, `X-Source-Module`, `X-Event-Version`, clave `recurso_id`).
- **Daño y estado del recurso**: no existe una transacción que abarque MySQL y M1. M3 guarda el reporte en estado "Pendiente" y el evento en el outbox en la misma transacción; M1 aplica el estado por su lado. M3 nunca asume que el recurso cambió de estado: la novedad pasa a "Resuelta" solo cuando M1 informa por `estado.actualizado` que salió de mantenimiento.

## Estructura completa del backend y frontend

```text
backend/
├── src/main/java/com/university/sanctions/
│   ├── presentation/            # PRESENTATION LAYER
│   │   ├── controller/            # REST controllers para el frontend (JWT)
│   │   │   ├── CheckOutController.java
│   │   │   ├── DamageReportController.java
│   │   │   ├── ChargeController.java              # pago y anulación (personal administrativo)
│   │   │   ├── AbsenceController.java             # pendientes, confirmar, anular
│   │   │   ├── UnblockController.java             # bloqueados y desbloqueo manual
│   │   │   ├── ComplianceController.java          # cumplimiento del propio estudiante
│   │   │   ├── AnalyticsController.java           # indicadores, rankings, exportación, detalle por estudiante
│   │   │   ├── AdminConfigController.java         # parámetros y catálogo de costos
│   │   │   └── external/                          # REST entrante de otros módulos (sin JWT)
│   │   │       ├── ModuleComplianceController.java    # M2 → M3
│   │   │       └── PendingIssuesController.java       # M1 → M3
│   │   └── dto/                     # request y response de cada endpoint
│   │
│   ├── business/                  # BUSINESS LOGIC LAYER
│   │   ├── service/                # servicios de aplicación de los casos de uso
│   │   │   ├── CheckOutService.java
│   │   │   ├── OverdueLoanService.java            # descuento diario por retraso
│   │   │   ├── DamageReportService.java
│   │   │   ├── ChargeService.java
│   │   │   ├── AbsenceService.java
│   │   │   ├── SanctionService.java
│   │   │   ├── TrustScoreService.java
│   │   │   ├── RecoveryService.java               # recuperación de puntos
│   │   │   ├── UnblockService.java
│   │   │   ├── ReservationMirrorService.java      # copia local de reservas de M2
│   │   │   ├── ComplianceQueryService.java
│   │   │   ├── AnalyticsService.java
│   │   │   └── CostCatalogService.java
│   │   ├── integration/             # interfaces Java de lo que el negocio necesita de otros módulos
│   │   │   ├── UsageEventsPublisher.java          # hacia M1: devolución y daño
│   │   │   ├── ReservationModulePublisher.java    # hacia M2: check-out, inasistencia, anulación
│   │   │   └── EvidenceStorage.java               # archivos de evidencia del daño
│   │   └── event/                   # listeners que coordinan casos de uso a partir de eventos internos
│   │
│   ├── domain/                     # DOMAIN LAYER: reglas, modelos y contratos de repositorio
│   │   ├── model/                    # ReservationCopy, Usage, CheckOut, OverdueDeduction, LostLoan,
│   │   │                              # AbsenceConfirmation, Sanction, TrustScore, ScoreHistory,
│   │   │                              # DamageReport, Charge, ChargeAudit, CostCatalog, UnblockAudit,
│   │   │                              # SystemConfiguration, StudentRef
│   │   ├── repository/               # interfaces de repositorio definidas por el dominio
│   │   │   ├── ReservationCopyRepository.java
│   │   │   ├── UsageRepository.java
│   │   │   ├── CheckOutRepository.java
│   │   │   ├── OverdueDeductionRepository.java
│   │   │   ├── LostLoanRepository.java
│   │   │   ├── AbsenceConfirmationRepository.java
│   │   │   ├── SanctionRepository.java
│   │   │   ├── TrustScoreRepository.java
│   │   │   ├── ScoreHistoryRepository.java
│   │   │   ├── DamageReportRepository.java
│   │   │   ├── ChargeRepository.java
│   │   │   ├── CostCatalogRepository.java
│   │   │   ├── UnblockAuditRepository.java
│   │   │   └── SystemConfigurationRepository.java
│   │   └── event/                    # eventos de negocio, independientes de Kafka:
│   │                                  # CheckOutRecorded, DamageReported, AbsenceAccepted,
│   │                                  # AbsenceAnnulled, LoanDeclaredLost, ScoreChanged
│   │
│   └── infrastructure/            # INFRASTRUCTURE LAYER: implementa los contratos con tecnologías externas
│       ├── persistence/
│       │   └── jpa/
│       │       ├── entity/           # entidades JPA mapeadas a tablas
│       │       ├── repository/       # solo interfaces Spring Data JPA (JpaXxxRepository)
│       │       ├── adapter/          # implementan los repositorios del dominio con las interfaces JPA
│       │       └── mapper/           # conversión entre modelos de dominio y entidades JPA
│       ├── integration/
│       │   ├── module1/
│       │   │   └── kafka/            # UsageEventsKafkaPublisher (reserva.finalizada, reporte.danos),
│       │   │                          # ResourceStateConsumer (estado.actualizado)
│       │   └── module2/
│       │       └── kafka/            # ReservationModuleKafkaPublisher (check-out, no-show, void),
│       │                              # consumidores: ReservationRecordConsumer, CancellationConsumer,
│       │                              # NoShowAckConsumer, CheckOutAckConsumer, LostLoanConsumer
│       ├── messaging/
│       │   ├── kafka/                # configuración, serializadores de sobre (estilo M1 y M2), DLT
│       │   ├── outbox/               # publicación durable de eventos de negocio
│       │   └── inbox/                # deduplicación por eventId
│       ├── scheduling/               # OverdueLoanJob, RecoveryJob, SanctionExpiryJob, AckResendJob
│       ├── storage/                  # implementación de EvidenceStorage
│       └── config/                   # seguridad (dos cadenas: JWT y llamadas entre módulos) y configuración técnica
│
└── src/test/java/com/university/sanctions/
    ├── contract/        # espejan presentation/controller/ y los contratos de eventos (fixtures de M1 y M2)
    ├── integration/     # pruebas de business e infrastructure/persistence con Testcontainers
    └── unit/

frontend/
├── index.html
├── vite.config.js
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── components/          # componentes reutilizables (formularios, tablas, badges de estado)
│   ├── pages/                # una vista por caso de uso, orientada a estudiante o a personal administrativo
│   │   ├── CheckOutPage.jsx             # estudiante
│   │   ├── CompliancePage.jsx           # estudiante: score, historial, sanciones, cobros, avisos
│   │   ├── DamageReportPage.jsx         # personal administrativo
│   │   ├── ChargesPage.jsx              # personal administrativo: pago y anulación
│   │   ├── AbsencePage.jsx              # personal administrativo: pendientes, confirmar, anular
│   │   ├── BlockedStudentsPage.jsx      # personal administrativo: bloqueados y desbloqueo
│   │   ├── AnalyticsPage.jsx            # personal administrativo: indicadores, rankings, exportación
│   │   └── StudentDetailPage.jsx        # personal administrativo: detalle por estudiante
│   ├── services/               # clientes HTTP hacia el backend (uno por controller)
│   └── hooks/                    # hooks compartidos (auth, fetch con manejo de error)
└── tests/
```

**Layered Architecture:** separamos el código según la responsabilidad de cada grupo. `presentation` recibe solicitudes; `business` coordina los casos de uso; `domain` contiene las reglas, los conceptos del negocio y los contratos de repositorio; `infrastructure` conecta esas reglas con MySQL/JPA, Kafka y otros detalles técnicos. No significa que cada carpeta sea una aplicación independiente.

**Decisión de estructura:** un paquete raíz por capa — `presentation/`, `business/`, `domain/`, `infrastructure/` — con las clases agrupadas por responsabilidad. Las interfaces de `domain/repository` describen qué datos necesita el negocio, sin depender de Spring. En `infrastructure/persistence/jpa/repository` quedan solo interfaces Spring Data JPA; `adapter` implementa los contratos del dominio con ellas y convierte entidades JPA a modelos de dominio, de modo que el negocio no importa clases JPA. Se guardan los datos propios de M3 y las referencias necesarias a estudiante y recurso; no se duplica la ficha maestra de otros módulos (la copia local de reservas guarda solo lo que M2 informa para decidir check-outs e inasistencias).

**Dependency Flow:**

```
presentation → business → domain
infrastructure ─────────→ domain / business
```

- **`presentation`** recibe y valida solicitudes HTTP y llama a casos de uso de `business`. Los controladores de `external/` atienden a M1 y M2 y no usan JWT.
- **`business`** coordina los casos de uso con las interfaces de repositorio del dominio y las de `business/integration`; no importa JPA, Kafka ni clientes concretos. Un servicio nunca llama al servicio de otro caso de uso: se comunican por evento.
- **`domain`** define modelos, reglas y contratos de repositorio; no depende de Spring, JPA ni Kafka.
- **`infrastructure`** implementa los contratos: JPA para los repositorios, Kafka para `UsageEventsPublisher` y `ReservationModulePublisher`, y los consumidores de Kafka que entregan los mensajes de M1 y M2 a servicios de `business`. También aloja los procesos programados.

`domain/event` define qué ocurrió en términos del negocio; `business/event` reacciona para coordinar el siguiente caso de uso. Ninguno es un mensaje Kafka por sí mismo: el mensaje se construye en `infrastructure` y sale por el outbox.
