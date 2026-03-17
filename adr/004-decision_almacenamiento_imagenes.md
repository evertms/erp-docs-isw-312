# ADR-004: Estrategia de Almacenamiento de Imágenes

## Estado
Aceptada

## Contexto
El sistema de inventario manejará un catálogo de **30,000 productos**. Se estima un promedio de **3 imágenes por producto**, totalizando **~90,000 imágenes**.
Con una concurrencia de **50 usuarios activos** navegando el catálogo, el consumo de ancho de banda (Egress) será significativo.

### Estimación de Recursos
*   **Volumen de Datos:** 90,000 imágenes x 100KB (optimizado) ≈ **9 GB**.
*   **Tráfico Mensual Estimado:** Si 50 usuarios ven 10 productos/minuto durante 8 horas/día:
    *   (50 usuarios * 30 imgs * 100KB) * 60 min * 8 horas * 20 días ≈ **1.4 TB / mes**.
    *   *Nota:* Este es un escenario de uso intensivo ("Worst Case"), pero debemos estar preparados para no quebrar por costos de ancho de banda.

## Decisión 1: Formato de Imagen
Se utilizará **WebP**.

### Justificación
*   **Reducción de Peso:** WebP es un 30-50% más ligero que JPEG/PNG con la misma calidad visual. Para 90,000 imágenes, esto significa bajar de ~15GB a ~9GB de almacenamiento y, más importante, reducir drásticamente el consumo de datos de los clientes y del servidor.
*   **Soporte:** Universal en navegadores modernos (Chrome, Edge, Firefox, Safari).

## Decisión 2: Ubicación del Almacenamiento
Se utilizará **Almacenamiento Local en el Servidor (VPS)** servido vía **Nginx**.

### Comparativa de Opciones (Precios 2026)

#### 1. VPS Local (Opción Elegida)
*   **Requerimiento del Servidor:** Se deberá contratar un VPS que ofrezca disco **NVMe** (para velocidad de lectura) y un ancho de banda generoso (mínimo 2TB o ilimitado).
*   **Costo Almacenamiento:** $0 (Debe estar incluido en la cuota mensual del servidor). Las imágenes ocuparán ~9GB, lo cual es manejable para cualquier VPS moderno de gama media.
*   **Costo Transferencia (Egress):** **$0**. A diferencia de la nube pública (AWS/Azure), la mayoría de los proveedores de VPS no cobran por GB transferido hasta llegar a límites muy altos (ej. 4TB), lo cual cubre nuestra necesidad sin sobrecostos.
*   **Ventajas:** Costo fijo predecible, latencia cero (imagen y backend en el mismo servidor).
*   **Desventajas:** Si el servidor se destruye sin backup, se pierden las imágenes. Requiere configurar Nginx para servir estáticos.

#### 2. AWS S3 (Standard)
*   **Costo Almacenamiento:** ~$0.20 USD/mes (9GB * $0.023). *Barato.*
*   **Costo Transferencia (El problema real):** ~$126 USD/mes.
    *   AWS cobra ~$0.09 por GB de salida a internet. Para 1.4 TB de tráfico, el costo es prohibitivo para este presupuesto.
*   **Ventajas:** Durabilidad 99.999999999%, escalado infinito.
*   **Desventajas:** **Costos de salida extremadamente altos**. Complejidad de gestión de Keys/IAM.

#### 3. Supabase Storage (Pro Plan)
*   **Costo Fijo:** $25 USD/mes (Incluye 100GB almacenamiento y 250GB ancho de banda).
*   **Costo Transferencia Extra:** ~$103 USD/mes.
    *   Los primeros 250GB son gratis. Los 1,150 GB restantes del estimado (1.4TB total) se cobran a $0.09/GB.
*   **Ventajas:** API fácil de usar, CDN básico incluido.
*   **Desventajas:** Sigue siendo caro si superamos el límite de ancho de banda del plan Pro.

### Justificación de la Estrategia Local
Para un proyecto con presupuesto limitado, pagar +$100 USD/mes solo en transferencia de imágenes es inviable.
Al aprovechar el disco **NVMe** del futuro servidor, obtenemos una velocidad de lectura comparable a S3 pero sin los costos de salida (Egress), manteniendo el presupuesto mensual bajo control.
Se mitigan los riesgos de "perder los archivos" mediante una política de **Backups periódicos** (rsync/rclone) a un *bucket* de "S3 Cold Storage" (Glacier) que es muy barato solo para escribir y guardar.

## Consecuencias

### Positivas
*   **Economía:** Mantiene el costo total del proyecto fijo, sin sorpresas por picos de tráfico.
*   **Performance:** Servir archivos estáticos desde un disco NVMe local es extremadamente rápido.

### Negativas / Riesgos
*   **Escalabilidad de Disco:** Estamos limitados al tamaño del disco duro que contratemos.
    *   *Mitigación:* Cuando el negocio crezca lo suficiente, se podrá migrar a una solución de nube + CDN (Cloudflare).
*   **Backup Manual:** Debemos configurar scripts para respaldar la carpeta `/var/www/uploads` a un lugar seguro.

## Notas de Implementación
*   Las imágenes se subirán al servidor vía API (.NET) y se guardarán en una carpeta física mapeada.
*   Se usará una librería en .NET (ej: `ImageSharp`) para convertir automáticamente cualquier subida (JPG/PNG) a **WebP** y redimensionar si es necesario antes de guardar.
