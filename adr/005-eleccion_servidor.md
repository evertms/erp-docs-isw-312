# ADR-005: Elección de Proveedor de Infraestructura (VPS vs Cloud)

## Estado
Aceptada

## Contexto
Para desplegar el sistema de inventario (Monolito Modular en .NET 10 + PostgreSQL + React), requerimos una infraestructura que soporte:
1.  **Carga Estimada:** 30,000 productos, ~90,000 imágenes (~9GB), y 50 usuarios concurrentes.
2.  **Tráfico de Imágenes:** Almacenamiento local de imágenes para evitar costos de transferencia (Egress), estimado en hasta 1.4 TB/mes (Ver ADR-004).
3.  **Presupuesto:** Limitado, priorizando costo-efectividad.
4.  **Requisitos de Hardware:**
    *   **RAM:** Mínimo 8 GB (para soportar .NET + PostgreSQL + Nginx + Caching en memoria sin *swapping*).
    *   **Almacenamiento:** Mínimo 80-100 GB **NVMe** (Indispensable para lectura rápida de imágenes y DB).
    *   **Ancho de Banda:** Generoso (>2TB) o Ilimitado para no pagar sobrecostos por ver las imágenes.

### Opciones Consideradas

1.  **Hostinger (VPS KVM 2)**
    *   *Specs:* 2 vCPU, 8 GB RAM, 100 GB NVMe.
    *   *Ancho de Banda:* 8 TB.
    *   *Costo:* ~$8 USD/mes (Intro), ~$21 USD/mes (Renovación).
    *   *Pros:* Mejor precio del mercado por 8GB RAM. Panel simple.
    *   *Contras:* Soporte limitado comparado con AWS Enterprise.

2.  **AWS Lightsail (Instancia 8GB)**
    *   *Specs:* 2 vCPU, 8 GB RAM, 160 GB SSD.
    *   *Ancho de Banda:* 5 TB incluidos.
    *   *Costo:* $40 USD/mes fijo.
    *   *Pros:* Integración nativa con ecosistema AWS si se necesita crecer. IP estática incluida.
    *   *Contras:* Doble de precio que Hostinger. SSD estándar vs NVMe (dependiendo de la zona).

3.  **AWS EC2 (t3.large)**
    *   *Specs:* 2 vCPU, 8 GB RAM.
    *   *Ancho de Banda:* **Pago por uso ($0.09/GB)**.
    *   *Costo:* ~$60 USD (instancia) + ~$120 USD (transferencia 1.4TB) = **~$180 USD/mes**.
    *   *Pros:* Escalabilidad infinita.
    *   *Contras:* **Inviable económicamente** debido al costo de transferencia de salida (Egress) de las imágenes.

4.  **Hetzner Cloud (CPX31)**
    *   *Specs:* 4 vCPU, 8 GB RAM, 160 GB NVMe.
    *   *Ancho de Banda:* 20 TB.
    *   *Costo:* ~€14 EUR/mes.
    *   *Pros:* Increíble rendimiento/precio en Europa.
    *   *Contras:* Latencia más alta si los usuarios están en Latam (vs servidores en USA). Dificultad de pago/registro en algunas regiones.

## Decisión
Se ha decidido contratar un **VPS de Gama Media (Ej. Hostinger KVM 2 o equivalente)**.

### Justificación
1.  **El Factor Decisivo: Ancho de Banda.**
    *   Necesitamos mover ~1.4 TB de imágenes al mes.
    *   En **AWS EC2**, esto costaría ~$120 USD extra.
    *   En un **VPS (Hostinger/Hetzner)**, esto cuesta **$0 USD** (incluido).
    *   *Conclusión:* La nube pública "pay-as-you-go" está descartada por el modelo de negocio de imágenes pesadas.

2.  **Memoria RAM vs Precio.**
    *   .NET y PostgreSQL son robustos pero consumen RAM. 8 GB es el "punto dulce" para dormir tranquilos.
    *   Conseguir 8 GB en Hostinger cuesta ~$8-20. En AWS Lightsail cuesta $40. En Heroku/Render costaría >$200.
    *   La relación costo-beneficio del VPS es imbatible (4x-10x más barato).

3.  **Almacenamiento NVMe.**
    *   El ADR-004 exige servir imágenes desde disco local. Los discos NVMe son esenciales para que la experiencia de usuario (cargar galería de fotos) sea instantánea. Hostinger ofrece NVMe por defecto.

## Consecuencias

### Positivas
*   **Sostenibilidad Financiera:** El proyecto es viable con un costo operativo menor a $25 USD/mes, incluso con tráfico alto.
*   **Performance:** Recursos dedicados (RAM/CPU) garantizan que el monolito modular funcione fluido.

### Negativas / Riesgos
*   **Vendor Lock-in (Leve):** Migrar de un VPS a otro es manual (copiar archivos, configurar Linux), a diferencia de contenedores orquestados que se mueven solos.
    *   *Mitigación:* Usar Docker para el despliegue hace que cambiar de proveedor sea cuestión de minutos (instalar Docker en el nuevo VPS y correr el contenedor).
*   **Latencia Geográfica:** Debemos elegir la ubicación del data center (DC) más cercana a los usuarios (ej. USA East para Latam) para minimizar el ping.

## Nota Final
Aunque se menciona "Hostinger KVM 2" como referencia principal por su precio actual, la decisión arquitectónica es **"Usar cualquier VPS con 8GB RAM + NVMe + Tráfico Ilimitado"**. Si mañana DigitalOcean o Contabo ofrecen mejor precio con las mismas specs, la arquitectura no cambia.
