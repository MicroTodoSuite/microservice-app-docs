# 🧠 **MicroTodoSuite** – Metodología Ágil

Este documento describe la metodología ágil adoptada para la gestión del proyecto **MicroTodoSuite**, explicando por qué **Scrum** es la opción más adecuada considerando las características del equipo, los objetivos del proyecto y el enfoque de despliegue continuo en la nube.

## 📌 Características del Proyecto

| Elemento                    | Descripción                                                                                |
| --------------------------- | ------------------------------------------------------------------------------------------ |
| 🎯 Objetivo principal       | Despliegue automatizado de infraestructura para una aplicación de microservicios en Azure. |
| 🧑‍🤝‍🧑 Estructura del equipo    | 2 equipos funcionales (Desarrollo y Operaciones), cada uno con 3 personas.                 |
| 🔁 Cadencia de entregas     | Iteraciones semanales con entregas frecuentes a ambientes reales.                          |
| 🛠️ Cultura DevOps           | Automatización de CI/CD, monitoreo con Prometheus y Grafana, y IaC con Terraform.          |
| ☁️ Infraestructura dinámica | Uso de recursos en la nube gestionados como código y desacoplados por servicio.            |

## ✅ Metodología Elegida: **Scrum**

Scrum es una metodología ágil iterativa e incremental que proporciona un marco estructurado para organizar el trabajo y mejorar progresivamente tanto los procesos como los resultados del equipo.

### ¿Por qué Scrum?

| Factor                   | Justificación en MicroTodoSuite                                                                                        |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| 🧭 Iteraciones claras    | La división del trabajo en Sprints semanales permite planificar, ejecutar y revisar con enfoque claro.                 |
| 👥 Roles definidos       | Scrum establece responsabilidades claras (Scrum Master, Product Owner y Developers), algo crucial en equipos pequeños. |
| 🔄 Mejora continua       | Las retrospectivas periódicas permiten ajustar procesos rápidamente.                                                   |
| 📋 Priorización efectiva | El Product Backlog permite gestionar dinámicamente tareas y prioridades tanto para desarrollo como operaciones.        |
| 🧪 Entrega incremental   | Scrum facilita la integración progresiva de componentes, útil en proyectos de microservicios con despliegue continuo.  |
| 🤝 Integración Dev + Ops | Las ceremonias Scrum (Daily, Planning, Review) fortalecen la colaboración entre equipos técnicos.                      |

## 🔁 Ciclo Scrum Propuesto

1. **Sprint Planning (semanal):** Definición de objetivos del Sprint, selección de tareas desde el Product Backlog.
2. **Daily Scrum (15 min):** Reunión breve diaria para sincronizar avances, identificar bloqueos y coordinar entre equipos.
3. **Sprint Review:** Demostración del trabajo completado al final de cada Sprint.
4. **Sprint Retrospective:** Espacio para identificar mejoras en procesos, comunicación y herramientas.
5. **Refinamiento continuo del Backlog:** Priorización colaborativa de tareas entre Sprints.

## 📌 Conclusión

Scrum proporciona la estructura necesaria para mantener la **coherencia, colaboración y mejora continua** en un equipo pequeño pero técnico. En el contexto de **MicroTodoSuite**, donde conviven tareas de desarrollo y despliegue de infraestructura, Scrum permite gestionar eficazmente ambos flujos, asegurar entregas frecuentes y mantener una cultura de retroalimentación constante.
