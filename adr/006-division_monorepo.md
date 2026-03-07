# ADR-006: División del Monorepo en Repositorios Independientes

## Estado
Aceptada

## Contexto
A medida que el ecosistema del proyecto crece, mantener el frontend, los módulos del backend y la documentación en un único repositorio (monorepo) presenta desafíos. Los distintos componentes tienen ciclos de vida de desarrollo, necesidades de infraestructura y dependencias muy diferentes:
1.  **Ciclos de Despliegue CI/CD Diferentes:** El frontend (React), el backend (.NET) y la documentación (Docs) requieren pipelines de construcción y despliegue independientes.
2.  **Evolución e Iteración:** La documentación puede actualizarse frecuentemente sin necesidad de tocar el código de la aplicación. Del mismo modo, el frontend puede requerir ajustes visuales sin que la API del backend cambie.
3.  **Gestión de Permisos:** Separar los repositorios permite una gestión de acceso más granular para diferentes perfiles del equipo (ej. redactores técnicos para docs, desarrolladores UI para frontend).

### Opciones Consideradas

1.  **Mantener un Monorepo Estricto**
    *   *Pros:* Única fuente de la verdad para todo el proyecto. Cambios sincronizados entre frontend y backend en un solo commit.
    *   *Contras:* Los pipelines de CI/CD se vuelven complejos (necesitan detectar qué carpetas cambiaron para disparar los builds correspondientes). Al descargar el repositorio, se descarga mucho código innecesario dependiendo del rol del desarrollador.

2.  **Dividir en Repositorios Diferentes (Polyrepo)**
    *   *Pros:* Historial de Git más limpio por tecnología. Pipelines de CI/CD directos y simples (un repositorio = un pipeline). Separación clara de responsabilidades.
    *   *Contras:* Los cambios "full-stack" (ej. agregar un campo a la base de datos y mostrarlo en la UI) requieren coordinar pull requests en múltiples repositorios.

## Decisión
Se ha tomado de decisión (ACEPTADA) de **dividir el monorepo en repositorios diferentes** correspondientes a las piezas principales del sistema:
*   **Frontend** (Aplicación en React)
*   **Documentación** (Archivos Markdown, diagramas, etc.)
*   **Módulos de Backend** (Servicios y API en .NET)

## Consecuencias

### Positivas
*   **CI/CD Simplificado y Rápido:** Cada repositorio se construye y despliega solo cuando su código cambia, reduciendo significativamente los tiempos de build.
*   **Autonomía de Desarrollo:** Los equipos o desarrolladores pueden trabajar en el frontend o editar la documentación sin preocuparse por compilar o interferir con el entorno del backend.
*   **Entornos Ligeros:** Los desarrolladores front-end no necesitan instalar dependencias complejas de .NET si están trabajando con datos mock o un entorno de staging.

### Negativas / Riesgos
*   **Gestión de Contratos API:** El riesgo de que el frontend intente consumir una versión de la API que no ha sido desplegada o viceversa.
    *   *Mitigación:* Depender de una buena documentación de la API (ej. Swagger/OpenAPI) y establecer una política clara de versiones y retrocompatibilidad en el backend.
*   **Fragmentación del Seguimiento de Issues:** Tener issues repartidos en varios tableros.
    *   *Mitigación:* Utilizar herramientas de gestión de proyectos que permitan unificar milestones y épicas a través de múltiples repositorios (ej. proyectos de GitHub a nivel de organización).

## Nota Final
Esta separación organizará mejor el código fuente a largo plazo y preparará el ecosistema de la aplicación para un desarrollo y escalado más ágil y descentralizado.
