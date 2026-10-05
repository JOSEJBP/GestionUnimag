# Módulo 3 – Control de Uso, Sanciones y Analítica

Especificaciones funcionales del **Módulo 3** del sistema de reservas universitario, encargado de gestionar el ciclo de uso de recursos (check-out), el cálculo y seguimiento de sanciones, el score de confianza de los estudiantes, el registro de daños y las notificaciones asociadas.

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
| **Reportar novedad técnica (Daño)** | Estudiante, Dirección universitaria | Comunica el daño a Módulo 1 para actualizar el recurso |
| **Reportar no asistencia** | Dirección universitaria | `«include»` Calcular penalización |

### Integraciones externas
- **Módulo 1 (Recursos):** recibe el reporte de daño según su contrato y actualiza el estado del recurso.
- **Módulo 2:** recibe check-out y reporte de no asistencia según UC12 y UC9. El cálculo de penalización es interno de M3.

## Mockups de pantalla

Ver [`pantallas-2-01.png`](./Especs/pantallas-2-01.png): perfil del estudiante (score y penalizaciones), proceso de check-out/devolución con reporte de daño, y panel administrativo de sanciones y configuración de la escala progresiva.

## Estructura del repositorio

```text
├── Especs/
│   ├── Diagrama.png
│   ├── actualizar-score-confianza.md
│   ├── bloquear-usuario.md
│   ├── calcular-penalizacion.md
│   ├── gitflow.md
│   ├── notificar-sancion.md
│   ├── pantallas-2-01.png
│   ├── plan.md
│   ├── realizar-checkout.md
│   ├── reportar-no-asistencia.md
│   └── reportar-novedad-tecnica.md
├── planes-tecnicos/
│   ├── plan-integracion-kafka.md
│   ├── plan-uc-notificar-sancion.md
│   ├── plan-uc-realizar-checkout.md
│   ├── plan-uc-reportar-no-asistencia.md
│   ├── plan-uc-reportar-novedad-tecnica.md
│   ├── plan-uc1-actualizar-score.md
│   ├── plan-uc2-calcular-penalizacion.md
│   └── plan-uc3-bloquear-usuario.md
├── README.md
└── .gitignore
```

## GitFlow

Se usa una estrategia GitFlow mínima con dos ramas principales:

- `develop`: integración y desarrollo activo
- `main`: entrega y despliegue validado

La documentación detallada está en [`Especs/gitflow.md`](./Especs/gitflow.md).
