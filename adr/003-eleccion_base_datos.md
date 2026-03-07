# ADR-003: Elección del Motor de Base de Datos

## Estado
Propuesta

## Contexto
El sistema de inventario debe gestionar un catálogo proyectado de **30,000 productos** y soportar la operación de **50 usuarios concurrentes**.
Los requisitos funcionales críticos son:
1.  **Integridad Transaccional (ACID):** El registro de entradas, salidas y el cálculo del stock actual (Kardex) debe ser exacto. No se tolera la "consistencia eventual" en los contadores de inventario.
2.  **Trazabilidad:** Se requiere un historial inmutable de movimientos.
3.  **Restricción de Tiempo:** El MVP debe entregarse en **1 semana**. El equipo tiene experiencia sólida en SQL relacional pero nula experiencia con tipos de datos documentales avanzados (JSONB) en PostgreSQL.

### Stack Tecnológico Definido (ADR-001 y ADR-002)
- **Backend:** .NET 10 (Entity Framework Core).
- **Arquitectura:** Monolito Modular + Clean Architecture.

### Opciones Consideradas

1.  **PostgreSQL (Modelo Relacional Puro)**
    *   *Descripción:* Uso de tablas estándar normalizadas (Productos, Categorías, Unidades).
    *   *Pros:* Compatibilidad nativa y perfecta con Entity Framework Core, curva de aprendizaje nula para el equipo, garantías ACID fuertes.
    *   *Contras:* Menos flexibilidad si los productos tienen atributos muy dinámicos (ej: una TV tiene atributos muy diferentes a una Camiseta).

2.  **PostgreSQL (Híbrido con JSONB)**
    *   *Descripción:* Usar una columna `Attributes` tipo `JSONB` para manejar la variabilidad de los productos.
    *   *Pros:* Flexibilidad extrema sin migraciones de esquema continuas.
    *   *Contras:* **Riesgo alto de retraso.** El equipo no tiene experiencia consultando o indexando JSONB eficientemente. Aprender esto durante la semana de desarrollo del MVP pone en riesgo la entrega.

3.  **NoSQL (MongoDB)**
    *   *Descripción:* Base de datos orientada a documentos.
    *   *Contras:* Las transacciones ACID multi-documento son más complejas de gestionar y menos robustas que en SQL. Para un Kardex financiero/físico, el modelo relacional es superior por seguridad.

4.  **SQL Server**
    *   *Descripción:* Opción nativa de Microsoft.
    *   *Contras:* Costos de licenciamiento en producción (aunque existe Developer/Express) y PostgreSQL ofrece prestaciones similares o superiores siendo Open Source.

## Decisión
Se ha decidido utilizar **PostgreSQL** con un **Modelo Relacional Estándar**.

### Justificación
1.  **Seguridad Transaccional (Kardex):** PostgreSQL ofrece uno de los motores transaccionales más robustos del mercado. Para el Kardex (donde *Entrada - Salida = Saldo*), la integridad referencial y las transacciones atómicas son innegociables.
2.  **Mitigación de Riesgos (JSONB):** Aunque `JSONB` es técnicamente superior para atributos dinámicos, **no se usará en el MVP** debido a la curva de aprendizaje.
    *   *Estrategia:* Si necesitamos optimizar lecturas o manejar atributos variables, optaremos por **desnormalización intencional** en tablas relacionales o tablas clave-valor (`ProductAttributes`), que son técnicas ya dominadas por el equipo.
3.  **Escala Manejable:** 30,000 registros es una carga trivial para PostgreSQL. No se requiere la complejidad de NoSQL o JSONB para obtener rendimiento con este volumen de datos.
4.  **Integración con .NET:** Entity Framework Core (el ORM de .NET) tiene el mejor soporte para bases relacionales. Mapear objetos a tablas es automático y rápido, acelerando el desarrollo.

## Consecuencias

### Positivas
*   **Velocidad de Desarrollo:** El equipo puede empezar a modelar entidades (`DbSet<Product>`) inmediatamente sin aprender sintaxis nueva.
*   **Integridad de Datos:** Las *Foreign Keys* garantizan que no existan "movimientos huérfanos" sin producto asociado.
*   **Reporting:** SQL estándar facilita la creación de reportes complejos (JOINs) para el módulo de reportes.

### Negativas / Riesgos
*   **Rigidez del Esquema:** Si los productos tienen características muy dispares, podríamos terminar con una tabla `Productos` con muchas columnas nulas (Sparse Columns).
    *   *Mitigación:* Se acepta esta deuda técnica para el MVP. Se puede refactorizar a JSONB en una Fase 2 cuando el sistema esté estable y haya tiempo para investigar.
