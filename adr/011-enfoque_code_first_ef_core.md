# ADR-011: Estrategia de Persistencia y Modelado (Code-First Entity Framework Core)

## Estado
Aceptada

## Contexto
Con la decisión de adoptar un modelo de "Schema-per-Module" (ADR-007) y una arquitectura basada en Domain-Driven Design y Clean Architecture, es imperativo establecer cómo las entidades de negocio (C#) serán mapeadas al almacenamiento relacional (PostgreSQL).

Existen dos escuelas principales en el entorno de Entity Framework Core para generar la base de datos: *Database-First* (hacer scripts SQL o diagramas físicos y pedirle al ORM que genere las clases C# en base a las tablas) y *Code-First* (crear el modelo de objetos ricos en C# primero, y dejar que el ORM genere los scripts DDL/Migraciones).

## Decisión
Se establece como estándar obligatorio en todo el proyecto el uso del enfoque **Code-First a través de Entity Framework Core**.

### Reglas de Implementación
1.  **Dominio Puro:** Las clases del modelo de dominio se ubicarán en los proyectos `Domain` de cada módulo y **completamente libres de atributos ligados a datos** (`[Table]`, `[Column]`, `[Required]`). Solo contendrán lógica de negocio y encapsulación pura.
2.  **Fluent API para Configuración:** El mapeo relacional dictando tipos, esquemas, longitudes y relaciones (sin saltarse la regla de no FKs inter-módulo de ADR-007) se hará exclusivamente con el `Fluent API` dentro de los archivos de configuración en las capas de `Infrastructure` correspondientes.
3.  **Migraciones Centralizadas:** Los archivos de migración generados por .NET residirán en el proyecto anfitrión de la Web API o en una librería de diseño dedicada, persistiendo en el control de versiones (Git) como única fuente de verdad evolutiva del esquema de la base de datos.
4.  **Seed Data en Código:** Los datos requeridos tempranamente para el sistema (ej. Roles base, esquema `Core`) y registros semilla, se cargarán en la sobrecarga `OnModelCreating` en el código de infraestructura, descartando scripts automáticos `.sql` (como el clásico `init.sql`).

## Consecuencias

### Positivas
*   **Fidelidad a Clean Architecture:** Code-First es el único método que permite que la lógica del entorno de negocio determine cómo es el almacén físico de la data, en contraposición de Database-First donde las peculiaridades de SQL infectan el modelo de clases, generando "Modelos de Dominio Anémicos".
*   **Versionamiento e Integración:** Las migraciones C# se pueden evaluar en de los Pull Requests. Cada vez que el equipo baja código nuevo, al aplicar las migraciones las bases de datos de todos quedan consistentes, permitiendo un gran soporte multi-developer.
*   **Multi-schema Nativo:** `Fluent API` provee maneras muy elegantes de aplicar las convenciones de esquemas (ej. `.HasDefaultSchema("inventory")`) con una línea de código directamente desde el `DbContext`.

### Negativas / Riesgos
*   **Curva de Aprendizaje de Migraciones:** Entity Framework impone un proceso rígido. Si alguien modifica la base de datos a mano (por un parche de urgencia), provocará problemas graves cuando otra persona intente correr `dotnet ef database update`. 
    * *Mitigación:* Se impone la prohibición absoluta de alterar esquemas relacionales fuera del pipeline oficial de Entity Framework.
*   **Limitaciones en características muy específicas del Motor SQL:** Configuraciones a muy bajo nivel como mapeos compuestos muy densos o extensiones en PostgreSQL (ej. triggers manuales), pueden requerir redactar `.Sql("...")` "crudo" en las migraciones C#, lo cual lo hace más tedioso que tenerlo en un script dedicado inicial.
