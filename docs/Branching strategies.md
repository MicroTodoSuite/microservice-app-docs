# 🧠 **MicroTodoSuite** – Estrategias de Branching

Este documento describe y justifica las estrategias de control de versiones adoptadas en el proyecto **MicroTodoSuite**, diferenciando las prácticas utilizadas por los equipos de **Desarrollo** y **Operaciones/DevOps**, alineadas con los principios de colaboración eficiente, integración continua y despliegue en la nube.

## 📌 Contexto General del Proyecto

- **Modelo de trabajo**: Equipos pequeños y especializados (3 personas por equipo).
- **Cultura DevOps**: Automatización de pruebas, builds y despliegues.
- **Objetivo principal del proyecto**: Despliegue continuo de una aplicación de microservicios en Azure.
- **Estrategia de despliegue**: CI/CD automatizado con validaciones por pull request en desarrollo y despliegues directos en infraestructura.

## 👨‍💻 Equipo de Desarrollo: GitHub Flow

![Github Flow](./assets/Github%20Flow.png)

### ✅ Estrategia Adoptada

El equipo de desarrollo utiliza **GitHub Flow**, una estrategia ligera y ágil basada en ramas cortas y ciclos rápidos de integración. El flujo se compone de:

1. **Rama principal (`main`)** como línea base siempre desplegable.
2. **Ramas de características (`feat/*`)** para nuevas funcionalidades o correcciones.
3. **Pull Requests** hacia `main` para revisión de código, pruebas automáticas y aprobación.
4. **Merge en `main`** luego de superar pruebas y revisión.

### 🤔 Justificación

| Factor                   | GitHub Flow – Ventajas clave                                                                    |
| ------------------------ | ----------------------------------------------------------------------------------------------- |
| 🔄 Frecuencia de cambios | Ciclos cortos, ideal para microservicios iterativos.                                            |
| 👥 Tamaño del equipo     | Permite colaboración simple y efectiva en equipos pequeños.                                     |
| 🚀 Integración continua  | Facilita pipelines CI/CD que despliegan automáticamente a staging o producción tras cada merge. |
| 🧪 Automatización        | Soporte completo para testing en cada Pull Request.                                             |
| 🔄 Revisión colaborativa | Promueve code reviews constantes y mejora la calidad del código.                                |

GitHub Flow es especialmente eficaz en contextos donde el producto está en evolución constante y se requiere **frecuencia alta de integración** sin la complejidad de múltiples ramas de largo plazo (como `develop` o `release`).

## ⚙️ Equipo de Operaciones: Trunk-Based Development

![Trunk-Based Development](./assets/Trunk-Based%20Development.png)

### ✅ Estrategia Adoptada

El equipo de operaciones utiliza **Trunk-Based Development**, una estrategia que promueve trabajar directamente sobre la rama principal (`main`), con cambios pequeños, revisados y desplegados frecuentemente.

1. Cambios directos sobre `main` (mediante PRs).
2. Validaciones automáticas (lint, validación de Terraform).
3. Despliegues inmediatos a entornos de prueba o producción tras cada commit exitoso.

### 🤔 Justificación

| Factor                           | Trunk-Based – Ventajas clave                                                     |
| -------------------------------- | -------------------------------------------------------------------------------- |
| ⏱️ Rapidez en cambios            | Permite aplicar cambios inmediatos en scripts, IAC o configuración.              |
| ⚙️ Automatización DevOps         | Compatible con pipelines de infraestructura como código (IaC).                   |
| 📉 Baja complejidad de branching | Evita ramas técnicas innecesarias que complican el estado de la infraestructura. |
| 💥 Minimiza conflictos           | En infra, los conflictos en ramas largas suelen ser costosos.                    |
| 👥 Coordinación del equipo       | Con 3 personas, es más simple mantener sincronía con una sola rama base.         |

La naturaleza **declarativa y atómica de la infraestructura como código (IaC)** hace que Trunk-Based Development sea una elección natural, ya que los cambios deben integrarse rápidamente y ser auditables en un entorno productivo con el menor desfase posible.

## ⚖️ Comparación entre ambas estrategias

| Criterio                     | GitHub Flow (Dev)            | Trunk-Based (Ops)                   |
| ---------------------------- | ---------------------------- | ----------------------------------- |
| Uso de ramas                 | `main` + `feat/*`            | Solo `main`                         |
| Pull Requests                | Obligatorios                 | Obligatorios                        |
| Frecuencia de despliegue     | Alta (tras merge a `main`)   | Muy alta (tras cada commit)         |
| Complejidad de la estrategia | Baja                         | Muy baja                            |
| Riesgo de conflictos         | Bajo (con buenas PR reviews) | Mínimo (cambios pequeños y rápidos) |

## 📌 Conclusión

Las estrategias seleccionadas se alinean con los **objetivos técnicos** del proyecto y la **estructura organizacional** del equipo. GitHub Flow permite a los desarrolladores mantener un ciclo de entrega ágil, revisado y automatizado; mientras que Trunk-Based Development proporciona al equipo de operaciones una vía directa, segura y rápida para actualizar la infraestructura y los procesos de despliegue.

Ambas estrategias refuerzan la cultura de **despliegue continuo**, reduciendo tiempos de entrega, evitando ramas divergentes y fomentando la calidad mediante automatización y revisión colaborativa.
