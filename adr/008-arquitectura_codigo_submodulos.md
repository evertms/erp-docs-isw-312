# ADR-008: Arquitectura de Código Backend para Módulos y Submódulos (Ventas y POS)

## Estado
Aceptada

## Contexto
El sistema ERP se está construyendo bajo un enfoque donde **cada módulo operativo es un repositorio separado**. Para priorizar la velocidad de desarrollo y tener un MVP rápido, el backend de cada módulo consta de un **único proyecto de .NET (Web API)**. En lugar de dividir las capas en múltiples ensamblados (DLLs), se utiliza una separación física por directorios: `Domain`, `Application` e `Infrastructure`.

Específicamente, estamos por iniciar el módulo de **Ventas** y prevemos la necesidad de un submódulo para **Punto de Venta (POS)**. Un POS comparte lógica central con las ventas tradicionales (órdenes, productos, clientes), pero tiene características muy específicas (cajas registradoras, cierres de turno, sincronización).

El desafío es: ¿Cómo estructurar los submódulos (como Backoffice y POS) dentro de este único proyecto Web.API de Ventas, logrando bajo acoplamiento sin complicar la estructura inicial del MVP?

### Opciones Consideradas

1.  **Carpetas Clásicas por Capa y Concepto Mezclado**
    *   *Pros:* Estructura estándar muy conocida.
    *   *Contras:* Los *handlers* del Backoffice y los del POS convivirían en la misma carpeta `Application/Commands`. Rápidamente se volvería un código espagueti difícil de mantener y separar a futuro.

2.  **Múltiples Proyectos (Clean Architecture Tradicional)**
    *   *Pros:* Separación estricta de capas.
    *   *Contras:* Sobrecarga (*overhead*) para un MVP. Va en contra de la directiva de mantener un solo proyecto Web API.

3.  **Arquitectura de un Proyecto con Vertical Slices Internos por Submódulo**
    *   *Pros:* Cumple con la premisa de usar un solo proyecto Web.API. Mantiene el Modelo de Dominio puramente unificado, pero los Casos de Uso (Application) y los Controladores se separan radicalmente por "submódulo funcional" usando carpetas (Vertical Slices).
    *   *Contras:* No hay barreras físicas a nivel compilador (solo de carpetas) que impidan llamar a infraestructura desde aplicación, asumiendo el riesgo temporal por velocidad.

## Decisión
Se ha decidido adoptar la arquitectura de **Un Proyecto (.NET Web.API) con Submódulos organizados en Vertical Slices dentro de la capa de Aplicación**. 

Se espera que la estructura de carpetas en el código C# para Ventas sea la siguiente:

```text
src/
└── Sales.Api/                     # Único proyecto .NET (Web.API)
    ├── Domain/                    # NÚCLEO COMPARTIDO
    │   ├── Entities/              # Ej. Order, OrderLine, Customer
    │   └── Exceptions/
    ├── Infrastructure/            # PERSISTENCIA COMPARTIDA
    │   ├── DbContext/
    │   └── Repositories/
    └── Application/               # APLICACIÓN SUBDIVIDIDA (Vertical Slices)
        ├── Submodules/            # Agrupación por submódulos funcionales
        │   ├── Backoffice/        # Submódulo 1: Facturación ERP
        │   │   ├── Dtos/          # Dtos específicos de B2B
        │   │   ├── Commands/      # Ej. CreateInvoiceHandler
        │   │   └── Queries/       # Ej. GetMonthlySalesHandler
        │   └── POS/               # Submódulo 2: Punto de Venta
        │       ├── Dtos/          # Dtos ligeros para POS
        │       ├── Commands/      # Ej. OpenCashRegisterHandler
        │       └── Queries/       # Ej. GetActiveSessionHandler
        └── Core/                  # Lógica de aplicación compartida (si existe)
```

### Reglas de Arquitectura
1.  **Dominio Único Compartido:** `Domain` contendrá el núcleo de negocio puro que afecta tanto al flujo de POS como al Backoffice corporativo.
2.  **Casos de Uso Aislados (Vertical Slices):** Dentro de `Application/Submodules/` cada carpeta representa una porción aislada. Jamás un Command Handler del POS debe llamar o depender de un Controller/Handler de Backoffice.
3.  **Riesgo Asumido (Un Solo Proyecto):** Se asume temporalmente el riesgo de que la falta de proyectos (`.csproj`) separados permita referencias circulares o saltos de arquitectura. Esto se controlará en Code Reviews.

## Consecuencias

### Positivas
*   **Velocidad MVP:** No hay sobrecarga de configuración de enlaces entre proyectos. Los tiempos de compilación y fricción al momento de agregar una nueva feature son nulos.
*   **Aislamiento Funcional:** Si se necesita extender la funcionalidad de la pantalla del POS, el desarrollador sabe que solo debe tocar código dentro de `Application/Submodules/POS`, sin miedo a romper el flujo de creación de facturas del ERP normal.
*   **Camino Trazado a Microservicios:** Pese a ser un solo proyecto, como los submódulos están encapsulados en "Slices", extraer el POS a su propia API en el futuro requerirá solo llevarse esa carpeta y una copia del modelo de dominio.

### Negativas / Riesgos
*   **Acoplamiento Accidental:** Al estar todo en la misma `Web.API`, es fácil (técnicamente) que un dev inyecte un Repositorio específico del POS dentro del Backoffice, o peor, acceder directamente a EF Core desde el controlador.
    *   *Mitigación:* Dependemos cien por ciento del control de calidad en PRs (Pull Requests) y disciplina del equipo para respetar los "feature folders".

## Nota Final
Esta estructura prioriza sacar un Minimum Viable Product (MVP) rápido al simplificar el manejo de dependencias a un solo proyecto, pero aplica Vertical Slices para evitar que el crecimiento esperado de los submódulos (como POS) ensucie la mantenibilidad de la aplicación a mediano plazo.
