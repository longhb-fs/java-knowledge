# Ngày 07: Flyway Migration & Data Seeding

## Mục tiêu hôm nay
- Hiểu tại sao cần Database Migration thay vì ddl-auto
- Cài đặt và cấu hình Flyway
- Viết migration scripts (SQL versioned)
- Seed dữ liệu mẫu (categories, products)
- Best practices cho production migrations

---

## 1. TẠI SAO CẦN DATABASE MIGRATION?

### 1.1. Vấn đề với ddl-auto

```yaml
# ❌ KHÔNG BAO GIỜ dùng trong Production
spring:
  jpa:
    hibernate:
      ddl-auto: update  # Nguy hiểm!
```

**Vấn đề:**
| ddl-auto | Vấn đề |
|----------|--------|
| `update` | Không xóa column cũ, gây inconsistent schema |
| `update` | Không rename được, chỉ thêm column mới |
| `update` | Không kiểm soát được thứ tự thay đổi |
| `create` | Xóa hết dữ liệu mỗi lần start |
| `none` | Không có migration history |

### 1.2. Database Migration giải quyết gì?

```
┌─────────────────────────────────────────────────────────────┐
│                   DATABASE MIGRATION                         │
│                                                              │
│  ✅ Version control cho database schema                      │
│  ✅ Team đồng bộ schema dễ dàng (Git)                        │
│  ✅ Rollback được khi có lỗi                                 │
│  ✅ CI/CD tự động apply migrations                           │
│  ✅ Audit trail (ai, khi nào, thay đổi gì)                   │
│  ✅ Reproducible (local = staging = prod)                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Flow:
Developer → Viết migration script → Commit Git → CI/CD apply → All environments consistent
```

### 1.3. Flyway vs Liquibase

| Tiêu chí | Flyway | Liquibase |
|----------|--------|-----------|
| **Cú pháp** | SQL thuần túy | XML, YAML, JSON, SQL |
| **Học** | Dễ (biết SQL là đủ) | Phức tạp hơn |
| **Rollback** | Có (Pro version) | Có (tự động) |
| **Phổ biến** | Rất phổ biến | Phổ biến |
| **Spring Boot** | Hỗ trợ tốt | Hỗ trợ tốt |

**Khóa học này dùng Flyway** vì đơn giản, dùng SQL thuần.

---

## 2. CÀI ĐẶT FLYWAY

### 2.1. Dependencies (đã có trong pom.xml)

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

### 2.2. Cấu hình application.yml

```yaml
spring:
  # TẮT hibernate ddl-auto
  jpa:
    hibernate:
      ddl-auto: none  # Flyway quản lý schema!

  # Flyway config
  flyway:
    enabled: true
    baseline-on-migrate: true  # Tạo baseline nếu DB đã có data
    baseline-version: 0        # Version baseline
    locations: classpath:db/migration  # Thư mục chứa scripts
    validate-on-migrate: true  # Validate trước khi migrate
    clean-disabled: true       # Cấm flyway clean trong prod!
```

### 2.3. Tạo thư mục migrations

```
src/main/resources/
└── db/
    └── migration/
        ├── V1__create_users_table.sql
        ├── V2__create_categories_table.sql
        ├── V3__create_products_table.sql
        ├── V4__create_orders_tables.sql
        ├── V5__create_cart_reviews_tables.sql
        └── V6__seed_initial_data.sql
```

---

## 3. VIẾT MIGRATION SCRIPTS

### 3.1. Quy tắc đặt tên file

```
V{version}__{description}.sql

V  = Prefix bắt buộc (Versioned migration)
{version} = Số version (1, 2, 3... hoặc 1.1, 1.2...)
__ = HAI dấu gạch dưới (separator)
{description} = Mô tả (dùng underscore thay space)
.sql = Extension

Ví dụ:
V1__create_users_table.sql       ✅
V2__add_phone_to_users.sql       ✅
V1.1__create_indexes.sql         ✅
V10__update_products.sql         ✅ (version 10)
v1__create_table.sql             ❌ (phải viết hoa V)
V1_create_table.sql              ❌ (thiếu 1 dấu gạch dưới)
```

### 3.2. V1__create_users_table.sql

```sql
-- V1__create_users_table.sql
-- Tạo bảng users

CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    avatar_url VARCHAR(500),
    role VARCHAR(20) NOT NULL DEFAULT 'CUSTOMER',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    is_email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    INDEX idx_users_email (email),
    INDEX idx_users_phone (phone),
    INDEX idx_users_role (role)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Tạo bảng addresses
CREATE TABLE addresses (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    address_line VARCHAR(500) NOT NULL,
    ward VARCHAR(100),
    district VARCHAR(100),
    city VARCHAR(100) NOT NULL,
    postal_code VARCHAR(20),
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT fk_address_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_address_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.3. V2__create_categories_table.sql

```sql
-- V2__create_categories_table.sql
-- Tạo bảng categories (hỗ trợ nested categories)

CREATE TABLE categories (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    image_url VARCHAR(500),
    parent_id BIGINT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    sort_order INT DEFAULT 0,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT fk_category_parent FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE SET NULL,
    INDEX idx_category_slug (slug),
    INDEX idx_category_parent (parent_id),
    INDEX idx_category_active (is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.4. V3__create_products_table.sql

```sql
-- V3__create_products_table.sql
-- Tạo bảng products và related tables

CREATE TABLE products (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(200) NOT NULL UNIQUE,
    sku VARCHAR(50) NOT NULL UNIQUE,
    description TEXT,
    price DECIMAL(12,2) NOT NULL,
    sale_price DECIMAL(12,2),
    stock_quantity INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    is_featured BOOLEAN NOT NULL DEFAULT FALSE,
    main_image VARCHAR(500),
    category_id BIGINT,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT fk_product_category FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL,
    INDEX idx_product_slug (slug),
    INDEX idx_product_sku (sku),
    INDEX idx_product_category (category_id),
    INDEX idx_product_price (price),
    INDEX idx_product_active (is_active),
    INDEX idx_product_featured (is_featured)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Tạo bảng product_images
CREATE TABLE product_images (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT NOT NULL,
    url VARCHAR(500) NOT NULL,
    alt_text VARCHAR(200),
    sort_order INT DEFAULT 0,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT fk_image_product FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    INDEX idx_image_product (product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Tạo bảng tags
CREATE TABLE tags (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    slug VARCHAR(50) NOT NULL UNIQUE,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Tạo bảng product_tags (many-to-many)
CREATE TABLE product_tags (
    product_id BIGINT NOT NULL,
    tag_id BIGINT NOT NULL,
    PRIMARY KEY (product_id, tag_id),
    CONSTRAINT fk_pt_product FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    CONSTRAINT fk_pt_tag FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.5. V4__create_orders_tables.sql

```sql
-- V4__create_orders_tables.sql
-- Tạo bảng orders, order_items, payments

CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_number VARCHAR(50) NOT NULL UNIQUE,
    user_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    subtotal DECIMAL(12,2) NOT NULL,
    shipping_fee DECIMAL(12,2) DEFAULT 0,
    discount_amount DECIMAL(12,2) DEFAULT 0,
    total_amount DECIMAL(12,2) NOT NULL,
    shipping_name VARCHAR(100) NOT NULL,
    shipping_phone VARCHAR(20) NOT NULL,
    shipping_address VARCHAR(500) NOT NULL,
    payment_method VARCHAR(20) NOT NULL,
    payment_status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    note TEXT,
    cancel_reason VARCHAR(500),
    cancelled_at DATETIME(6),
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT fk_order_user FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_order_number (order_number),
    INDEX idx_order_user (user_id),
    INDEX idx_order_status (status),
    INDEX idx_order_payment_status (payment_status),
    INDEX idx_order_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE order_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT NOT NULL,
    product_id BIGINT,
    quantity INT NOT NULL,
    unit_price DECIMAL(12,2) NOT NULL,
    subtotal DECIMAL(12,2) NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    product_sku VARCHAR(50) NOT NULL,
    product_image VARCHAR(500),
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT fk_oi_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    CONSTRAINT fk_oi_product FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE SET NULL,
    INDEX idx_oi_order (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE payments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT NOT NULL UNIQUE,
    amount DECIMAL(12,2) NOT NULL,
    method VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    transaction_id VARCHAR(100) UNIQUE,
    provider VARCHAR(50),
    paid_at DATETIME(6),
    refunded_at DATETIME(6),
    failure_reason VARCHAR(500),
    raw_response TEXT,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT fk_payment_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    INDEX idx_payment_transaction (transaction_id),
    INDEX idx_payment_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.6. V5__create_cart_reviews_tables.sql

```sql
-- V5__create_cart_reviews_tables.sql
-- Tạo bảng cart_items và reviews

CREATE TABLE cart_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT fk_cart_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_cart_product FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    UNIQUE KEY uk_cart_user_product (user_id, product_id),
    INDEX idx_cart_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE reviews (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    rating INT NOT NULL,
    comment TEXT,
    is_verified_purchase BOOLEAN NOT NULL DEFAULT FALSE,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT fk_review_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_review_product FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    UNIQUE KEY uk_review_user_product (user_id, product_id),
    INDEX idx_review_product (product_id),
    INDEX idx_review_rating (rating)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 4. DATA SEEDING

### 4.1. V6__seed_initial_data.sql

```sql
-- V6__seed_initial_data.sql
-- Seed dữ liệu mẫu cho development/testing

-- Seed Categories
INSERT INTO categories (name, slug, description, is_active, sort_order) VALUES
('Điện thoại', 'dien-thoai', 'Điện thoại di động các hãng', TRUE, 1),
('Laptop', 'laptop', 'Máy tính xách tay', TRUE, 2),
('Tablet', 'tablet', 'Máy tính bảng', TRUE, 3),
('Phụ kiện', 'phu-kien', 'Phụ kiện điện tử', TRUE, 4),
('Đồng hồ thông minh', 'dong-ho-thong-minh', 'Smartwatch các loại', TRUE, 5);

-- Seed Sub-categories
INSERT INTO categories (name, slug, description, parent_id, is_active, sort_order) VALUES
('iPhone', 'iphone', 'Điện thoại Apple iPhone', 1, TRUE, 1),
('Samsung', 'samsung', 'Điện thoại Samsung Galaxy', 1, TRUE, 2),
('Xiaomi', 'xiaomi', 'Điện thoại Xiaomi', 1, TRUE, 3),
('MacBook', 'macbook', 'Laptop Apple MacBook', 2, TRUE, 1),
('Dell', 'dell', 'Laptop Dell', 2, TRUE, 2),
('Asus', 'asus', 'Laptop Asus', 2, TRUE, 3);

-- Seed Tags
INSERT INTO tags (name, slug) VALUES
('Mới', 'moi'),
('Bán chạy', 'ban-chay'),
('Giảm giá', 'giam-gia'),
('Freeship', 'freeship'),
('Cao cấp', 'cao-cap'),
('Phổ thông', 'pho-thong');

-- Seed Products
INSERT INTO products (name, slug, sku, description, price, sale_price, stock_quantity, is_active, is_featured, main_image, category_id) VALUES
('iPhone 15 Pro Max 256GB', 'iphone-15-pro-max-256gb', 'IP15PM256',
 'iPhone 15 Pro Max với chip A17 Pro, camera 48MP, màn hình Super Retina XDR 6.7 inch',
 34990000, 33990000, 100, TRUE, TRUE, '/images/products/iphone-15-pro-max.jpg', 6),

('iPhone 15 128GB', 'iphone-15-128gb', 'IP15128',
 'iPhone 15 với Dynamic Island, camera 48MP, chip A16 Bionic',
 22990000, NULL, 150, TRUE, TRUE, '/images/products/iphone-15.jpg', 6),

('Samsung Galaxy S24 Ultra 256GB', 'samsung-galaxy-s24-ultra-256gb', 'SGS24U256',
 'Samsung Galaxy S24 Ultra với Galaxy AI, S Pen, camera 200MP',
 33990000, 31990000, 80, TRUE, TRUE, '/images/products/samsung-s24-ultra.jpg', 7),

('MacBook Pro 14" M3 Pro', 'macbook-pro-14-m3-pro', 'MBP14M3P',
 'MacBook Pro 14 inch với chip M3 Pro, RAM 18GB, SSD 512GB',
 49990000, NULL, 50, TRUE, TRUE, '/images/products/macbook-pro-14.jpg', 9),

('MacBook Air 13" M3', 'macbook-air-13-m3', 'MBA13M3',
 'MacBook Air 13 inch với chip M3, RAM 8GB, SSD 256GB',
 27990000, 26490000, 120, TRUE, FALSE, '/images/products/macbook-air-13.jpg', 9),

('Dell XPS 15', 'dell-xps-15', 'DELLXPS15',
 'Dell XPS 15 với Intel Core i7-13700H, RAM 16GB, SSD 512GB',
 42990000, NULL, 40, TRUE, FALSE, '/images/products/dell-xps-15.jpg', 10),

('iPad Pro 12.9" M2 256GB', 'ipad-pro-12-9-m2-256gb', 'IPADPRO129M2',
 'iPad Pro 12.9 inch với chip M2, màn hình Liquid Retina XDR',
 28990000, 27490000, 60, TRUE, TRUE, '/images/products/ipad-pro.jpg', 3),

('Apple Watch Series 9 45mm', 'apple-watch-series-9-45mm', 'AWS945',
 'Apple Watch Series 9 GPS + Cellular, vỏ nhôm, dây thể thao',
 11990000, NULL, 100, TRUE, FALSE, '/images/products/apple-watch-s9.jpg', 5),

('AirPods Pro 2', 'airpods-pro-2', 'APP2',
 'AirPods Pro thế hệ 2 với chip H2, chống ồn chủ động',
 6790000, 5990000, 200, TRUE, TRUE, '/images/products/airpods-pro-2.jpg', 4),

('Xiaomi 14 Ultra', 'xiaomi-14-ultra', 'XM14U',
 'Xiaomi 14 Ultra với camera Leica, Snapdragon 8 Gen 3',
 29990000, NULL, 45, TRUE, FALSE, '/images/products/xiaomi-14-ultra.jpg', 8);

-- Seed product_tags
INSERT INTO product_tags (product_id, tag_id) VALUES
(1, 1), (1, 2), (1, 5),  -- iPhone 15 Pro Max: Mới, Bán chạy, Cao cấp
(2, 1), (2, 2),          -- iPhone 15: Mới, Bán chạy
(3, 1), (3, 5),          -- Samsung S24 Ultra: Mới, Cao cấp
(4, 5),                   -- MacBook Pro: Cao cấp
(5, 3), (5, 4),          -- MacBook Air: Giảm giá, Freeship
(7, 3), (7, 5),          -- iPad Pro: Giảm giá, Cao cấp
(9, 2), (9, 3);          -- AirPods Pro: Bán chạy, Giảm giá

-- Seed Admin user (password: Admin@123)
-- BCrypt hash generated with: https://bcrypt-generator.com/
INSERT INTO users (email, password, full_name, phone, role, is_active, is_email_verified) VALUES
('admin@example.com', '$2a$10$5Dqx7QqLKqVqQQe8q5YU0eJ3V5qF1JZ6Xu5Oo8OxFzO3hKqFj5qNe',
 'Admin', '0123456789', 'ADMIN', TRUE, TRUE);
```

---

## 5. FLYWAY HISTORY TABLE

### 5.1. Cách Flyway tracking migrations

```sql
-- Flyway tự tạo table này
SELECT * FROM flyway_schema_history;

+----------------+---------+-------------------------+------+---------------------------+
| installed_rank | version | description             | type | script                    |
+----------------+---------+-------------------------+------+---------------------------+
| 1              | 1       | create users table      | SQL  | V1__create_users_table.sql|
| 2              | 2       | create categories table | SQL  | V2__create_categories... |
| 3              | 3       | create products table   | SQL  | V3__create_products...   |
| 4              | 4       | create orders tables    | SQL  | V4__create_orders_tables |
| 5              | 5       | create cart reviews...  | SQL  | V5__create_cart_reviews..|
| 6              | 6       | seed initial data       | SQL  | V6__seed_initial_data... |
+----------------+---------+-------------------------+------+---------------------------+
```

### 5.2. Chạy migrations

```bash
# Khởi động ứng dụng → Flyway tự động apply migrations
./mvnw spring-boot:run

# Output:
# Flyway Community Edition 9.x.x
# Schema version: << Empty Schema >>
# Migrating schema `ecommerce` to version "1 - create users table"
# Migrating schema `ecommerce` to version "2 - create categories table"
# ...
# Successfully applied 6 migrations to schema `ecommerce`
```

---

## 6. REPEATABLE MIGRATIONS

### 6.1. Repeatable Migration là gì?

```
Versioned Migration (V):
- Chạy 1 lần duy nhất
- Có version number
- Không được sửa sau khi applied

Repeatable Migration (R):
- Chạy lại mỗi khi file thay đổi (checksum)
- Không có version
- Dùng cho views, stored procedures, functions
```

### 6.2. Ví dụ R__create_views.sql

```sql
-- R__create_views.sql
-- Repeatable: chạy lại mỗi khi nội dung thay đổi

-- View: Product với Category name
CREATE OR REPLACE VIEW v_products_with_category AS
SELECT
    p.id,
    p.name,
    p.slug,
    p.sku,
    p.price,
    p.sale_price,
    p.stock_quantity,
    p.is_active,
    p.is_featured,
    p.main_image,
    c.name AS category_name,
    c.slug AS category_slug,
    p.created_at,
    p.updated_at
FROM products p
LEFT JOIN categories c ON p.category_id = c.id;

-- View: Order summary
CREATE OR REPLACE VIEW v_order_summary AS
SELECT
    o.id,
    o.order_number,
    o.status,
    o.total_amount,
    o.payment_status,
    o.created_at,
    u.email AS user_email,
    u.full_name AS user_name,
    COUNT(oi.id) AS item_count,
    SUM(oi.quantity) AS total_quantity
FROM orders o
JOIN users u ON o.user_id = u.id
LEFT JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id;
```

---

## 7. BEST PRACTICES

### 7.1. Migration script rules

```
1. ❌ KHÔNG BAO GIỜ sửa migration đã applied
   → Thay đổi? Viết migration mới

2. ✅ Mỗi migration = 1 logical change
   → V7__add_coupon_table.sql
   → V8__add_coupon_code_to_orders.sql

3. ✅ Migration phải idempotent-safe
   → Dùng IF NOT EXISTS, IF EXISTS

4. ✅ Test migration trên local trước khi push

5. ✅ Rollback script (nếu cần)
   → U7__add_coupon_table.sql (Undo)
```

### 7.2. Thêm column mới (ví dụ)

```sql
-- V7__add_discount_code_to_orders.sql
ALTER TABLE orders
ADD COLUMN discount_code VARCHAR(50) AFTER discount_amount;

ALTER TABLE orders
ADD INDEX idx_order_discount_code (discount_code);
```

### 7.3. Rename column (an toàn)

```sql
-- V8__rename_product_description.sql
-- Bước 1: Thêm column mới
ALTER TABLE products ADD COLUMN short_description VARCHAR(500);

-- Bước 2: Copy data
UPDATE products SET short_description = LEFT(description, 500);

-- Bước 3 (migration sau): Xóa column cũ
-- V9__drop_old_description.sql
-- ALTER TABLE products DROP COLUMN description;
-- ALTER TABLE products RENAME COLUMN short_description TO description;
```

---

## 8. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Tạo thư mục `src/main/resources/db/migration`
- [ ] Viết V1 → V6 migration scripts
- [ ] Cấu hình Flyway trong application.yml
- [ ] Tắt `ddl-auto` hoặc set `none`
- [ ] Chạy ứng dụng → verify tables được tạo
- [ ] Kiểm tra `flyway_schema_history` table
- [ ] Thêm migration V7 (thêm 1 column mới)
- [ ] Tạo 1 Repeatable migration cho view

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| Tại sao cần Database Migration | ☐ |
| Flyway vs Liquibase | ☐ |
| Quy tắc đặt tên file migration | ☐ |
| Versioned (V) vs Repeatable (R) | ☐ |
| flyway_schema_history table | ☐ |
| baseline-on-migrate là gì | ☐ |
| Best practices cho migration | ☐ |
| Không sửa migration đã applied | ☐ |

---

**Trước đó:** [← Ngày 06 - JPA Entity](day-06-jpa-entity.md)

**Tiếp theo:** [Ngày 08 - Repository & Custom Queries →](day-08-repository-queries.md)
