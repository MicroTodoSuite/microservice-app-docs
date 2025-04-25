# 🧠 **MicroTodoSuite** – Metodología Ágil

Este documento describe la metodología ágil adoptada para la gestión del proyecto **MicroTodoSuite**, explicando por qué
**Kanban** es la opción más adecuada considerando las características del equipo, los objetivos del proyecto y el
enfoque de despliegue continuo en la nube.

## 📌 Características del Proyecto

| Elemento                       | Descripción                                                                                |
|--------------------------------|--------------------------------------------------------------------------------------------|
| 🎯 Objetivo principal          | Despliegue automatizado de infraestructura para una aplicación de microservicios en Azure. |
| 🧑‍🤝‍🧑 Estructura del equipo | 2 equipos funcionales (Desarrollo y Operaciones), cada uno con 3 personas.                 |
| 🔁 Cadencia de entregas        | Flujo continuo con despliegues incrementales diarios y entregas bajo demanda.              |
| 🛠️ Cultura DevOps             | Automatización de CI/CD, monitoreo con Prometheus y Grafana, y IaC con Terraform.          |
| ☁️ Infraestructura dinámica    | Uso de recursos en la nube gestionados como código y desacoplados por servicio.            |

## ✅ Metodología Elegida: **Kanban Dual**

Hemos implementado un modelo **Kanban Dual** con dos tableros independientes pero coordinados:

- **Kanban - Development Team**: Gestión del ciclo de vida de desarrollo de microservicios
- **Kanban - Operations Team**: Gestión de despliegues, monitorización y operaciones en la nube

### ¿Por qué Kanban?

| Factor                    | Justificación en MicroTodoSuite                                                                               |
|---------------------------|---------------------------------------------------------------------------------------------------------------|
| 🎯 Flujo continuo         | Permite entregas bajo demanda sin iteraciones rígidas, ideal para despliegues frecuentes en la nube.          |
| 📊 Visualización dual     | Dos tableros especializados facilitan el enfoque específico de cada equipo.                                   |
| 🚦 Control de WIP         | Límites de trabajo en progreso aseguran capacidad equilibrada entre desarrollo y operaciones.                 |
| 🧩 Flexibilidad           | Priorización dinámica adaptada a necesidades cambiantes de infraestructura cloud.                             |
| 🤝 Colaboración asíncrona | Tarjetas compartidas entre tableros facilitan la transición desarrollo → operaciones sin ceremonias formales. |

## 🔁 Flujo de Trabajo Kanban

### Estructura de los Tableros

**Development Team Board**  
`Backlog` → `In Analysis` (WIP 3) → `In Development` (WIP 4) → `Done`

**Operations Team Board**  
`Backlog` (WIP 2) → `In Configuration` → `Stable`

## 📊 Métricas Clave

- **Cycle Time Desarrollo:** Tiempo promedio desde "En Desarrollo" hasta "Listo para Despliegue"
- **Deployment Lead Time:** Tiempo desde "Listo para Despliegue" hasta "Estable"
- **Throughput Semanal:** Tareas completadas por equipo
- **WIP Overflow:** Porcentaje de veces que se exceden los límites WIP

## 📌 Conclusión

El modelo **Kanban Dual** provee la flexibilidad necesaria para gestionar paralelamente el desarrollo ágil de
microservicios y las operaciones cloud complejas. Los dos tableros especializados permiten:

- 🎯 Enfoque en dominios técnicos distintos
- 🔄 Flujo visualizado de desarrollo a producción
- 🚦 Control granular de capacidad por equipo
- 🤝 Coordinación mínima pero efectiva entre equipos

Esta adaptación de Kanban soporta perfectamente los requerimientos de **MicroTodoSuite**, combinando entrega continua
con estabilidad operacional en entornos cloud dinámicos.