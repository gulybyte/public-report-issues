[case explain](https://github.com/gulybyte/public-report-issues/blob/main/PERFORMANCE%20regression%3A%20EXECUTE%20loop%20TRUNCATE%20loop%20slower%20in%2017.5%20vs%2017.2/EMAIL.md)

# setup steps
run both versions:
```sh
docker compose up postgres172 postgres175
```
psql postgres:17.2-alpine:
```sh
docker exec -it postgres172 psql -U postgres -d postgres
```
psql postgres:17.5-alpine:
```sh
docker exec -it postgres175 psql -U postgres -d postgres
```

sample schema to test (run both):
```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP
);
CREATE TABLE roles (
    role_id SERIAL PRIMARY KEY,
    role_name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT
);
CREATE TABLE user_roles (
    user_id INT NOT NULL,
    role_id INT NOT NULL,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(role_id) ON DELETE CASCADE
);
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT
);
CREATE TABLE suppliers (
    supplier_id SERIAL PRIMARY KEY,
    name VARCHAR(150) NOT NULL,
    contact_name VARCHAR(100),
    contact_email VARCHAR(100),
    phone VARCHAR(30)
);
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    category_id INT NOT NULL,
    supplier_id INT,
    name VARCHAR(150) NOT NULL,
    description TEXT,
    price NUMERIC(12,2) NOT NULL CHECK (price >= 0),
    stock_quantity INT NOT NULL CHECK (stock_quantity >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(category_id),
    FOREIGN KEY (supplier_id) REFERENCES suppliers(supplier_id)
);
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE,
    phone VARCHAR(30),
    address TEXT,
    city VARCHAR(100),
    country VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(30) NOT NULL,
    total_amount NUMERIC(12,2) NOT NULL CHECK (total_amount >= 0),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(12,2) NOT NULL CHECK (unit_price >= 0),
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
CREATE TABLE payments (
    payment_id SERIAL PRIMARY KEY,
    order_id INT NOT NULL,
    payment_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    amount NUMERIC(12,2) NOT NULL CHECK (amount >= 0),
    payment_method VARCHAR(50) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);
CREATE TABLE shipments (
    shipment_id SERIAL PRIMARY KEY,
    order_id INT NOT NULL,
    shipped_date TIMESTAMP,
    delivery_date TIMESTAMP,
    carrier VARCHAR(100),
    tracking_number VARCHAR(100),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);
CREATE TABLE reviews (
    review_id SERIAL PRIMARY KEY,
    product_id INT NOT NULL,
    customer_id INT NOT NULL,
    rating INT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment TEXT,
    review_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
CREATE TABLE carts (
    cart_id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
CREATE TABLE cart_items (
    cart_item_id SERIAL PRIMARY KEY,
    cart_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    FOREIGN KEY (cart_id) REFERENCES carts(cart_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
CREATE TABLE discounts (
    discount_id SERIAL PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    description TEXT,
    discount_percent NUMERIC(5,2) CHECK (discount_percent >= 0 AND discount_percent <= 100),
    active BOOLEAN DEFAULT TRUE,
    valid_from DATE,
    valid_to DATE
);
CREATE TABLE product_discounts (
    product_id INT NOT NULL,
    discount_id INT NOT NULL,
    PRIMARY KEY (product_id, discount_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (discount_id) REFERENCES discounts(discount_id)
);
CREATE TABLE audit_logs (
    log_id SERIAL PRIMARY KEY,
    user_id INT,
    action VARCHAR(100) NOT NULL,
    action_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    details JSONB,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
CREATE TABLE addresses (
    address_id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(100) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

# use case
before run `\timing`:
```sh
\timing
```
use case
```sql
DO $$
DECLARE
    r RECORD;
BEGIN
    FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname = 'public') LOOP
        EXECUTE 'TRUNCATE TABLE ' || quote_ident(r.tablename) || ' RESTART IDENTITY CASCADE';
    END LOOP;
END $$;
```
