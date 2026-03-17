# ADR-001: Arquitectura Base (Monolito Modular vs Microservicios)

## Estado
Aceptada

## Contexto
El proyecto consiste en el desarrollo de un **Sistema de Control de Inventario** con los siguientes objetivos y restricciones críticas:

1.  **Restricción de Tiempo y Recursos:**
    *   **Equipo:** 1 desarrollador (Full-Stack).
    *   **Deadline:** 1 semana real para el MVP funcional.
    *   **Objetivo Inmediato:** Lograr visibilidad de stock real, registrar movimientos (entradas/salidas) y asegurar trazabilidad (Kardex).

2.  **Visión a Largo Plazo:**
    *   Evolucionar hacia un modelo **SaaS Multi-tenant** donde múltiples empresas puedan gestionar sus inventarios de forma aislada.

3.  **Dilema Arquitectónico:**
    *   Existe la tentación de diseñar una arquitectura altamente escalable basada en microservicios pensando en el futuro SaaS.
    *   Sin embargo, la complejidad inherente de los sistemas distribuidos (microservicios) podría impedir cumplir con la entrega funcional de la primera semana.

### Opciones Consideradas

1.  **Arquitectura de Microservicios:**
    *   Servicios separados para `Inventario`, `Auth`, `Reportes`, `Notificaciones`.
    *   Despliegue independiente, bases de datos separadas.
    *   *Problema:* Introduce "latencia de red", "transacciones distribuidas" (Saga pattern), y complejidad de orquestación (Kubernetes/Docker Compose complejo) que es inmanejable para 1 persona en 1 semana.

2.  **Monolito Tradicional (Spaghetti Code):**
    *   Todo el código en una sola estructura sin límites claros.
    *   *Problema:* Difícil de mantener a largo plazo y complicado de migrar a SaaS posteriormente.

3.  **Monolito Modular:**
    *   Una única unidad de despliegue (un solo backend).
    *   Separación lógica estricta por módulos de negocio (`Modules/Inventory`, `Modules/Identity`, `Modules/Reporting`).
    *   Comunicación interna vía llamadas a funciones/interfaces, no red.

## Decisión
Se ha decidido implementar un **Monolito Modular**.

### Justificación
1.  **Velocidad de Desarrollo (Time-to-Market):**
    *   Para cumplir con el *deadline* de 1 semana, necesitamos cero sobrecarga de infraestructura. Configurar comunicación entre microservicios, service discovery y manejo de fallos de red consumiría el 50% del tiempo disponible.
    *   Un monolito permite depurar, probar y desplegar todo el sistema en un solo paso.

2.  **Simplicidad vs. Complejidad innecesaria:**
    *   La "escalabilidad teórica" de los microservicios es irrelevante si no hay un producto funcional ("YAGNI - You Aren't Gonna Need It Yet").
    *   El cuello de botella de un sistema de inventario suele ser la base de datos (consistencia ACID para stock), no el procesamiento CPU. Un monolito gestiona transacciones ACID de forma nativa y simple.

3.  **Preparación para SaaS (Multi-tenancy):**
    *   Implementar multi-tenancy (aislamiento de datos por cliente) es más sencillo en un monolito: se puede aplicar una estrategia de columna `tenant_id` en las tablas o esquemas por tenant, gestionado centralmente en el middleware de base de datos.
    *   En microservicios, propagar el contexto del tenant y asegurar consistencia entre servicios distribuidos añadiría una complejidad exponencial innecesaria en esta etapa.

4.  **Refactorización Futura:**
    *   Al usar un diseño **modular** (con límites claros entre dominios), si en el futuro el módulo de `Reportes` requiere escalar masivamente, será fácil extraerlo a un microservicio independiente. *Es más fácil romper un monolito bien hecho que arreglar microservicios mal hechos.*

## Nota: Relación con Clean Architecture
Es común confundir **Monolito Modular** con **Clean Architecture**, pero son conceptos complementarios que trabajan juntos:

1.  **Monolito Modular (Nivel Macro):** Define cómo organizamos toda la aplicación (por Módulos de Negocio: *Inventario*, *Ventas*, *Usuarios*).
2.  **Clean Architecture (Nivel Micro):** Define cómo organizamos el código **dentro** de cada módulo (o de la app en general) para separar la lógica de negocio de la infraestructura.

**Estrategia Elegida:**
Implementaremos un Monolito Modular donde **cada módulo** (o el núcleo compartido) siga principios de Clean Architecture. Esto significa que dentro del módulo de *Inventario*, la "Lógica de Negocio" (Domain) no dependerá de la "Base de Datos" (Infrastructure), ni de la "API" (Presentation).

## Consecuencias

### Positivas
*   **Entrega Rápida:** El MVP podrá estar listo en 1 semana.
*   **Infraestructura Simple:** Solo se requiere 1 servidor/contenedor y 1 base de datos.
*   **Consistencia de Datos:** Garantizar que el Kardex cuadre es trivial gracias a las transacciones de base de datos locales (ACID).
*   **Testing:** Las pruebas de integración son sencillas y rápidas de ejecutar.

### Negativas / Riesgos
*   **Disciplina de Código:** Requiere rigor para no acoplar los módulos (ej. no importar clases de `Inventario` directamente en `Auth`). Si no se respeta la modularidad, se degradará a un "Monolito de Barro".
*   **Escalado:** En el futuro SaaS, si un solo cliente consume muchos recursos, podría afectar a los demás (Vecino Ruidoso), aunque esto se mitiga escalando verticalmente el servidor inicialmente.
