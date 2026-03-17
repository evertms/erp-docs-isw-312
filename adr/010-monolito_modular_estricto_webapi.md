# ADR-010: Adopción del Patrón Monolito Modular Estricto (Única Web API)

## Estado
Aceptada

## Contexto
Durante el diseño inicial de los módulos operativos del ERP (Inventario, Ventas, etc.), se había considerado mantener los backend en repositorios separados y exponerlos como APIs independientes integradas por un API Gateway, o bien organizarlos con Vertical Slices dentro de una sola Web API pero separada solo por carpetas (ver ADR-006 y ADR-008).

A medida que se avanzó en el análisis de las integraciones inter-módulo (ej. descontar stock en el Módulo de Inventario al realizarse el cobro de un ticket en el Módulo de Ventas / PDV), se evidenció que una arquitectura física distribuida añadiría una complejidad técnica desproporcionada para la fase MVP del proyecto. Además, las transacciones distribuidas y la consistencia eventual generarían un sobrecosto operativo inasumible en este momento.

## Decisión
Se ha decidido **abandonar el despliegue de múltiples APIs y consolidar todo el backend del sistema en un único proyecto ejecutable (Una sola Web API)**. 

Para mantener la separación de responsabilidades y las ventajas de Clean Architecture dentro de este monolito, se adoptará un enfoque de **Monolito Modular Estricto**:

1.  **Única Web API de Entrada:** Existirá un solo `.csproj` de tipo Web API que servirá como host de toda la aplicación y punto de entrada HTTP.
2.  **Librerías de Clases por Módulo y Capa:** Cada módulo funcional (Core, Inventario, Ventas) estará compuesto por sus propios proyectos de tipo Librería de Clases (`.csproj`). Estos proyectos respetarán las capas de Clean Architecture (`Domain`, `Application`, `Infrastructure`).
3.  **Comunicación en Memoria (In-Process):** La interacción entre módulos (ej. Ventas informando a Inventario de una reducción de stock) se realizará estrictamente en memoria utilizando el patrón Mediador (con `MediatR`) o inyectando interfaces expuestas por cada módulo, garantizando transaccionalidad y baja latencia.

## Consecuencias

### Positivas
*   **Aislamiento Fuerte por Compilador:** A diferencia del ADR-008 donde la separación era solo por carpetas, tener Librerías de Clases distintas impide referencias circulares y llamadas accidentales entre componentes internos (por ejemplo, el Controller de Ventas no podrá referenciar el DbContext de Inventario si no existe la referencia en el `.csproj`).
*   **Resiliencia Transaccional:** Al estar todo en el mismo proceso y base de datos, descontar inventario tras un pago es síncrono y seguro; si ocurre una excepción, se aplica un simple Rollback de la transacción en la base de datos sin requerir patrones complejos como Saga.
*   **Simplicidad de Despliegue:** Un solo pipeline CI/CD, un solo contenedor/servicio corriendo en el VPS, eliminando la necesidad inmediata de orquestadores complejos o API Gateways para enrutamiento interno.

### Negativas / Riesgos
*   **Escalabilidad Acoplada:** Si el módulo PDV sufre un pico masivo de peticiones un viernes por la noche, consumirá los recursos computacionales de la única Web API y podría ralentizar el sistema Backoffice corporativo.
*   **Tiempos de Compilación:** A medida que crezcan los módulos, compilar la solución entera tomará más tiempo, aunque en la escala actual (MVP) el impacto es despreciable.

## Nota de Reemplazo
Esta decisión **reemplaza explícitamente al ADR-008** y matiza la división técnica del monorepo mencionada en el ADR-006 (todo el backend ahora convivirá en una única solución unificada de C#).
