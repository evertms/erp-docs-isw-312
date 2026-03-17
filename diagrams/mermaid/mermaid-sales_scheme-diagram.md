erDiagram
    %% Esquema: SALES
    TAX_CONFIGURATIONS {
        uuid id PK
        uuid company_id "Logical Ref (Core)"
        decimal global_tax_rate "Para HU-09"
    }

    STATION_CATEGORY_CONFIGS {
        uuid id PK
        uuid company_id "Logical Ref (Core)"
        uuid category_id "Logical Ref (Inventory.Categories)"
        enum station "COCINA, BAR"
    }

    TICKETS {
        uuid id PK
        uuid company_id "Logical Ref (Core)"
        uuid customer_id "Logical Ref (Core.Customers) Nullable"
        uuid waiter_id "Logical Ref (Core.Users) Para HU-11"
        enum status "OPEN, PAID, CANCELED"
        decimal subtotal
        decimal tax_amount
        decimal total
        timestamp created_at
    }

    TICKET_LINES {
        uuid id PK
        uuid ticket_id FK
        integer command_number "Secuencial visual para agrupar (reemplaza a COMMANDS)"
        uuid product_id "Logical Ref (Inventory.Products)"
        string product_name "Réplica para evitar consultas cross schema"
        decimal quantity
        decimal unit_price
        string notes "Para HU-13"
        enum station "COCINA, BAR"
        timestamp sent_at "Nullable, fecha en que se envió a cocina/bar"
        enum status "PENDING, PREPARING, READY, SERVED"
    }

    %% Nueva tabla para registrar el cobro (HU-19)
    PAYMENTS {
        uuid id PK
        uuid ticket_id FK
        enum method "EFECTIVO, TARJETA, QR"
        decimal amount
        timestamp paid_at
    }

    TICKETS ||--o{ TICKET_LINES : contains
    TICKETS ||--o{ PAYMENTS : receives