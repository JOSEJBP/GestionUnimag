# GitFlow del módulo

Este repositorio usa una estrategia GitFlow mínima con dos ramas principales:

- `develop`: rama de integración y trabajo diario.
- `master`: rama de entrega para ambiente/producto final.

## Flujo recomendado

1. Cada cambio nuevo se crea desde `develop`.
2. Se integran los avances en `develop` con validación técnica y revisión.
3. Cuando el estado es estable, se promociona a `master`.
4. Si se requiere un ajuste urgente en producción, se corrige desde `master` y luego se fusiona de regreso a `develop`.

## Regla de trabajo

- `develop` recibe features, fixes y mejoras del desarrollo activo.
- `master` solo debe contener versiones validadas y listas para despliegue.
- Los cambios en `master` deben volver a integrarse a `develop` para evitar divergencias.

## Diagrama de ramas

```text
develop ────────┐
    │           │
    ├─ feature/* ──> develop
    │
    └─ release/* ──> master

master ─────────┐
    │            │
    └─ hotfix/* ──> master
```

## Uso del repositorio

```bash
git checkout develop
git pull origin develop
# trabajo local

git checkout master
git pull origin master
# despliegue/estabilización
```

En este proyecto se dejó la convención concreta:

- rama de desarrollo: `develop`
- rama principal de producción: `master`
