# Implementation Plan: Gestión de Sanciones

**Date**: 2026-10-03
**Status**: initial technical proposal. Defines context, architecture, and setup tasks; implementation phases for the use cases are still pending.
**Specifications**: `Especs/` folder (8 use cases listed below)

## Summary

El módulo de Gestión de Sanciones cubre el ciclo completo de uso de recursos universitarios: el check-out de un recurso dispara, según corresponda, una actualización positiva o negativa del score de confianza del estudiante, un cálculo de penalización progresiva, o un reporte de daño técnico. Estos flujos internos terminan en dos posibles consecuencias visibles hacia afuera: un bloqueo/desbloqueo automático de cuenta y un cobro por reposición, ambos comunicados al estudiante. Los ocho casos de uso se coordinan con eventos internos. Para integraciones externas se conserva el contrato adecuado a cada operación: REST síncrono cuando se necesita respuesta inmediata (por ejemplo, actualizar el estado de un recurso del Módulo 1 o consultar/pagar un cobro en el Módulo 2) y **Apache Kafka** para notificaciones asíncronas al Módulo 2.

## Technical Context

**Language/Version**: Java 21 (LTS), versión de Spring Boot con soporte vigente al iniciar el desarrollo
**Primary Dependencies**: Spring Web, Spring Data JPA, Spring Validation, Spring Security (JWT), Spring for Apache Kafka (`spring-kafka`) para notificaciones asíncronas al Módulo 2, Spring `RestClient`/WebClient para REST síncrono con Módulos 1 y 2, Flyway (migraciones)
**Storage**: MySQL 8
**Testing**: JUnit 5 + Mockito + Spring Boot Test; Testcontainers (MySQL y Kafka) para tests de integración
**Target Platform**: Linux server, contenedores Docker (backend, frontend, MySQL y Kafka)
**Project Type**: Web application (backend + frontend separados)
**Performance Goals**: agregado de los SC de cada especificación — cálculo de penalización <2s, actualización de score <1s, notificación entregada/procesada por Módulo 2 <5s desde el cambio, bloqueo/desbloqueo <1s, consulta/pago de cobro <5s
**Constraints**: el score de confianza debe mantenerse siempre entre 0 y 50; los flujos que disparan `«extend»`/`«include»` internos no deben bloquearse esperando la notificación (Módulo 2); la actualización del estado de un recurso dañado sí requiere confirmación síncrona (Módulo 1) antes de continuar
**Scale/Scope**: 8 casos de uso de un mismo módulo, alcance institucional (una universidad, escala de miles de estudiantes y recursos)

**Frontend Framework**: React + Vite (confirmado)

**Message Broker**: Apache Kafka (confirmado, para la comunicación asíncrona de notificaciones hacia el Módulo 2)

## Module Communication

**Síncrono** significa que quien hace una solicitud espera la respuesta antes de continuar. En estas especificaciones, REST se usa para operaciones cuyo resultado se necesita conocer en ese momento:

- **Módulo 1:** confirmar el cambio de estado del recurso al reportar un daño. Si la actualización falla, el reporte queda registrado y se marca para revisión; no se debe asumir que el recurso cambió de estado.
- **Módulo 2:** consultar cobros y procesar pagos; el usuario necesita conocer el resultado de inmediato.

**Asíncrono** significa que el sistema registra/publica un mensaje y sigue trabajando sin esperar a que el otro módulo lo procese. Kafka se usa para las confirmaciones y notificaciones que Módulo 2 consume después (check-out, inasistencia, cambios de score y bloqueos). Si Módulo 2 no está disponible, el evento se conserva para reintento y no se revierte el check-out, reporte, sanción o cambio de score ya guardado.

En Kafka, el productor puede esperar la confirmación técnica de que el broker recibió el mensaje; lo que no espera es que Módulo 2 lo procese o entregue la notificación final. Se propone guardar los eventos pendientes en una tabla `outbox` dentro de la misma transacción MySQL del cambio de negocio y publicarlos después con reintentos, para no perderlos si la aplicación falla entre guardar y publicar.

| External Module | Operation from the specifications | Communication |
|---|---|---|
| Módulo 1 (Recursos) | Cambiar el recurso a mantenimiento/fuera de servicio después de registrar un daño | REST; confirmar éxito antes de considerar actualizado el recurso |
| Módulo 2 (Notificaciones) | Enviar confirmaciones y notificaciones | Kafka; no esperar el procesamiento/entrega final |
| Módulo 2 (Cobros) | Consultar cobros y procesar pagos | REST; recibir el resultado de la operación |

Para reportar un daño no existe una transacción única que abarque MySQL y el REST del Módulo 1. Por eso se persiste el reporte con estado de actualización pendiente, se llama al Módulo 1 y se marca como sincronizado solo cuando responde con éxito. Si la llamada falla, se conserva el reporte y se marca para revisión, tal como pide la especificación.

## Project Structure

### Documentation (this feature)

```text
Especs/
├── plan.md                              # This file
├── Diagrama.png
├── Realizar check-out.md
├── Calcular penalizacion.md
├── Actualizar score confianza.md
├── Bloquear usuario.md
├── Notificar sancion.md
├── Reportar no asistencia.md
├── Reportar novedad tecnica.md
└── Generar cobro.md
```

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
│   │   │   └── ChargeController.java
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
│   │   │   ├── ChargeService.java
│   │   │   └── NotificationService.java
│   │   ├── integration/             # Java interfaces for external module communication
│   │   │   ├── ResourceModuleClient.java
│   │   │   ├── NotificationPublisher.java
│   │   │   └── ChargePaymentClient.java
│   │   └── event/                   # listeners that coordinate use cases from internal events
│   │
│   ├── domain/                     # DOMAIN LAYER: business rules, models, and repository contracts
│   │   ├── model/                    # Usage, CheckOut, PenaltyRule, ProgressiveScale,
│   │   │                              # PenaltyHistory, TrustScore, DamageReport, Charge, and ID references
│   │   ├── repository/               # repository interfaces defined by the domain
│   │   │   ├── UsageRepository.java
│   │   │   ├── PenaltyRuleRepository.java
│   │   │   ├── ProgressiveScaleRepository.java
│   │   │   ├── PenaltyHistoryRepository.java
│   │   │   ├── TrustScoreRepository.java
│   │   │   ├── BlockHistoryRepository.java
│   │   │   ├── DamageReportRepository.java
│   │   │   └── ChargeRepository.java
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
│       │   │   └── rest/             # REST implementation of ResourceModuleClient
│       │   └── module2/
│       │       ├── rest/             # REST implementation of ChargePaymentClient
│       │       └── kafka/            # Kafka implementation of NotificationPublisher
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
│   │   ├── ChargesPage.jsx              # lookup/payment (student) and approval (administration)
│   │   └── ProgressiveScalesPage.jsx    # admin
│   ├── services/               # HTTP clients to the backend (one per controller)
│   └── hooks/                    # shared hooks (auth, fetch with error handling)
└── tests/
```

**Layered Architecture:** separamos el código según la responsabilidad de cada grupo. `presentation` recibe solicitudes del frontend; `business` coordina los casos de uso; `domain` contiene las reglas, conceptos del negocio y contratos de repositorio; `infrastructure` conecta esas reglas con MySQL/JPA, Kafka y otros detalles técnicos. No significa que cada carpeta sea una aplicación independiente.

**Decisión de estructura:** mantenemos un paquete raíz por capa — `presentation/`, `business/`, `domain/`, `infrastructure/` — y agrupamos las clases por responsabilidad. Las interfaces `domain/repository` describen qué datos necesita el negocio, sin depender de Spring. Dentro de `infrastructure/persistence/jpa/repository` quedan solo interfaces Spring Data JPA (`JpaRepository`); `adapter` implementa los contratos de dominio usando esas interfaces y convierte entidades JPA a modelos de dominio. Así, el negocio no importa clases JPA y la persistencia puede cambiar sin reescribir las reglas. `business/integration` contiene interfaces Java que expresan qué necesita el negocio de los otros módulos; sus implementaciones REST/Kafka viven en `infrastructure/integration`.

**Dependency Flow:**

```
presentation → business → domain
infrastructure ─────────→ domain / business
```

- **`presentation`** recibe y valida solicitudes HTTP, y llama a casos de uso de `business`.
- **`business`** coordina los casos de uso y usa interfaces de repositorio del dominio e interfaces de `business/integration`; no importa JPA, Kafka ni clientes REST concretos.
- **`domain`** define modelos, reglas y contratos de repositorio como `TrustScoreRepository`; no depende de Spring, JPA ni Kafka.
- **`infrastructure`** implementa los contratos: adapta JPA a los repositorios de dominio, REST a `ResourceModuleClient` y `ChargePaymentClient`, y Kafka a `NotificationPublisher`. Spring conecta estas implementaciones al iniciar la aplicación.

`domain/event` define qué ocurrió en términos del negocio; `business/event` reacciona a esos eventos para coordinar el siguiente caso de uso. Ninguno representa un mensaje Kafka por sí mismo.

En simple: si un caso de uso necesita guardar un cambio de score, llama a un repositorio definido en `domain`; el adaptador de infraestructura traduce esa operación a una interfaz JPA (que extiende `JpaRepository`) y a MySQL. La interfaz JPA no se usa directamente desde `business`. Se guardan los datos propios de sanciones y las referencias necesarias a estudiante/recurso; no se duplica la ficha maestra de otros módulos.

Para hablar con otro módulo, una **interfaz** es una lista de operaciones que el sistema necesita, sin decir cómo se hacen. Por ejemplo, `NotificationPublisher` podría declarar `publicarNotificacion(...)`; `NotificationService` usa esa operación sin conocer Kafka. Luego `infrastructure/integration/module2/kafka` implementa la interfaz y realiza la publicación real. Lo mismo aplica a `ResourceModuleClient` (REST al Módulo 1) y `ChargePaymentClient` (REST al Módulo 2). No son servicios extra ni temas de Kafka: son contratos Java para que la lógica del negocio no quede amarrada al mecanismo de comunicación.

Las operaciones que exigen respuesta inmediata conservan REST (Módulo 1 y consultas/pagos del Módulo 2). Kafka sustituye RabbitMQ para publicar notificaciones asíncronas al Módulo 2; un problema de entrega no revierte el caso de uso.

El plan actual solo detalla la preparación inicial. Las fases que implementan los casos de uso todavía deben agregarse; cada fase deberá indicar qué cambia en las capas y cómo se prueba de punta a punta.

## Phase 1: Setup

**Purpose**: Inicialización del proyecto y estructura básica

- [ ] T001 Crear proyecto backend (Spring Boot con versión soportada vigente, Maven, Java 21) con la estructura de paquetes de `Project Structure`
- [ ] T002 Crear proyecto frontend (React + Vite) con estructura de `Project Structure`
- [ ] T003 Configurar linting y formato: Spotless/Checkstyle (backend), ESLint + Prettier (frontend)
- [ ] T004 Configurar `docker-compose.yml` con MySQL, Kafka, backend y frontend para desarrollo local
- [ ] T005 Configurar Flyway y el primer script de migración vacío
