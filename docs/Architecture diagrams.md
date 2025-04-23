# 🏗️ **MicroTodoSuite** – Justificación de Arquitectura en la Nube

Este documento explica y justifica las decisiones de diseño adoptadas en la arquitectura cloud para **MicroTodoSuite**, abordando tanto la propuesta inicial como la implementación final ajustada a requisitos de costos del cliente.

## 📌 Arquitecturas Propuestas

### Propuesta Inicial
![IMAGEN 1: Diagrama de la propuesta inicial](./assets/Initial%20Proposal%20Diagram.png)

### Solución Implementada
![IMAGEN 2: Diagrama de la solución implementada](./assets/Solution%20Diagram.png)

*La implementación final fue adaptada siguiendo sugerencias del cliente para optimizar costos mientras se mantiene la funcionalidad esencial.*

## 🛠️ Decisiones de Diseño Principales

### 📦 Azure Container Apps
- **Escalabilidad automática**: Configuración de `min_replicas` y `max_replicas` para ajuste dinámico según demanda
- **Aislamiento de servicios**: Cada microservicio en su propio contenedor para gestión independiente
- **Administración simplificada**: Delegación de aspectos complejos de infraestructura a la plataforma

### 🗄️ Azure Container Registry
- **Almacenamiento seguro**: Repositorio centralizado para imágenes de contenedores
- **Control de versiones**: Gestión eficiente de diferentes versiones de microservicios
- **Integración nativa**: Conexión fluida con otros servicios Azure

### 📝 Implementación con Terraform
- **Infraestructura como código**: Definición declarativa que garantiza despliegues consistentes
- **Automatización**: Reducción de errores manuales en el aprovisionamiento
- **Versionamiento**: Control de cambios en la infraestructura

### 👁️ Arquitectura de Observabilidad
- **Zipkin**: Rastreo distribuido entre microservicios
- **Prometheus**: Recolección centralizada de métricas
- **Grafana**: Dashboards visuales para monitoreo en tiempo real

### 💬 Comunicación entre Servicios
- **Comunicación sincrónica**: Para operaciones que requieren respuesta inmediata
- **Redis para comunicación asincrónica**: Desacoplamiento en el procesamiento de logs

## 🔄 Patrones de Arquitectura Implementados

### 🛡️ Patrones de Resiliencia
- **Retry Pattern**: Implementado en cada Container App para reintentar automáticamente operaciones fallidas ante problemas transitorios de red o servicios
- **Circuit Breaker Pattern**: Evita llamadas a servicios degradados, previniendo cascadas de fallos y permitiendo recuperación controlada

### ☁️ Patrones Adicionales de Nube
- **Tolerancia a Fallos**: Complementando Retry y Circuit Breaker, la arquitectura incorpora redundancia y aislamiento que fortalecen la resiliencia general del sistema
- **Monitoreo y Diagnóstico**: La implementación de Zipkin, Prometheus y Grafana proporciona visibilidad completa del sistema, facilitando la detección temprana de problemas y el análisis de rendimiento

## 📊 Conclusión

La arquitectura implementada para **MicroTodoSuite** representa una solución equilibrada que satisface los requisitos técnicos del proyecto mientras respeta las restricciones presupuestarias del cliente. El despliegue basado en Azure Container Apps con patrones de resiliencia incorporados garantiza una aplicación robusta y escalable, mientras que la infraestructura de monitoreo asegura la observabilidad necesaria para una operación confiable en producción.

Esta implementación aprovecha las mejores prácticas de arquitectura en la nube, priorizando la mantenibilidad, la escalabilidad y la confiabilidad del sistema.