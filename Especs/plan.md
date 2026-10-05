# Implementation Plan: Gestión de Sanciones

**Date**: 2026-10-03
**Status**: initial technical proposal. Defines context, architecture, and setup tasks; implementation phases for the use cases are still pending.
**Specifications**: `Especs/` folder (7 use cases listed below)

## Summary

El Módulo 3 cubre el check-out, el cálculo interno de penalizaciones, la actualización del score, los bloqueos y el registro de daños. Al reportar un daño, M3 comunica el evento a Módulo 1, que es dueño del inventario y actualiza el estado del recurso conforme a su contrato. Los contratos externos solo se implementan conforme a los documentos de los módulos responsables.

## Technical Context

**Language/Version**: Java 21 (LTS), versión de Spring Boot con soporte vigente al iniciar el desarrollo
**Primary Dependencies**: Spring Web, Spring Data JPA, Spring Validation, Spring Security (JWT), Spring for Apache Kafka (`spring-kafka`) y clientes de integración definidos por los contratos acordados con Módulos 1 y 2, Flyway (migraciones)
**Storage**: MySQL 8 para datos y metadatos. Para los archivos binarios de evidencia de UC8 se propone almacenamiento persistente separado de la BD; capacidad, retención y respaldos siguen pendientes de definición en [NC-04](../planes-tecnicos/plan-uc-reportar-novedad-tecnica.md#needs-clarification-abiertos).
**Testing**: JUnit 5 + Mockito + Spring Boot Test; Testcontainers (MySQL y Kafka) para tests de integración
**Target Platform**: Linux server, contenedores Docker (backend, frontend, MySQL y Kafka)
**Project Type**: Web application (backend + frontend separados)
**Performance Goals**: agregado de los SC de los casos de uso de M3 — cálculo de penalización <2s, actualización de score <1s y bloqueo/desbloqueo <1s
**Constraints**: el score de confianza debe mantenerse siempre entre 0 y 50; los flujos internos no deben bloquearse esperando integraciones asíncronas; la actualización del estado de un recurso dañado sigue el contrato de Módulo 1
**Scale/Scope**: 7 casos de uso de M3, alcance institucional (una universidad, escala de miles de estudiantes y recursos).

**Frontend Framework**: React + Vite (confirmado)

**Message Broker**: Apache Kafka para los contratos asíncronos establecidos con Módulos 1 y 2

## Module Communication

**Síncrono** significa que quien hace una solicitud espera la respuesta antes de continuar. El protocolo de cada integración se determina por el contrato del módulo destino.

- **Módulo 1:** recibir de M3 los eventos de check-out y reporte de daño definidos en su contrato Kafka. M1 aplica los cambios de estado de los recursos.
- **Módulo 2:** recibir el check-out y el reporte de no asistencia según UC12 y UC9. El cálculo de penalización es interno de M3; no se envía a M2 como un evento de penalización.

**Asíncrono** significa que el sistema registra/publica un mensaje y sigue trabajando sin esperar a que el otro módulo lo procese. Kafka se usa únicamente donde lo indiquen los contratos de las integraciones establecidas con Módulos 1 y 2. Los eventos pendientes deben poder reintentarse sin revertir los cambios de negocio ya guardados.

Los contratos Kafka entre M3 y M1/M2, incluidos los nombres, topics, payloads y pendientes observados en sus documentos fuente, se consolidan en [plan-integracion-kafka.md](../planes-tecnicos/plan-integracion-kafka.md). Los campos o topics que siguen inconsistentes o requieren acuerdo se distinguen de los contratos fuente y no se completan por inferencia.

En Kafka, el productor puede esperar la confirmación técnica de que el broker recibió el mensaje; el flujo originador no espera el procesamiento posterior. Se propone guardar los eventos pendientes en una tabla `outbox` dentro de la misma transacción MySQL del cambio de negocio y publicarlos después con reintentos, para no perderlos si la aplicación falla entre guardar y publicar.

| External Module | Operation from the specifications | Communication |
|---|---|---|
| Módulo 1 (Recursos) | Recibir el reporte de daño y actualizar el estado del recurso | Evento Kafka de M3 según el contrato de M1 |
| Módulo 1 (Recursos) | Recibir el check-out para actualizar la disponibilidad del recurso | Evento Kafka de M3 según el contrato de M1 |
| Módulo 2 | Recibir el check-out | Según M2 UC12 |
| Módulo 2 | Recibir reporte de no asistencia | Según M2 UC9 |

Los cambios de estado del inventario pertenecen a M1. M3 registra el check-out o el daño y publica el evento definido por el contrato Kafka de M1; no actualiza la base de inventario de M1 directamente. La confirmación del procesamiento y cualquier evento de respuesta se rigen por el contrato de M1.

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan.md                              # Este archivo
├── gitflow.md                           # Estrategia de ramas (develop / main)
├── Diagrama.png
├── pantallas-2-01.png                   # Mockups de UI
├── realizar-checkout.md
├── calcular-penalizacion.md
├── actualizar-score-confianza.md
├── bloquear-usuario.md
├── notificar-sancion.md
├── reportar-no-asistencia.md
├── reportar-novedad-tecnica.md
└── ...

planes-tecnicos/
├── plan-integracion-kafka.md
├── plan-uc-realizar-checkout.md
├── plan-uc1-actualizar-score.md
├── plan-uc2-calcular-penalizacion.md
├── plan-uc3-bloquear-usuario.md
├── plan-uc-notificar-sancion.md
├── plan-uc-reportar-no-asistencia.md
├── plan-uc-reportar-novedad-tecnica.md
└── ...
```

## GitFlow

Se define la estrategia de ramas como sigue:

- `develop`: rama de integración y evolución del módulo.
- `main`: rama de despliegue y entrega validada.

Las reglas y el flujo recomendado están documentados en [`gitflow.md`](./gitflow.md).

### Source Code (repository root)

```text
backend/
├── src/main/java/com/university/sanctions/
│   ├── presentation/            # PRESENTATION LAYER
│   │   ├── controller/            # REST controllers for frontend endpoints
│   │   │   ├── CheckOutController.java
│   │   │   ├── AbsenceController.java
│   │   │   ├── PenaltyController.java             # progressive scale administration
│   │   │   ├── TrustScoreController.java          # trust score lookup
│   │   │   ├── BlockController.java                # manual unblock
│   │   │   ├── DamageReportController.java
│   │   └── dto/                     # request and response data transfer objects
│   │
│   ├── business/                  # BUSINESS LOGIC LAYER
│   │   ├── service/                # application services for use cases
│   │   │   ├── CheckOutService.java
│   │   │   ├── AbsenceService.java
│   │   │   ├── PenaltyService.java
│   │   │   ├── TrustScoreService.java
│   │   │   ├── BlockService.java
│   │   │   ├── DamageReportService.java
│   │   │   └── NotificationService.java
│   │   ├── integration/             # Java interfaces for external module communication
│   │   │   ├── NotificationPublisher.java
│   │   └── event/                   # listeners that coordinate use cases from internal events
│   │
│   ├── domain/                     # DOMAIN LAYER: business rules, models, and repository contracts
│   │   ├── model/                    # Usage, CheckOut, PenaltyRule, ProgressiveScale,
│   │   │                              # PenaltyHistory, TrustScore, DamageReport and ID references
│   │   ├── repository/               # repository interfaces defined by the domain
│   │   │   ├── UsageRepository.java
│   │   │   ├── PenaltyRuleRepository.java
│   │   │   ├── ProgressiveScaleRepository.java
│   │   │   ├── PenaltyHistoryRepository.java
│   │   │   ├── TrustScoreRepository.java
│   │   │   ├── BlockHistoryRepository.java
│   │   │   ├── DamageReportRepository.java
│   │   └── event/                    # business event types, independent of Kafka
│   │
│   └── infrastructure/            # INFRASTRUCTURE LAYER: implements contracts using external technologies
│       ├── persistence/
│       │   └── jpa/
│       │       ├── entity/           # JPA entities mapped to database tables
│       │       ├── repository/       # JPA interfaces only: JpaStudentRepository, etc.
│       │       ├── adapter/          # domain repository implementations using JpaRepository interfaces
│       │       └── mapper/           # mappings between domain models and JPA entities
│       ├── integration/
│       │   ├── module1/
│       │   │   └── kafka/            # eventos de ciclo de vida de recursos según contrato de M1
│       │   └── module2/
│       │       └── kafka/            # adaptadores de los tres flujos establecidos con M2
│       ├── messaging/
│       │   ├── kafka/                # Kafka configuration, listeners, and publishers
│       │   └── outbox/               # durable publishing of business events
│       └── config/                   # Spring security and infrastructure configuration
│
└── src/test/java/com/university/sanctions/
    ├── contract/        # mirrors presentation/controller/
    ├── integration/     # business and infrastructure/persistence integration tests
    └── unit/

frontend/
├── index.html
├── vite.config.js
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── components/          # reusable components (forms, tables, status badges)
│   ├── pages/                # one view per use case, student- or admin-facing
│   │   ├── CheckOutPage.jsx
│   │   ├── TrustScorePage.jsx           # trust score and history lookup
│   │   ├── AbsencePage.jsx              # university administration
│   │   ├── DamageReportPage.jsx
│   │   └── ProgressiveScalesPage.jsx    # admin
│   ├── services/               # HTTP clients to the backend (one per controller)
│   └── hooks/                    # shared hooks (auth, fetch with error handling)
└── tests/
```

**Layered Architecture:** separamos el código según la responsabilidad de cada grupo. `presentation` recibe solicitudes del frontend; `business` coordina los casos de uso; `domain` contiene las reglas, conceptos del negocio y contratos de repositorio; `infrastructure` conecta esas reglas con MySQL/JPA, Kafka y otros detalles técnicos. No significa que cada carpeta sea una aplicación independiente.

**Decisión de estructura:** mantenemos un paquete raíz por capa — `presentation/`, `business/`, `domain/`, `infrastructure/` — y agrupamos las clases por responsabilidad. Las interfaces `domain/repository` describen qué datos necesita el negocio, sin depender de Spring. Dentro de `infrastructure/persistence/jpa/repository` quedan solo interfaces Spring Data JPA (`JpaRepository`); `adapter` implementa los contratos de dominio usando esas interfaces y convierte entidades JPA a modelos de dominio. Así, el negocio no importa clases JPA y la persistencia puede cambiar sin reescribir las reglas. Las integraciones con M1/M2 respetan el protocolo definido por cada contrato y sus adaptadores viven en `infrastructure/integration`.

**Dependency Flow:**

```
presentation → business → domain
infrastructure ─────────→ domain / business
```

- **`presentation`** recibe y valida solicitudes HTTP, y llama a casos de uso de `business`.
- **`business`** coordina los casos de uso y usa interfaces de repositorio del dominio e interfaces de `business/integration`; no importa JPA, Kafka ni clientes REST concretos.
- **`domain`** define modelos, reglas y contratos de repositorio como `TrustScoreRepository`; no depende de Spring, JPA ni Kafka.
- **`infrastructure`** implementa los contratos: adapta JPA a los repositorios de dominio, publica eventos de recursos a M1 según su contrato, implementa las integraciones acordadas con M2 y conecta los publicadores asíncronos con Kafka. Spring conecta estas implementaciones al iniciar la aplicación.

`domain/event` define qué ocurrió en términos del negocio; `business/event` reacciona a esos eventos para coordinar el siguiente caso de uso. Ninguno representa un mensaje Kafka por sí mismo.

En simple: si un caso de uso necesita guardar un cambio de score, llama a un repositorio definido en `domain`; el adaptador de infraestructura traduce esa operación a una interfaz JPA (que extiende `JpaRepository`) y a MySQL. La interfaz JPA no se usa directamente desde `business`. Se guardan los datos propios de sanciones y las referencias necesarias a estudiante/recurso; no se duplica la ficha maestra de otros módulos.

Para hablar con otro módulo, una **interfaz** es una lista de operaciones que el sistema necesita, sin decir cómo se hacen. Las implementaciones de las integraciones con M1 y de los flujos establecidos con M2 se mantienen en infraestructura y siguen sus respectivos contratos; no se deben inventar operaciones adicionales.

El protocolo y la asincronía de cada integración se determinan por el contrato del módulo destino.

El plan actual solo detalla la preparación inicial. Las fases que implementan los casos de uso todavía deben agregarse; cada fase deberá indicar qué cambia en las capas y cómo se prueba de punta a punta.

## Phase 1: Setup

**Purpose**: Inicialización del proyecto y estructura básica

- [ ] T001 Crear proyecto backend (Spring Boot con versión soportada vigente, Maven, Java 21) con la estructura de paquetes de `Project Structure`
- [ ] T002 Crear proyecto frontend (React + Vite) con estructura de `Project Structure`
- [ ] T003 Configurar linting y formato: Spotless/Checkstyle (backend), ESLint + Prettier (frontend)
- [ ] T004 Configurar `docker-compose.yml` con MySQL, Kafka, backend y frontend para desarrollo local
- [ ] T005 Configurar Flyway y el primer script de migración vacío
