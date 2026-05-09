# SQL Tasks Documentation

This README documents several SQL tasks implemented on top of the ecommerce database schema.

The examples are written for **PostgreSQL**.

---

## Database Context

The queries assume the existence of these main tables:

- `CATEGORY`
- `PRODUCT`
- `CUSTOMER`
- `ORDERS`
- `ORDER_DETAILS`

The `PRODUCT` table contains product information such as name, description, price, stock quantity, and category.

The `ORDERS` table stores order-level data such as customer, order date, and total amount.

The `ORDER_DETAILS` table stores the products included in each order.

---

# Task 1: Search Products by Keyword

## Objective

Search for all products that contain the word `camera` in either:

- product name
- product description

The search should be case-insensitive.

## SQL Query

```sql
/*
   SQL query to search for all products
   with the word "camera"
   in either the product name
   or description.
*/

SELECT *
FROM PRODUCT
WHERE NAME ILIKE '%camera%'
   OR DESCRIPTION ILIKE '%camera%';
```

## Explanation

`ILIKE` is a PostgreSQL operator used for case-insensitive pattern matching.

This query matches values like:

- `camera`
- `Camera`
- `CAMERA`
- `CaMeRa`

## Result
<img width="1572" height="608" alt="image" src="https://github.com/user-attachments/assets/68284ca9-0bb3-486a-9af4-5946b9ae475d" />

---

# Task 2: Suggest Popular Products from the Same Category

## Objective

Suggest popular products from the same category as a purchased product.

The purchased product itself must be excluded from the recommendations.

## SQL Query

```sql
/*
   Suggest popular products from the same category
   as the purchased product, excluding that product.
*/

SELECT
    p.PRODUCT_ID,
    p.NAME,
    p.DESCRIPTION,
    p.PRICE,
    p.CATEGORY_ID,
    COALESCE(SUM(od.QUANTITY), 0) AS TOTAL_SOLD
FROM PRODUCT p
LEFT JOIN ORDER_DETAILS od
    ON p.PRODUCT_ID = od.PRODUCT_ID
WHERE p.CATEGORY_ID = (
    SELECT CATEGORY_ID
    FROM PRODUCT
    WHERE PRODUCT_ID = :purchased_product_id
)
AND p.PRODUCT_ID <> :purchased_product_id
AND p.STOCK_QTY > 0
GROUP BY
    p.PRODUCT_ID,
    p.NAME,
    p.DESCRIPTION,
    p.PRICE,
    p.CATEGORY_ID
ORDER BY TOTAL_SOLD DESC
LIMIT 10;
```

## Result
**With purchased product ID = 105**
<img width="1570" height="395" alt="image" src="https://github.com/user-attachments/assets/9caf8837-b756-431c-80b1-985ecd762afb" />



The query works as follows:

1. Finds the category of the purchased product.
2. Searches for other products in the same category.
3. Excludes the purchased product itself.
4. Excludes products that are out of stock.
5. Counts total sold quantity using `ORDER_DETAILS`.
6. Orders products by popularity.

---

# Task 3: Create Sale History Table

## Objective

Create a sale history table that stores historical sales records.

Each sale history record captures:

- order ID
- order date
- customer ID
- product ID
- quantity
- unit price
- total amount
- creation timestamp

## SQL Script

```sql
CREATE TABLE SALE_HISTORY
(
    SALE_HISTORY_ID INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    ORDER_ID        INTEGER NOT NULL,
    ORDER_DATE      DATE NOT NULL,
    CUSTOMER_ID     INTEGER NOT NULL,
    PRODUCT_ID      INTEGER NOT NULL,
    QUANTITY        INTEGER NOT NULL,
    UNIT_PRICE      NUMERIC(6, 2) NOT NULL,
    TOTAL_AMOUNT    NUMERIC(10, 2) NOT NULL,
    CREATED_AT      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (ORDER_ID) REFERENCES ORDERS(ORDER_ID),
    FOREIGN KEY (CUSTOMER_ID) REFERENCES CUSTOMER(CUSTOMER_ID),
    FOREIGN KEY (PRODUCT_ID) REFERENCES PRODUCT(PRODUCT_ID)
);
```

## Explanation

`SALE_HISTORY` is used to keep a permanent record of sold products.

This is useful for:

- reporting
- analytics
- auditing
- tracking customer purchases
- tracking product sales performance

---

# Task 4: Create Indexes for Sale History

## Objective

Improve query performance on the `SALE_HISTORY` table.

## SQL Script

```sql
CREATE INDEX idx_sale_history_customer
ON SALE_HISTORY (CUSTOMER_ID);

CREATE INDEX idx_sale_history_product
ON SALE_HISTORY (PRODUCT_ID);

CREATE INDEX idx_sale_history_order_date
ON SALE_HISTORY (ORDER_DATE);
```

## Explanation

These indexes help speed up common queries such as:

- sales by customer
- sales by product
- sales by date

Example:

```sql
SELECT *
FROM SALE_HISTORY
WHERE CUSTOMER_ID = 100;
```

The index on `CUSTOMER_ID` can help this query execute faster.

---

# Task 5: Create Sale History Trigger Function

## Objective

Automatically insert a sale history record whenever a new row is inserted into `ORDER_DETAILS`.

Although the requirement mentions triggering on `ORDERS`, the correct table is `ORDER_DETAILS` because product and quantity data are stored there.

## SQL Script

```sql
CREATE OR REPLACE FUNCTION create_sale_history()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO SALE_HISTORY
    (
        ORDER_ID,
        ORDER_DATE,
        CUSTOMER_ID,
        PRODUCT_ID,
        QUANTITY,
        UNIT_PRICE,
        TOTAL_AMOUNT
    )
    SELECT
        o.ORDER_ID,
        o.ORDER_DATE,
        o.CUSTOMER_ID,
        NEW.PRODUCT_ID,
        NEW.QUANTITY,
        NEW.UNIT_PRICE,
        NEW.QUANTITY * NEW.UNIT_PRICE
    FROM ORDERS o
    WHERE o.ORDER_ID = NEW.ORDER_ID;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## Explanation

`NEW` refers to the newly inserted row in `ORDER_DETAILS`.

For every inserted order detail, the function:

1. Gets the related order from `ORDERS`.
2. Reads the order date and customer ID.
3. Reads the product, quantity, and unit price from the new `ORDER_DETAILS` row.
4. Calculates the total amount using:

```sql
NEW.QUANTITY * NEW.UNIT_PRICE
```

5. Inserts the result into `SALE_HISTORY`.

---

# Task 6: Create Trigger

## Objective

Attach the trigger function to the `ORDER_DETAILS` table.

## SQL Script

```sql
CREATE TRIGGER trg_create_sale_history
AFTER INSERT ON ORDER_DETAILS
FOR EACH ROW
EXECUTE FUNCTION create_sale_history();
```

## Explanation

This trigger runs automatically after inserting a new row into `ORDER_DETAILS`.

It is defined as:

```sql
FOR EACH ROW
```

This means the trigger runs once for every inserted order detail record.

---

# Task 7: Test the Trigger

## Objective

Insert a new order and order detail to verify that the trigger automatically inserts into `SALE_HISTORY`.

## SQL Script

```sql
-- Create order
INSERT INTO ORDERS
(
    CUSTOMER_ID,
    ORDER_DATE,
    TOTAL_AMOUNT
)
VALUES
(
    100,
    CURRENT_DATE,
    2500.00
);

-- Insert product into the latest created order
INSERT INTO ORDER_DETAILS
(
    ORDER_ID,
    PRODUCT_ID,
    QUANTITY,
    UNIT_PRICE
)
VALUES
(
    (SELECT MAX(ORDER_ID) FROM ORDERS),
    105,
    2,
    500.00
);

-- Verify trigger fired
SELECT *
FROM SALE_HISTORY
ORDER BY SALE_HISTORY_ID DESC;
```

## Expected Result

After inserting into `ORDER_DETAILS`, a new row should automatically appear in `SALE_HISTORY`.

Example result:

| SALE_HISTORY_ID | ORDER_ID | ORDER_DATE | CUSTOMER_ID | PRODUCT_ID | QUANTITY | UNIT_PRICE | TOTAL_AMOUNT |
|---|---|---|---|---|---|---|---|
| 1 | 100 | 2026-05-09 | 100 | 105 | 2 | 500.00 | 1000.00 |

---

# Important Notes

## Why the Trigger Is on `ORDER_DETAILS`

The `ORDERS` table does not contain product or quantity information.

Product and quantity are stored in `ORDER_DETAILS`.

Therefore, this trigger should be placed on `ORDER_DETAILS`, not `ORDERS`.

## Better Production Design

In a production system, avoid this pattern:

```sql
SELECT MAX(ORDER_ID) FROM ORDERS
```

It can be unsafe when multiple users insert orders at the same time.

A better approach is to use `RETURNING`:

```sql
INSERT INTO ORDERS
(
    CUSTOMER_ID,
    ORDER_DATE,
    TOTAL_AMOUNT
)
VALUES
(
    100,
    CURRENT_DATE,
    2500.00
)
RETURNING ORDER_ID;
```

Then use the returned `ORDER_ID` when inserting into `ORDER_DETAILS`.

---

# Full Execution Order

Run the scripts in this order:

1. Create `SALE_HISTORY` table.
2. Create indexes.
3. Create trigger function.
4. Create trigger.
5. Insert test order.
6. Insert test order details.
7. Query `SALE_HISTORY`.

---

# Summary

This implementation provides:

- case-insensitive product search
- popular product recommendations
- sale history table
- performance indexes
- automatic sale history creation using triggers
- test insert statements

