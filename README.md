# Módulo 3 – Control de Uso, Sanciones y Analítica

Especificaciones funcionales del **Módulo 3** del sistema de reservas universitario, encargado de gestionar el ciclo de uso de recursos (check-out), el cálculo y seguimiento de sanciones, el score de confianza de los estudiantes, los cobros por daño y las notificaciones asociadas.

## Diagrama de casos de uso

Ver [`Diagrama.png`](./Especs/Diagrama.png).

Casos de uso incluidos en el módulo:

| Caso de uso | Actor(es) principal(es) | Relación de disparo |
|---|---|---|
| **Realizar check-out** | Estudiante | `«extend»` Actualizar score de confianza / Calcular penalización / Reportar novedad técnica |
| **Calcular penalización** | Sistema | `«include»` desde Reportar no asistencia · `«extend»` desde Realizar check-out |
| **Actualizar score de confianza** | Sistema | `«include»` Notificar sanción · `«extend»` Bloquear usuario |
| **Bloquear usuario** | Sistema | `«extend»` desde Actualizar score de confianza |
| **Notificar sanción** | Sistema | `«include»` desde Calcular penalización / Actualizar score de confianza |
| **Reportar novedad técnica (Daño)** | Estudiante, Dirección universitaria | `«include»` Generar cobro por daño o reposición |
| **Generar cobro por daño o reposición** | Estudiante, Dirección universitaria | `«include»` desde Reportar novedad técnica |
| **Reportar no asistencia** | Dirección universitaria | `«include»` Calcular penalización |

### Integraciones externas
- **Módulo 1 (Recursos):** actualización síncrona (REST) del estado del recurso al reportar un daño.
- **Módulo 2:** notificaciones y confirmaciones (mayormente asíncronas vía cola), y consulta/pago de cobros (síncrono REST).

## Mockups de pantalla

Ver [`Pantallas.png`](./Especs/Pantallas.png): perfil del estudiante (score, penalizaciones y cobros vigentes), proceso de check-out/devolución con reporte de daño, y panel administrativo de sanciones y configuración de la escala progresiva.

## Estructura del repositorio

```
├── actualizar-score-confianza.md       # Actualizar score de confianza
├── Bloquear_usuario.md                 # Bloquear usuario
├── Calcular_penalizacion.md            # Calcular penalización
├── Diagrama.png                        # Diagrama de casos de uso del módulo
├── generar-cobro.md                    # Generar cobro por daño o reposición
├── notificar-sancion.md                # Notificar sanción
├── Pantallas.png                       # Mockups de UI (estudiante y administración)
├── realizar-checkout.md                # Realizar check-out
├── reportar-no-asistencia.md           # Reportar no asistencia
└── reportar-novedad-tecnica.md         # Reportar novedad técnica (daño)
```
