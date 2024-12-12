# PostgreSQL EXPLAIN 與效能調校入門

## 1. 前言：查詢效能

在資料庫設計中，查詢效能往往決定了應用程式的使用體驗。如果查詢效能差，可能導致系統反應遲緩、使用者流失並浪費資源，嚴重甚至導致 CPU 過載，資料庫崩潰。常見問題包括：

- 錯誤的查詢設計，導致全表掃描 (Seq Scan)
- 未充分利用索引 (index)
- 過多的子查詢 (Inline Subquery)
- 過多 JOIN 或不當的 CTE 使用

---

## 2. 書店資料庫範例 Schema

以下是一個基本的書店資料庫結構範例：

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    author_id INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    stock INT NOT NULL,
    published_date DATE
);

CREATE TABLE authors (
    author_id SERIAL PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100)
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255) UNIQUE
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INT NOT NULL,
    book_id INT NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL
);
```

---

## 3. 使用 EXPLAIN 與 EXPLAIN ANALYZE

`EXPLAIN` 提供查詢執行計劃，而 `EXPLAIN ANALYZE` 實際執行查詢並返回預估與實際成本，適合進一步效能分析。

### 查詢範例：查詢特定書籍資訊

```sql
EXPLAIN SELECT  FROM books WHERE title = 'PostgreSQL Performance Tuning';
```

---

## 4. 查詢計劃關鍵元素解析

- 掃描模式 (Scan Types)：

  - `Seq Scan`: 全表掃描，效能差。
  - `Index Scan`: 使用索引，提高查詢效能。
  - `Index Only Scan`: 只使用索引資料，避免讀取資料表。

- 連接策略 (Join Types)：

  - `Nested Loop`: 適合小型資料集。
  - `Hash Join`: 適合大型未排序資料。
  - `Merge Join`: 適合已排序資料。

- 成本 (Cost)：預估查詢成本，格式 `(啟動成本..總成本)`，越小越好。

---

## 5. 效能調校技巧與範例

### 5.1 善用 WHERE 條件篩選資料

#### 錯誤範例：未使用 WHERE 條件

```sql
EXPLAIN SELECT  FROM orders;
```

- 依序查詢全表，資料表變大時效能急劇下降。

#### 改進範例：使用條件篩選

```sql
EXPLAIN SELECT  FROM orders WHERE order_date >= '2024-01-01';
```

- 限制日期範圍，有效減少掃描行數。

---

### 5.2 減少 Inline Subquery

#### 錯誤範例：過多內嵌子查詢

```sql
SELECT  FROM books WHERE author_id = (
    SELECT author_id FROM authors WHERE last_name = 'Doe'
);
```

- PostgreSQL 必須先執行內嵌子查詢，然後比對外層查詢，成本較高。

#### 改進範例：使用 JOIN

```sql
SELECT b.
FROM books b
JOIN authors a ON b.author_id = a.author_id
WHERE a.last_name = 'Doe';
```

- JOIN 查詢計劃可被優化器更好地處理，成本較低。

---

### 5.3 使用 CTE： materialized 與非 materialized 的選擇

#### 非 materialized CTE (預設行為)

```sql
WITH recent_orders AS (
    SELECT  FROM orders WHERE order_date >= '2024-01-01'
)
SELECT  FROM recent_orders WHERE customer_id = 10;
```

- PostgreSQL 預設不會 materialize CTE，將查詢內嵌進主查詢，適合大多數情況。

#### 強制 materialized CTE

```sql
WITH MATERIALIZED recent_orders AS (
    SELECT  FROM orders WHERE order_date >= '2024-01-01'
)
SELECT  FROM recent_orders WHERE customer_id = 10;
```

- 當需要重複使用資料時，可考慮使用 `MATERIALIZED` 關鍵字，避免多次執行。

---

## 6. 查詢效能分析範例

### 範例：熱門作者的總銷售額查詢

```sql
EXPLAIN ANALYZE
SELECT a.first_name, a.last_name, SUM(oi.price  oi.quantity) AS total_sales
FROM authors a
JOIN books b ON a.author_id = b.author_id
JOIN order_items oi ON b.book_id = oi.book_id
GROUP BY a.author_id;
```

#### 優化建議：

- 為 `books.author_id` 與 `order_items.book_id` 建立索引。
- 若查詢經常執行，可考慮建立 materialized view。

---

## 7. 整體效能最佳實踐

### 7.1 資料庫層面調校

- 定期執行 `ANALYZE` 以更新統計資料。
- 建立適當的索引，如 `author_id`, `order_date`。
- 使用分區表管理大型資料表，如 `orders`。

### 7.2 查詢層面調校

- 避免 `SELECT `，僅選取需要的欄位。
- 善用 `JOIN`，減少內嵌子查詢。
- 避免過多 CTE，必要時使用 materialized CTE。
