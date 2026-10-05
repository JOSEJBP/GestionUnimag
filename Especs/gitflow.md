# GitFlow del módulo

Este repositorio usa una estrategia GitFlow mínima con dos ramas principales:

- `develop`: rama de integración y trabajo diario.
- `main`: rama de entrega para ambiente/producto final.

## Flujo recomendado

1. Cada cambio nuevo se crea desde `develop`.
2. Se integran los avances en `develop` con validación técnica y revisión.
3. Cuando el estado es estable, se promociona a `main`.
4. Si se requiere un ajuste urgente en producción, se corrige desde `main` y luego se fusiona de regreso a `develop`.

## Regla de trabajo

- `develop` recibe features, fixes y mejoras del desarrollo activo.
- `main` solo debe contener versiones validadas y listas para despliegue.
- Los cambios en `main` deben volver a integrarse a `develop` para evitar divergencias.

## Diagrama de ramas

```text
develop ────────┐
    │           │
    ├─ feature/* ──> develop
    │
    └─ release/* ──> main

main ───────────┐
    │            │
    └─ hotfix/* ──> main
```

## Uso del repositorio

```bash
git checkout develop
git pull origin develop
# trabajo local

git checkout main
git pull origin main
# despliegue/estabilización
```

En este proyecto se dejó la convención concreta:

- rama de desarrollo: `develop`
- rama principal de producción: `main`
