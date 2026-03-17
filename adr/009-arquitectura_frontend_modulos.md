# ADR-009: Arquitectura de Frontend para Múltiples Módulos Backend

## Estado
Aceptada

## Contexto
El backend del sistema ERP SaaS ha sido diseño para operar de manera desacoplada: cada módulo operativo (como Inventario, Ventas, etc.) reside en su propio repositorio y se despliega como una API independiente (ADR-006 y ADR-008). 

Sin embargo, desde la perspectiva del cliente, la experiencia de usuario (UX) debe ser completamente fluida y unificada. El usuario debe sentir que navega por una única aplicación (SPA) sin saltos, inicios de sesión múltiples o cambios de dominio al pasar de consultar un producto a crear una factura de venta.

El desafío consiste en decidir cómo estructurar el proyecto Frontend (React) para que pueda comunicarse y escalar con "x" cantidad de módulos backend diferentes, manteniendo la premisa de velocidad para el MVP y simplicidad operativa.

### Opciones Consideradas

1.  **Micro-Frontends (Module Federation / Webpack / Vite)**
    *   *Pros:* Escalabilidad extrema. Cada módulo backend tendría su respectivo sub-proyecto de frontend en el mismo repositorio del módulo. Permite despliegues totalmente independientes sin afectar el resto de la interfaz.
    *   *Contras:* Complejidad técnica altísima. Compartir estado global (como el usuario autenticado, el Tenant seleccionado o el carrito del POS) entre micro-frontends suele ser un dolor de cabeza enorme. Sobre-ingeniería (Overkill) total para un MVP.

2.  **Monorepo Frontend (Nx o Turborepo)**
    *   *Pros:* Permite crear paquetes como `@erp/ui-components`, `@erp/auth` y aplicaciones separadas estructuradas profesionalmente.
    *   *Contras:* Mayor curva de aprendizaje y configuración de herramientas.

3.  **Monolito SPA (Single Page Application) estructurado por Módulos**
    *   *Pros:* Simplicidad absoluta. Un solo repositorio, un solo pipeline de CI/CD. Layouts (menú lateral, cabeceras) y estado global (Context/Zustand) unificados y fáciles de manejar. Mayor velocidad de desarrollo inicial.
    *   *Contras:* A medida que se agreguen más y más módulos, el tamaño final del *bundle* de JavaScript crecerá, y el equipo de frontend podría empezar a pisarse los talones si no hay orden.

## Decisión
Se ha decidido adoptar la arquitectura de **Monolito Frontend (SPA clásico en React) estructurado por Feature Folders (Módulos Internos)**.

Esto significa que, mientras el Backend son varios repositorios e instancias/APIs, el Frontend será una única aplicación alojada en el repositorio establecido en el ADR-006.

### Reglas de Arquitectura e Implementación

1.  **Estructura de Carpetas basada en Características (Feature Folders):**
    El código en React se agrupará lógicamente según los módulos del negocio, evitando agrupar por tipos técnicos (no tener una carpeta global "components" con todo mezclado).
    ```text
    src/
    ├── core/                # Diseño base, Layout global, Autenticación, API base (Axios)
    ├── shared/              # Componentes de UI genéricos (Botones, Modales, Tablas)
    └── modules/
        ├── inventory/       # Pantallas, componentes y llamadas API exclusivas de Inventario
        └── sales/           # Pantallas, Submódulos (POS, Backoffice) propios de Ventas
    ```

2.  **Rutas con Lazy Loading (Code Splitting):**
    Para mitigar el problema de crecimiento del tamaño del archivo JS (bundle), se usará obligatoriamente *Lazy Loading* (ej. `React.lazy()` o utilidades de React Router). Cuando un usuario ingrese al sistema, solo descargará el código de Autenticación y el Módulo Principal. Si nunca entra a Ventas, nunca descargará el código JS del módulo `sales`.

3.  **Comunicación API a través de un Gateway Proxy:**
    Para no obligar al frontend a conocer y lidiar con 10 o 20 URLs diferentes (`http://api-inventario...`, `http://api-ventas...`), todas las llamadas de red se harán hacia un **único dominio** o punto de entrada en el servidor (Ej: `api.mi-erp.com`).
    *   Soporte en Infraestructura: Detrás de ese dominio habrá un Reverse Proxy (como Nginx, YARP o Traefik - abordado en ADR-005) que evaluará la ruta de la petición:
        *   Cualquier llamada frontend a `/api/inventory/*` es redirigida al puerto/servicio del Módulo 1.
        *   Cualquier llamada frontend a `/api/sales/*` es redirigida al puerto/servicio del Módulo 2.
    *   *Beneficio:* Esto soluciona problemas de CORS completamente y mantiene Axios configurado con una sola `baseURL`.

## Consecuencias

### Positivas
*   **Velocidad de Creación (Time-to-Market):** Mantener un solo frontend permite tener un UI unificado muy rápido. No hay que reinventar la rueda configurando el layout y el menú en múltiples proyectos.
*   **Estado Global Simple:** Compartir el `TenantId` o los datos del usuario logueado en un Monolito React es trivial usando React Context o librerías de estado estándar.

### Negativas / Riesgos
*   **Acoplamiento de Equipos:** Si en el futuro el equipo de Frontend crece a decenas de desarrolladores, el repositorio centralizado generará fricciones en los Pull Requests.
    *   *Mitigación:* Se establecerán reglas de "Code Ownership" por módulo (ej. un Dev solo aprueba PRs que toquen la carpeta de su módulo asignado).
*   **Riesgo de Acoplamiento de UI:** Un desarrollador podría intentar importar un componente muy específico de `modules/sales/POS_Register` dentro de la pantalla de `modules/inventory/warehouse`.
    *   *Mitigación:* Dependemos de revisiones de código estrictas para forzar que si algo se usa en más de un módulo, primero deba ser promovido y refactorizado a la carpeta local `/shared`.

## Nota Final
Empezar con un Monolito Frontend robusto y bien parcelado por carpetas es el camino recomendado por excelencia para proyectos MVP rentables. Se descartan temporalmente los Micro-Frontends para evitar sobrecostos operativos.
