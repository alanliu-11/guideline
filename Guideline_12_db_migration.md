## 12. Database Migration (Flyway)

### 12.1 Migration File Structure
> Use Flyway for version-controlled database migrations.  
> Follow naming convention: V{version}__{description}.sql

```
text
src/main/resources/
└── db/
    └── migration/
        ├── V1__create_customers_table.sql
        ├── V2__create_orders_table.sql
        ├── V3__create_order_items_table.sql
        ├── V4__add_order_status_index.sql
        └── V5__add_audit_columns.sql
```

### 12.2 Migration Examples
> Always use transactions for migrations.  
> Add constraints and indexes for data integrity and performance.

```
sql
-- V1__create_customers_table.sql
CREATE TABLE customers (
    id              UUID PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    registered_at   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_customer_status CHECK (status IN ('ACTIVE', 'INACTIVE', 'SUSPENDED'))
);

CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_status ON customers(status);

-- V2__create_orders_table.sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'DRAFT',
    total_amount    DECIMAL(19, 4) NOT NULL DEFAULT 0,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    placed_at       TIMESTAMP WITH TIME ZONE,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_order_status CHECK (status IN ('DRAFT', 'SUBMITTED', 'PAID', 'SHIPPED', 'DELIVERED', 'CANCELLED')),
    CONSTRAINT chk_order_currency CHECK (currency IN ('USD', 'EUR', 'GBP'))
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_placed_at ON orders(placed_at);

-- V3__create_order_items_table.sql
CREATE TABLE order_items (
    id              UUID PRIMARY KEY,
    order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL,
    product_name    VARCHAR(255) NOT NULL,
    quantity        INT NOT NULL,
    unit_price      DECIMAL(19, 4) NOT NULL,
    currency        VARCHAR(3) NOT NULL,
    
    CONSTRAINT chk_item_quantity CHECK (quantity > 0),
    CONSTRAINT chk_item_price CHECK (unit_price >= 0)
);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- V4__add_order_status_index.sql
-- Partial index for common queries
CREATE INDEX idx_orders_active ON orders(customer_id, placed_at) 
    WHERE status NOT IN ('DELIVERED', 'CANCELLED');

-- V5__add_audit_columns.sql
ALTER TABLE orders ADD COLUMN created_by VARCHAR(255);
ALTER TABLE orders ADD COLUMN updated_by VARCHAR(255);

ALTER TABLE customers ADD COLUMN created_by VARCHAR(255);
ALTER TABLE customers ADD COLUMN updated_by VARCHAR(255);
```
