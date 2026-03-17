erDiagram
    %% Esquema: CORE
    COMPANIES {
        uuid id PK
        string name
        boolean is_active
        timestamp created_at
    }

    USERS {
        uuid id PK
        uuid company_id FK
        string name
        enum role "SUPERADMIN, ADMIN, CAJERO, MESERO, COCINA, BAR"
        boolean is_active
    }

    CUSTOMERS {
        uuid id PK
        uuid company_id FK
        string name
        string phone "nullable"
        boolean is_active
        timestamp created_at
    }

    MASTER_PRODUCTS {
        uuid id PK
        string barcode
        string standard_name
        string image_url
    }

    COMPANIES ||--o{ USERS : has
    COMPANIES ||--o{ CUSTOMERS : has