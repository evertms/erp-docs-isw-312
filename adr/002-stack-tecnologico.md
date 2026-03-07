# ADR-002: Backend (.NET 10) & Frontend (React/TypeScript)

## Estado
Propuesta

## Contexto
El sistema de inventario debe ser robusto, rápido de desarrollar y fácil de mantener.
1.  **Backend:** Se requiere una API sólida que soporte Clean Architecture y fácil integración con bases de datos relacionales.
2.  **Frontend:** Se requiere una interfaz rica e interactiva para la gestión de productos y dashboard de inventario en tiempo real.
3.  **Factor Humano:** El desarrollador (yo) tiene experiencia previa en .NET y cuenta con una plantilla de React que acelera el desarrollo.

### Opciones Consideradas (Backend)

1.  **.NET 10 (C#)**
    *   *Pros:* Tipado estático fuerte, ecosistema maduro, excelente rendimiento, inyección de dependencias nativa, familiaridad del equipo.
    *   *Estructura:* Permite separar capas físicamente mediante **Class Libraries**, lo que fuerza el cumplimiento de la arquitectura (el proyecto de Dominio *físicamente* no puede ver al de Infraestructura si no se referencia).
    *   *Contras:* Verbose en comparación con Node.js o Python para scripts simples, pero ventajoso para sistemas empresariales.

2.  **Node.js (NestJS/Express)**
    *   *Pros:* Mismo lenguaje en frontend y backend (JS/TS), enorme comunidad.
    *   *Estructura:* NestJS usa módulos (decoradores) para organizar el código.
    *   *Contras:* La arquitectura limpia se basa en "disciplina" (carpetas), no en restricciones duras de compilación como en .NET. Es más fácil romper las reglas de dependencia accidentalmente.
    
3.  **Python (FastAPI/Django)**
    *   *Pros:* Desarrollo muy rápido.
    *   *Contras:* Tipado dinámico (incluso con TypeHints) puede llevar a errores en tiempo de ejecución en sistemas grandes. Menor rendimiento bruto que .NET para lógica compleja.

### Opciones Consideradas (Frontend)

1.  **React (TypeScript) con Vite**
    *   *Pros:* Estándar de la industria, gran ecosistema de componentes, plantilla ya disponible (ahorro de tiempo).
    *   *Contras:* Requiere configuración de estado global (Zustand/Context).

2.  **Angular**
    *   *Pros:* Estructura rígida ideal para empresas.
    *   *Contras:* Curva de aprendizaje alta, verboso.

3.  **Blazor (Server/WASM)**
    *   *Pros:* Todo en C#.
    *   *Contras:* Menor ecosistema de librerías UI comparado con React, interactividad en WASM tiene tiempo de carga inicial alto.

## Decisión
Se utilizará la pila tecnológica: **Backend en .NET 10** y **Frontend con React (TypeScript)**.

### Justificación
1.  **Eficiencia y Conocimiento Previo:** Al usar tecnologías conocidas por mí (.NET) y recursos existentes (Plantilla React), eliminamos la curva de aprendizaje y nos enfocamos 100% en la lógica de negocio del inventario.
2.  **Robustez de .NET para Clean Architecture:** La capacidad de .NET de dividir la solución en múltiples proyectos (`.csproj`) permite una implementación **estricta** de la Arquitectura Limpia.
    *   A diferencia de Node/Python donde separar capas suele ser solo "una convención de carpetas", en .NET podemos crear una *Class Library* para el **Dominio** que no tenga referencias a nada externo. Si intentas usar `EntityFramework` o `HTTP` en el Dominio, el código **ni siquiera compila**. Esto es una garantía de calidad arquitectónica invaluable.
3.  **React + TypeScript:** Provee la interactividad necesaria para una SPA (Single Page Application) moderna, y el uso de TypeScript asegura que los contratos de datos (DTOs) del backend se respeten en el frontend.

## Estrategia de Implementación (.NET)
Para implementar Clean Architecture correctamente en .NET, la solución se estructurará típicamente así:

*   **`Core/Domain` (Class Library):** Entidades, Interfaces del Repositorio, Excepciones de Dominio. *Sin dependencias.*
*   **`Core/Application` (Class Library):** Casos de Uso (Services/Handlers), DTOs, Validaciones. *Depende de Domain.*
*   **`Infrastructure` (Class Library):** Implementación de Repositorios (EF Core), Servicios Externos (Email/S3). *Depende de Application y Domain.*
*   **`API` (.NET Web API):** Controladores, Inyección de Dependencias (Composition Root). *Depende de Application e Infrastructure.*

Esta estructura física impide las referencias circulares y fuerza el flujo de dependencia en una sola dirección (hacia adentro).

## Consecuencias
### Positivas
*   **Velocidad:** Aprovechamiento inmediato del *know-how* del equipo.
*   **Calidad:** Tipado estático total (C# en back, TS en front) reduce drásticamente los *bugs* de tipo "undefined is not a function".
*   **Mantenibilidad:** La estructura de proyectos de .NET asegura que la deuda técnica arquitectónica sea difícil de introducir accidentalmente.

### Negativas / Riesgos
*   **Duplicidad de Modelos:** Será necesario mantener sincronizados los DTOs de C# con las interfaces de TypeScript (se puede mitigar con herramientas de generación de código como NSwag o TypeGen si el tiempo lo permite).
