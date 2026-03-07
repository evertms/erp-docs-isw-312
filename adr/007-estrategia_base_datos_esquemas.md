# ADR-007: Estrategia de Base de Datos y Esquemas Temporales para Monolito Modular SaaS

## Estado
Aceptada

## Contexto
El sistema ERP es un SaaS multi-tenant que está siendo desarrollado bajo una arquitectura de **Monolito Modular**. Actualmente, tenemos el módulo de **Inventario** y estamos por iniciar el módulo de **Ventas**.

Necesitamos definir una estrategia clara de persistencia para responder a dos preguntas principales:
1.  ¿Cuántas bases de datos físicas vamos a utilizar?
2.  ¿Cuántos esquemas se implementarán dentro de la base de datos y cómo se manejará la interacción entre ellos (datos compartidos vs. aislados)?

### Desafíos
*   **Complejidad Operativa:** Manejar múltiples bases de datos (una por módulo) obligaría a implementar transacciones distribuidas (Outbox Pattern, 2PC, Eventual Consistency), lo cual es prematuro y costoso para la fase actual.
*   **Limpieza de Código (Clean Code):** Queremos mantener los módulos desacoplados para que no terminen siendo un "espagueti", lo que a menudo pasa al usar un solo esquema `public`.
*   **Multi-tenancy y Datos Compartidos:** Al ser un SaaS, existe información global (ej. Catálogo de Inquilinos/Empresas, Usuarios base) que, estrictamente hablando, no pertenece a un solo módulo como Inventario o Ventas.

### Opciones Consideradas

1.  **Una Base de Datos Física por Módulo (Estilo Microservicios)**
    *   *Pros:* Aislamiento perfecto. Inposible hacer un JOIN accidental entre Módulo Ventas y Módulo Inventario.
    *   *Contras:* Altísima complejidad operativa. Requiere múltiples cadenas de conexión, sincronización de datos asíncrona (RabbitMQ/Kafka) para mantener consistencia. Inviable para el tamaño actual del equipo.

2.  **Una Base de Datos, Todos en el esquema `public`**
    *   *Pros:* Extremadamente simple. Una sola cadena de conexión.
    *   *Contras:* Los desarrolladores tenderán a hacer JOINs directos entre tablas de Ventas e Inventario, creando un alto acoplamiento a nivel de base de datos. Rompe las reglas del monolito modular.

3.  **Una Base de Datos, Esquemas Separados por Módulo (Schema-per-Module)**
    *   *Pros:* Simplicidad de despliegue (1 base de datos, 1 cadena de conexión, 1 respaldo), pero con separación lógica a nivel de DB. Permite aislar tablas de cada bounded context.
    *   *Contras:* Requiere disciplina en el código para no saltarse los límites de los esquemas haciendo consultas cross-schema no deseadas.

## Decisión
Se ha decidido adoptar la opción **Schema-per-Module** bajo las siguientes reglas:

1.  **Una Única Base de Datos:** Todo el sistema ERP vivirá dentro de una única base de datos PostgreSQL, administrada por una sola cadena de conexión manejada a nivel de la infraestructura del backend.
2.  **Un Esquema por Módulo Operativo:** Cada módulo funcional creará sus tablas en su propio esquema:
    *   Módulo de Inventario -> Esquema `inventory` (ej. `inventory.products`, `inventory.movements`).
    *   Módulo de Ventas -> Esquema `sales` (ej. `sales.orders`, `sales.order_lines`).
3.  **Módulo y Esquema Compartido (`core` o `identity`):** Se creará un módulo especial transversal (con su esquema `core` o `tenant`) única y exclusivamente para los datos que son el centro del SaaS y que habilitan el Multi-tenant. 
    *   Aquí vivirán tablas como `core.tenants` (empresas) y `core.users`.
4.  **REGLA DE ORO DE DESACOPLAMIENTO (Sin JOINs Inter-módulos):** Para ganar experiencia en sistemas limpios y prepararnos para posibles microservicios futuros, **queda estrictamente prohibido crear Llaves Foráneas (Foreign Keys - FK) entre los esquemas de módulos operativos**.
    *   *Ejemplo Práctico:* Una `Venta` (`sales.orders`) no tendrá una *Foreign Key* hacia el `Producto` (`inventory.products`). En su lugar, guardará `ProductId` como un simple `Guid` o `Int`. 
    *   Si Ventas necesita información del Producto, no hará un `JOIN`, sino que usará la Interfaz interna del módulo de Inventario a nivel de código C# (.NET) para consultar los datos, o mantendrá una réplica con datos parciales (si fuese estrictamente necesario por performance).

## Consecuencias

### Positivas
*   **Simplicidad Mantenida:** Los backups son atómicos, la infraestructura es simple (un solo servidor de DB) y solo hay que lidiar con una `ConnectionString`.
*   **Límites Arquitectónicos Claros:** Al obligarnos a no usar FKs entre distintos esquemas, garantizamos que cada módulo sea "dueño" exclusivo de su persistencia. Si mañana quisiéramos extraer el módulo de Ventas a su propio microservicio real, no tendríamos que romper relaciones en la base de datos.
*   **Experiencia Relevante:** El equipo aprende a modelar datos asilados y compartir información mediante contratos e interfaces en memoria (In-Process Calls), en vez de depender mágicamente de los JOINs de SQL.

### Negativas / Riesgos
*   **Pérdida de Integridad Referencial Estricta:** Como no hay FKs entre módulos operativos, si se elimina forzosamente un producto en el módulo `inventory`, podrían quedar "huérfanos" en `sales`.
    *   *Mitigación:* Dependemos al 100% de la lógica de negocio (Application Layer / Domain Layer) para evitar eliminaciones físicas indebidas (usar *Soft Delete*) y validar la existencia de las entidades cuando se comunican entre módulos.
*   **Consultas más Complejas en Reportes:** Generar un reporte de ventas con nombres de productos requerirá unir datos en memoria (C#) o crear modelos de lectura dedicados (CQRS). No se puede hacer un simple `SELECT * FROM sales JOIN inventory`.

## Nota Final
Manejar "esquemas" es solo una convención de agrupación en PostgreSQL, pero impone una barrera psicológica útil para los desarrolladores. La verdadera arquitectura limpia en este escenario provendrá de aplicar disciplina a la regla de "Cero Foreign Keys entre Módulos Operativos", forzando la encapsulación del monolito modular.
