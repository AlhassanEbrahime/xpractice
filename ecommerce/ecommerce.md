# 🛒 E-Commerce Database

> A relational database schema for an e-commerce platform, implemented in both **PostgreSQL** and **MySQL**. Covers product cataloguing, customer management, order processing, and business reporting.

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Entity-Relationship Diagram](#-entity-relationship-diagram)
- [Schema](#-schema)
  - [PostgreSQL](#postgresql)
  - [MySQL](#mysql)
- [Indexes](#-indexes)
- [Auto-Increment Configuration](#-auto-increment-configuration)
- [Table Reference](#-table-reference)
- [Relationships](#-relationships)
- [Business Reports / Queries](#-business-reports--queries)
  - [Daily Revenue Report](#1-daily-revenue-report)
  - [Monthly Top-Selling Products](#2-monthly-top-selling-products)
  - [High-Value Customers (Last 30 Days)](#3-high-value-customers-last-30-days)
- [PostgreSQL vs MySQL — Key Differences](#-postgresql-vs-mysql--key-differences)
- [Design Decisions](#-design-decisions)

---

## 🗂 Overview

| Table          | Purpose                                              |
|----------------|------------------------------------------------------|
| `CATEGORY`     | Hierarchical product categories (supports nesting)   |
| `PRODUCT`      | Product catalogue with pricing and stock levels      |
| `CUSTOMER`     | Registered customer accounts                         |
| `ORDERS`       | Order headers placed by customers                    |
| `ORDER_DETAILS`| Line items (products) within each order              |

**IDs start at 100** in both implementations.  
**Soft-delete** (`IS_DELETED`) is used on `CATEGORY` and `ORDERS` to preserve audit history.

---

## 🗺 Entity-Relationship Diagram

```
┌───────────────────────┐
│       CATEGORY        │
├───────────────────────┤
│ PK  CATEGORY_ID  int  │◄────────────────────────────────┐
│ FK  PARENT_ID    int? │─────────────────────────────────┘  (self-ref)
│     CATEGORY_NAME     │
│     IS_DELETED        │
└──────────┬────────────┘
           │ 1
           │ (ON DELETE SET NULL)
           │ 0..*
┌──────────▼────────────┐
│        PRODUCT        │
├───────────────────────┤
│ PK  PRODUCT_ID   int  │◄──────────────────────────────────────────┐
│ FK  CATEGORY_ID  int? │                                            │
│     NAME              │                                            │
│     DESCRIPTION       │                                            │
│     PRICE    dec(6,2) │                                            │
│     STOCK_QTY    int  │                                            │
└───────────────────────┘                                            │
                                                                     │ 0..*
┌───────────────────────┐           ┌───────────────────────────────────────────────┐
│       CUSTOMER        │           │                ORDER_DETAILS                  │
├───────────────────────┤           ├───────────────────────────────────────────────┤
│ PK  CUSTOMER_ID  int  │◄──┐       │ PK  ORDER_DETAIL_ID  int                      │
│     FIRST_NAME        │   │       │ FK  ORDER_ID          int  ──►ORDERS           │
│     LAST_NAME         │   │       │ FK  PRODUCT_ID        int  ──►PRODUCT (above)  │
│     EMAIL    (unique) │   │       │     QUANTITY           int                     │
│     PASSWORD          │   │       │     UNIT_PRICE   dec(6,2)                      │
│     IS_ACTIVE         │   │       └───────────────────────────────────────────────┘
└───────────────────────┘   │                        ▲
                            │ 1                      │ 1..*
                            │ (ON DELETE RESTRICT)   │
                            │ 0..*                   │
                   ┌────────┴────────────────┐       │
                   │         ORDERS          │───────┘
                   ├─────────────────────────┤
                   │ PK  ORDER_ID       int  │
                   │ FK  CUSTOMER_ID    int  │
                   │     ORDER_DATE    date  │
                   │     TOTAL_AMOUNT dec(10,2)
                   │     IS_DELETED  boolean │
                   └─────────────────────────┘
```

### Cardinality Summary

| Relationship                    | Type  | Rule on Delete        |
|---------------------------------|-------|-----------------------|
| `CATEGORY` → `CATEGORY`         | 1:N   | RESTRICT              |
| `PRODUCT` → `CATEGORY`          | N:1   | SET NULL              |
| `ORDERS` → `CUSTOMER`           | N:1   | RESTRICT              |
| `ORDER_DETAILS` → `ORDERS`      | N:1   | CASCADE (implicit)    |
| `ORDER_DETAILS` → `PRODUCT`     | N:1   | CASCADE (implicit)    |

---

## 🏗 Schema

### PostgreSQL

#### CATEGORY

```sql
CREATE TABLE CATEGORY
(
    CATEGORY_ID   INTEGER,
    PARENT_ID     INTEGER,
    CATEGORY_NAME VARCHAR(100),
    IS_DELETED    BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (CATEGORY_ID),
    FOREIGN KEY (PARENT_ID) REFERENCES CATEGORY ON DELETE RESTRICT
);
```

#### PRODUCT

```sql
CREATE TABLE PRODUCT
(
    PRODUCT_ID  INTEGER,
    CATEGORY_ID INTEGER,
    NAME        VARCHAR(100)  NOT NULL,
    DESCRIPTION VARCHAR(250),
    PRICE       NUMERIC(6, 2) NOT NULL CHECK (PRICE > 0 AND PRICE <= 9999.99),
    STOCK_QTY   INTEGER       NOT NULL CHECK ( STOCK_QTY >= 0 ),

    PRIMARY KEY (PRODUCT_ID),
    FOREIGN KEY (CATEGORY_ID) REFERENCES CATEGORY ON DELETE SET NULL
);
```

#### CUSTOMER

```sql
CREATE TABLE CUSTOMER
(
    CUSTOMER_ID INTEGER,
    FIRST_NAME  VARCHAR(20)        NOT NULL,
    LAST_NAME   VARCHAR(20),
    EMAIL       VARCHAR(50) UNIQUE NOT NULL,
    PASSWORD    VARCHAR(100)       NOT NULL,
    IS_ACTIVE   BOOLEAN            NOT NULL DEFAULT FALSE,
    PRIMARY KEY (CUSTOMER_ID)
);
```

#### ORDERS

```sql
CREATE TABLE ORDERS
(
    ORDER_ID     INTEGER,
    CUSTOMER_ID  INTEGER        NOT NULL,
    ORDER_DATE   DATE                    DEFAULT CURRENT_DATE NOT NULL,
    TOTAL_AMOUNT NUMERIC(10, 2) NOT NULL CHECK ( TOTAL_AMOUNT > 0 ),
    IS_DELETED   BOOLEAN        NOT NULL DEFAULT FALSE,
    PRIMARY KEY (ORDER_ID),
    FOREIGN KEY (CUSTOMER_ID) REFERENCES CUSTOMER ON DELETE RESTRICT
);
```

#### ORDER_DETAILS

```sql
CREATE TABLE ORDER_DETAILS
(
    ORDER_DETAIL_ID INTEGER,
    ORDER_ID        INTEGER       NOT NULL,
    PRODUCT_ID      INTEGER       NOT NULL,
    QUANTITY        INTEGER       NOT NULL CHECK ( QUANTITY > 0 ),
    UNIT_PRICE      NUMERIC(6, 2) NOT NULL CHECK (UNIT_PRICE > 0 AND UNIT_PRICE <= 9999.99),
    PRIMARY KEY (ORDER_DETAIL_ID),
    FOREIGN KEY (ORDER_ID) REFERENCES ORDERS,
    FOREIGN KEY (PRODUCT_ID) REFERENCES PRODUCT
);
```

---

### MySQL

#### CATEGORY

```sql
CREATE TABLE CATEGORY
(
    CATEGORY_ID   INTEGER AUTO_INCREMENT,
    PARENT_ID     INTEGER,
    CATEGORY_NAME VARCHAR(100),
    IS_DELETED    TINYINT(1) NOT NULL DEFAULT FALSE,
    PRIMARY KEY (CATEGORY_ID),
    FOREIGN KEY (PARENT_ID) REFERENCES CATEGORY (CATEGORY_ID) ON DELETE RESTRICT
);
```

#### PRODUCT

```sql
CREATE TABLE PRODUCT
(
    PRODUCT_ID  INTEGER AUTO_INCREMENT,
    CATEGORY_ID INTEGER,
    NAME        VARCHAR(100)  NOT NULL,
    DESCRIPTION VARCHAR(250),
    PRICE       DECIMAL(6, 2) NOT NULL CHECK (PRICE > 0 AND PRICE <= 9999.99),
    STOCK_QTY   INTEGER       NOT NULL CHECK ( STOCK_QTY >= 0 ),

    PRIMARY KEY (PRODUCT_ID),
    FOREIGN KEY (CATEGORY_ID) REFERENCES CATEGORY (CATEGORY_ID) ON DELETE SET NULL
);
```

#### CUSTOMER

```sql
CREATE TABLE CUSTOMER
(
    CUSTOMER_ID INTEGER AUTO_INCREMENT,
    FIRST_NAME  VARCHAR(20)        NOT NULL,
    LAST_NAME   VARCHAR(20),
    EMAIL       VARCHAR(50) UNIQUE NOT NULL,
    PASSWORD    VARCHAR(100)       NOT NULL,
    IS_ACTIVE   TINYINT(1)         NOT NULL DEFAULT FALSE,
    PRIMARY KEY (CUSTOMER_ID)
);
```

#### ORDERS

```sql
CREATE TABLE ORDERS
(
    ORDER_ID     INTEGER AUTO_INCREMENT,
    CUSTOMER_ID  INTEGER        NOT NULL,
    ORDER_DATE   DATE           NOT NULL DEFAULT (CURRENT_DATE),
    TOTAL_AMOUNT DECIMAL(10, 2) NOT NULL CHECK ( TOTAL_AMOUNT > 0 ),
    IS_DELETED   TINYINT(1)     NOT NULL DEFAULT FALSE,
    PRIMARY KEY (ORDER_ID),
    FOREIGN KEY (CUSTOMER_ID) REFERENCES CUSTOMER (CUSTOMER_ID) ON DELETE RESTRICT
);
```

#### ORDER_DETAILS

```sql
CREATE TABLE ORDER_DETAILS
(
    ORDER_DETAIL_ID INTEGER AUTO_INCREMENT,
    ORDER_ID        INTEGER       NOT NULL,
    PRODUCT_ID      INTEGER       NOT NULL,
    QUANTITY        INTEGER       NOT NULL CHECK ( QUANTITY > 0 ),
    UNIT_PRICE      DECIMAL(6, 2) NOT NULL CHECK (UNIT_PRICE > 0 AND UNIT_PRICE <= 9999.99),
    PRIMARY KEY (ORDER_DETAIL_ID),
    FOREIGN KEY (ORDER_ID) REFERENCES ORDERS (ORDER_ID),
    FOREIGN KEY (PRODUCT_ID) REFERENCES PRODUCT (PRODUCT_ID)
);
```

---

## ⚡ Indexes

Both PostgreSQL and MySQL use the same index definitions:

```sql
CREATE INDEX idx_orders_date          ON ORDERS        (ORDER_DATE);
CREATE INDEX idx_category_parent      ON CATEGORY      (PARENT_ID);
CREATE INDEX idx_orders_customer      ON ORDERS        (CUSTOMER_ID);
CREATE INDEX idx_product_category     ON PRODUCT       (CATEGORY_ID);
CREATE INDEX idx_order_details_order  ON ORDER_DETAILS (ORDER_ID);
CREATE INDEX idx_order_details_product ON ORDER_DETAILS (PRODUCT_ID);
```

| Index                        | Table          | Column        | Purpose                                  |
|------------------------------|----------------|---------------|------------------------------------------|
| `idx_orders_date`            | ORDERS         | ORDER_DATE    | Fast date-range filtering in reports     |
| `idx_category_parent`        | CATEGORY       | PARENT_ID     | Hierarchical category traversal          |
| `idx_orders_customer`        | ORDERS         | CUSTOMER_ID   | Lookup all orders for a customer         |
| `idx_product_category`       | PRODUCT        | CATEGORY_ID   | Listing products by category             |
| `idx_order_details_order`    | ORDER_DETAILS  | ORDER_ID      | Fetching line items for an order         |
| `idx_order_details_product`  | ORDER_DETAILS  | PRODUCT_ID    | Finding orders containing a product      |

---

## 🔢 Auto-Increment Configuration

### PostgreSQL — Identity Columns (start at 100)

```sql
ALTER TABLE CATEGORY
    ALTER COLUMN CATEGORY_ID ADD GENERATED ALWAYS AS IDENTITY (START WITH 100);

ALTER TABLE PRODUCT
    ALTER COLUMN PRODUCT_ID ADD GENERATED ALWAYS AS IDENTITY (START WITH 100);

ALTER TABLE CUSTOMER
    ALTER COLUMN CUSTOMER_ID ADD GENERATED ALWAYS AS IDENTITY (START WITH 100);

ALTER TABLE ORDERS
    ALTER COLUMN ORDER_ID ADD GENERATED ALWAYS AS IDENTITY (START WITH 100);

ALTER TABLE ORDER_DETAILS
    ALTER COLUMN ORDER_DETAIL_ID ADD GENERATED ALWAYS AS IDENTITY (START WITH 100);
```

> **Note:** `GENERATED ALWAYS` means the value is always system-generated. Use `GENERATED BY DEFAULT` if you need to manually insert IDs (e.g. for seeding/migrations).

### MySQL — InnoDB Engine + AUTO_INCREMENT

IDs start at 1 by default with `AUTO_INCREMENT`. To start at 100, add `AUTO_INCREMENT = 100` to the `CREATE TABLE` statement or use:

```sql
ALTER TABLE CATEGORY      AUTO_INCREMENT = 100;
ALTER TABLE PRODUCT       AUTO_INCREMENT = 100;
ALTER TABLE CUSTOMER      AUTO_INCREMENT = 100;
ALTER TABLE ORDERS        AUTO_INCREMENT = 100;
ALTER TABLE ORDER_DETAILS AUTO_INCREMENT = 100;
```

Setting InnoDB storage engine explicitly (required for foreign key support):

```sql
ALTER TABLE CATEGORY      ENGINE = InnoDB;
ALTER TABLE PRODUCT       ENGINE = InnoDB;
ALTER TABLE CUSTOMER      ENGINE = InnoDB;
ALTER TABLE ORDERS        ENGINE = InnoDB;
ALTER TABLE ORDER_DETAILS ENGINE = InnoDB;
```

---

## 📖 Table Reference

### CATEGORY

| Column        | Type         | Nullable | Default | Notes                                      |
|---------------|--------------|----------|---------|--------------------------------------------|
| CATEGORY_ID   | INTEGER      | NO       | auto    | PK — auto-generated                        |
| PARENT_ID     | INTEGER      | YES      | NULL    | FK → CATEGORY(CATEGORY_ID), self-reference |
| CATEGORY_NAME | VARCHAR(100) | YES      | NULL    |                                            |
| IS_DELETED    | BOOLEAN      | NO       | FALSE   | Soft-delete flag                           |

**Constraints:** `PARENT_ID` → `RESTRICT` on parent delete (cannot delete a parent that has children)

---

### PRODUCT

| Column      | Type          | Nullable | Default | Notes                                      |
|-------------|---------------|----------|---------|--------------------------------------------|
| PRODUCT_ID  | INTEGER       | NO       | auto    | PK — auto-generated                        |
| CATEGORY_ID | INTEGER       | YES      | NULL    | FK → CATEGORY(CATEGORY_ID)                 |
| NAME        | VARCHAR(100)  | NO       | —       |                                            |
| DESCRIPTION | VARCHAR(250)  | YES      | NULL    |                                            |
| PRICE       | NUMERIC(6,2)  | NO       | —       | CHECK: `> 0 AND <= 9999.99`                |
| STOCK_QTY   | INTEGER       | NO       | —       | CHECK: `>= 0`                              |

**Constraints:** If category is deleted → `SET NULL` on `CATEGORY_ID`

---

### CUSTOMER

| Column      | Type         | Nullable | Default | Notes                        |
|-------------|--------------|----------|---------|------------------------------|
| CUSTOMER_ID | INTEGER      | NO       | auto    | PK — auto-generated          |
| FIRST_NAME  | VARCHAR(20)  | NO       | —       |                              |
| LAST_NAME   | VARCHAR(20)  | YES      | NULL    |                              |
| EMAIL       | VARCHAR(50)  | NO       | —       | UNIQUE constraint             |
| PASSWORD    | VARCHAR(100) | NO       | —       | Store hashed; never plaintext|
| IS_ACTIVE   | BOOLEAN      | NO       | FALSE   | Account activation flag      |

---

### ORDERS

| Column       | Type           | Nullable | Default      | Notes                         |
|--------------|----------------|----------|--------------|-------------------------------|
| ORDER_ID     | INTEGER        | NO       | auto         | PK — auto-generated           |
| CUSTOMER_ID  | INTEGER        | NO       | —            | FK → CUSTOMER(CUSTOMER_ID)    |
| ORDER_DATE   | DATE           | NO       | CURRENT_DATE | Auto-set to today             |
| TOTAL_AMOUNT | NUMERIC(10,2)  | NO       | —            | CHECK: `> 0`                  |
| IS_DELETED   | BOOLEAN        | NO       | FALSE        | Soft-delete flag              |

**Constraints:** `CUSTOMER_ID` → `RESTRICT` on customer delete (cannot delete a customer with existing orders)

---

### ORDER_DETAILS

| Column          | Type          | Nullable | Default | Notes                          |
|-----------------|---------------|----------|---------|--------------------------------|
| ORDER_DETAIL_ID | INTEGER       | NO       | auto    | PK — auto-generated            |
| ORDER_ID        | INTEGER       | NO       | —       | FK → ORDERS(ORDER_ID)          |
| PRODUCT_ID      | INTEGER       | NO       | —       | FK → PRODUCT(PRODUCT_ID)       |
| QUANTITY        | INTEGER       | NO       | —       | CHECK: `> 0`                   |
| UNIT_PRICE      | NUMERIC(6,2)  | NO       | —       | CHECK: `> 0 AND <= 9999.99`    |

> `UNIT_PRICE` is captured at the time of the order — not a live reference to `PRODUCT.PRICE`. This ensures historical order accuracy even after price changes.

---

## 🔗 Relationships

```
CATEGORY   ──< CATEGORY        (self-referencing, PARENT_ID, RESTRICT)
CATEGORY   >── PRODUCT          (1 category → many products, SET NULL)
CUSTOMER   ──< ORDERS           (1 customer → many orders, RESTRICT)
ORDERS     ──< ORDER_DETAILS    (1 order → many line items)
PRODUCT    ──< ORDER_DETAILS    (1 product → many order lines)
```

---

## 📊 Business Reports / Queries

---

### 1. Daily Revenue Report

Returns total orders, total items sold, and total revenue for a specific date. Excludes soft-deleted orders.

**Returns:**

| Column           | Description                             |
|------------------|-----------------------------------------|
| `ORDER_DATE`     | The queried date                        |
| `TOTAL_ORDERS`   | Count of distinct orders on that date   |
| `TOTAL_ITEMS_SOLD` | Count of all order detail rows        |
| `TOTAL_REVENUE`  | SUM of `quantity × unit_price`          |

#### PostgreSQL

```sql
-- Daily report of total revenue for a specific date.
-- Usage: pass the date via psql variable:  \set report_date '2026-01-03'

SELECT o.order_date,
       COUNT(DISTINCT o.order_id)       AS TOTAL_ORDERS,
       COUNT(od.order_detail_id)        AS TOTAL_ITEMS_SOLD,
       SUM(od.quantity * od.unit_price) AS TOTAL_REVENUE
FROM   ORDERS o
JOIN   ORDER_DETAILS od ON o.order_id = od.order_id
WHERE  o.ORDER_DATE  = :'report_date'
  AND  o.is_deleted  = FALSE
GROUP  BY o.order_date;
```

#### MySQL

```sql
-- Daily report of total revenue for a specific date.

SET @report_date = '2026-01-03';

SELECT  o.ORDER_DATE,
        COUNT(DISTINCT o.ORDER_ID)       AS TOTAL_ORDERS,
        COUNT(od.ORDER_DETAIL_ID)        AS TOTAL_ITEMS_SOLD,
        SUM(od.QUANTITY * od.UNIT_PRICE) AS TOTAL_REVENUE
FROM    `ORDERS` o
JOIN    ORDER_DETAILS od ON o.ORDER_ID = od.ORDER_ID
WHERE   o.ORDER_DATE = @report_date
  AND   o.IS_DELETED = FALSE
GROUP   BY o.ORDER_DATE;
```

---

### 2. Monthly Top-Selling Products

Returns the top 10 products by units sold in a given calendar month. Excludes soft-deleted orders.

**Returns:**

| Column                 | Description                          |
|------------------------|--------------------------------------|
| `PRODUCT_ID`           | Product identifier                   |
| `NAME`                 | Product name                         |
| `TOTAL_QUANTITY_SOLD`  | Total units sold, descending order   |

#### PostgreSQL

```sql
-- Monthly report of the top-selling products.
-- Change the date literal to target a different month.

SELECT p.product_id,
       p.name,
       SUM(od.quantity) AS total_quantity_sold
FROM   order_details od
JOIN   product p ON od.product_id = p.product_id
JOIN   orders  o ON od.order_id   = o.order_id
WHERE  o.is_deleted = FALSE
  AND  DATE_TRUNC('month', o.order_date) = DATE_TRUNC('month', DATE '2026-03-01')
GROUP  BY p.product_id, p.name
ORDER  BY total_quantity_sold DESC
LIMIT  10;
```

#### MySQL

```sql
-- Monthly report of the top-selling products.
-- Change YEAR() and MONTH() values to target a different month.

SELECT  p.product_id,
        p.name,
        SUM(od.quantity) AS total_quantity_sold
FROM    order_details od
JOIN    product p ON od.product_id = p.product_id
JOIN    orders  o ON od.order_id   = o.order_id
WHERE   o.is_deleted = FALSE
  AND   YEAR(o.order_date)  = 2026
  AND   MONTH(o.order_date) = 3
GROUP   BY p.product_id, p.name
ORDER   BY total_quantity_sold DESC
LIMIT   10;
```

---

### 3. High-Value Customers (Last 30 Days)

Returns customers who have spent more than **$500** in the last 30 days, sorted by highest spend. Excludes soft-deleted orders.

**Returns:**

| Column         | Description                                  |
|----------------|----------------------------------------------|
| `CUSTOMER_ID`  | Customer identifier                          |
| `FIRST_NAME`   | Customer first name                          |
| `LAST_NAME`    | Customer last name                           |
| `TOTAL_SPENT`  | Aggregated order total, filtered by > $500   |

#### PostgreSQL

```sql
-- Customers who spent more than $500 in the last month.

SELECT c.customer_id,
       c.first_name,
       c.last_name,
       SUM(o.total_amount) AS total_spent
FROM   customer c
JOIN   orders o ON c.customer_id = o.customer_id
WHERE  o.is_deleted = FALSE
  AND  o.order_date >= CURRENT_DATE - INTERVAL '1 month'
GROUP  BY c.customer_id, c.first_name, c.last_name
HAVING SUM(o.total_amount) > 500
ORDER  BY total_spent DESC;
```

#### MySQL

```sql
-- Customers who spent more than $500 in the last month.

SELECT  c.customer_id,
        c.first_name,
        c.last_name,
        SUM(o.total_amount) AS total_spent
FROM    customer c
JOIN    orders o ON c.customer_id = o.customer_id
WHERE   o.is_deleted = FALSE
  AND   o.order_date >= CURRENT_DATE - INTERVAL 1 MONTH
GROUP   BY c.customer_id, c.first_name, c.last_name
HAVING  total_spent > 500
ORDER   BY total_spent DESC;
```

> **Note — PostgreSQL vs MySQL HAVING difference:**
> PostgreSQL requires repeating the aggregate in `HAVING`: `HAVING SUM(o.total_amount) > 500`  
> MySQL allows referencing the alias directly: `HAVING total_spent > 500`

---

## 🔄 PostgreSQL vs MySQL — Key Differences

| Feature                  | PostgreSQL                                    | MySQL                                         |
|--------------------------|-----------------------------------------------|-----------------------------------------------|
| Auto-increment           | `GENERATED ALWAYS AS IDENTITY (START WITH N)` | `AUTO_INCREMENT` + `ALTER TABLE ... AUTO_INCREMENT = N` |
| Boolean type             | Native `BOOLEAN` (`TRUE`/`FALSE`)             | `TINYINT(1)` (`1`/`0`)                        |
| Decimal type             | `NUMERIC(p, s)`                               | `DECIMAL(p, s)`                               |
| Date truncation          | `DATE_TRUNC('month', date)`                   | `YEAR(date)` + `MONTH(date)`                  |
| Interval syntax          | `INTERVAL '1 month'`                          | `INTERVAL 1 MONTH`                            |
| HAVING alias             | Must repeat aggregate expression              | Can reference `SELECT` alias directly         |
| psql variables           | `\set var value` → `:'var'`                   | `SET @var = value` → `@var`                   |
| Storage engine           | Built-in (no config needed)                   | Must set `ENGINE = InnoDB` for FK support      |
| Default date in column   | `DEFAULT CURRENT_DATE`                        | `DEFAULT (CURRENT_DATE)` (parentheses required in MySQL 8+) |

---



## Denormalization : 
- **create a table with specfied columns**
- **create a trigger to insert the deta in normalized table in the same time they insted in its orgiginal tables**

## 🧠 Design Decisions

**Soft-delete on `ORDERS` and `CATEGORY`**  
Rather than hard-deleting records, an `IS_DELETED` flag is set. This preserves audit history and avoids cascading deletes breaking order history. All reports filter `WHERE is_deleted = FALSE`.

**`UNIT_PRICE` stored in `ORDER_DETAILS`**  
The price at time of purchase is captured directly on the line item. This decouples historical order data from future price changes on `PRODUCT.PRICE`.

**`TOTAL_AMOUNT` denormalised on `ORDERS`**  
Storing the total on the order header avoids re-summing `ORDER_DETAILS` on every read — useful for high-frequency reporting queries.

**`RESTRICT` on customer delete**  
A customer with existing orders cannot be deleted. This protects referential integrity of order history. Deactivate customers via `IS_ACTIVE = FALSE` instead.

**`SET NULL` on category delete**  
If a category is removed, associated products are not deleted — their `CATEGORY_ID` becomes `NULL`. Products remain available; re-categorisation can happen separately.

**Self-referencing `CATEGORY`**  
The `PARENT_ID` foreign key pointing back to `CATEGORY_ID` allows unlimited nesting depth (e.g. Electronics → Phones → Smartphones) without schema changes.

---

*Generated from PostgreSQL and MySQL DDL scripts — schema version 1.0*