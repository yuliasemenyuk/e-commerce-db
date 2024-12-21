Діаграма - https://dbdiagram.io/d/E-commerce-Diagram-676480e46ae6af4766bf8da6

## Бізнес процеси та загальна логіка

### Структура даних
База даних розділена на 3 основні схеми:

1. **auth**
  - Керування користувачами (реєстрація, автентифікація)
  - Збереження обраних товарів користувачів

2. **products** 
  - Управління товарами
  - Підтримка множинних категорій та знижок для товарів
  - Зберігання зображень товарів

3. **orders**
  - Обробка замовлень
  - Відстеження статусів
  - Управління транзакціями

### Основні бізнес-процеси

#### 1. Робота з товарами
- Товар може належати до кількох категорій
- Може мати декілька зображень
- Підтримує систему знижок
- Відслідковується залишок на складі

#### 2. Процес замовлення
1. **Створення замовлення**:
  - Перевірка наявності товарів
  - Створення запису в orders
  - Додавання товарів до order_items
  - Оновлення залишків на складі
  - Встановлення початкового статусу 'preparing'

2. **Життєвий цикл замовлення**:
preparing -> paid -> sent -> delivered
та canceled окремо
- Кожна зміна статусу фіксується в status_history
- Автоматичне скасування неоплачених замовлень через 3 дні

3. **Фінансові операції**:
- Оплата замовлення (payment)
- Повернення коштів (refund)
- Додаткові послуги (additional_service)

#### 3. Система обраного
- Користувачі можуть додавати товари до списку обраного
- Один товар може бути в списку обраного у різних користувачів

### Обмеження та валідації
1. **Товари**:
- Ціна завжди більше 0
- Кількість на складі не може бути від'ємною
- Знижка в межах 0-100%

2. **Замовлення**:
- Кількість товарів більше 0
- Статуси змінюються в певній послідовності
- Зберігається повна історія змін статусів

3. **Користувачі**:
- Унікальні email адреси
- Валідація формату email


~~~sql
CREATE SCHEMA "auth";
CREATE SCHEMA "products";
CREATE SCHEMA "orders";

CREATE TYPE "category_name" AS ENUM (
   'electronics',
   'clothing',
   'food',
   'furniture',
   'household'
);

CREATE TYPE "order_status" AS ENUM (
   'preparing',
   'paid',
   'sent',
   'delivered',
   'canceled'
);

CREATE TYPE "transaction_type" AS ENUM (
   'payment',
   'refund',
   'additional_service'
);

CREATE TABLE "products"."items" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "title" varchar(100) NOT NULL,
   "description" text,
   "price" decimal(10,2) NOT NULL CHECK ("price" >= 0),
   "left_in_stock" integer NOT NULL DEFAULT 0 CHECK ("left_in_stock" >= 0),
   "created_at" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
   "updated_at" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE "products"."categories" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "name" category_name UNIQUE NOT NULL
);

CREATE TABLE "products"."discounts" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "value" decimal(5,2) NOT NULL CHECK ("value" > 0 AND "value" <= 100)
);

CREATE TABLE "products"."images" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "url" varchar(500) NOT NULL,
   "filename" varchar(100) NOT NULL,
   "type" varchar(4) NOT NULL,
   "item_id" integer NOT NULL
);

CREATE TABLE "auth"."users" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "username" varchar(50) NOT NULL,
   "email" varchar(200) UNIQUE NOT NULL CHECK ("email" ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
   "password" varchar(200) NOT NULL,
   "created_at" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
   "updated_at" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE "auth"."user_favorites" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "user_id" integer NOT NULL,
   "item_id" integer NOT NULL,
   "created_at" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE "orders"."orders" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "user_id" integer NOT NULL,
   "created_at" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
   "status" INTEGER
);

CREATE TABLE "orders"."status_history" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "status" order_status NOT NULL,
   "changed_at" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
   "order_id" integer NOT NULL,
   CONSTRAINT fk_order_id FOREIGN KEY ("order_id") REFERENCES "orders"."orders"("id")
);

CREATE TABLE "orders"."transactions" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "order_id" integer NOT NULL,
   "transaction_type" transaction_type NOT NULL,
   "amount" decimal(10,2) NOT NULL,
   "transaction_date" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE "products"."item_categories" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "item_id" integer NOT NULL,
   "category_id" integer NOT NULL
);

CREATE TABLE "products"."item_discounts" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "item_id" integer NOT NULL,
   "discount_id" integer NOT NULL
);

CREATE TABLE "orders"."order_items" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "order_id" integer NOT NULL,
   "item_id" integer NOT NULL,
   "quantity" integer NOT NULL DEFAULT 1 CHECK ("quantity" > 0)
);

-- Для пошуку замовлень за статусом, в т.ч. для процедури сheck_and_update_order_status()
-- Used for: SELECT ... FROM status_history WHERE status = 'preparing';
CREATE INDEX "idx_order_history_status" ON "orders"."status_history" ("status");

-- Для пошуку замовлень за датою зміни статусу в т.ч. для процедури сheck_and_update_order_status()
-- Used for: SELECT ... FROM status_history WHERE changed_at < some_date
CREATE INDEX "idx_order_changed_at" ON "orders"."status_history" ("changed_at");

-- Для пошуку товарів за назвою та залишками на складі
-- Used for: SELECT ... FROM items WHERE title LIKE ? AND left_in_stock > 0;
CREATE INDEX "idx_items_search" ON "products"."items" ("title", "left_in_stock");

-- Для сортування товарів за датою додавання
-- Used for: SELECT ... FROM items ORDER BY created_at;
CREATE INDEX "idx_items_date" ON "products"."items" ("created_at");

-- Для швидкого пошуку користувача за emai, автентифікації
-- Used for: SELECT ... FROM users WHERE email = ?;
CREATE UNIQUE INDEX "idx_users_email" ON "auth"."users" ("email");

-- Для сортування улюблених товарів за датою додавання
-- Used for: SELECT ... FROM user_favorites WHERE user_id = ? ORDER BY created_at;
CREATE INDEX "idx_favorites_date" ON "auth"."user_favorites" ("created_at");

-- Для запитів на історію замовлень користувача
-- Used for: SELECT ... FROM orders WHERE user_id = ? ORDER BY created_at;
CREATE INDEX "idx_orders_user_history" ON "orders"."orders" ("user_id", "created_at");

-- Для швидкого пошуку замовлень за статусом та датою
-- Used in check_and_update_order_status procedure
CREATE INDEX "idx_status_history_lookup" ON "orders"."status_history" ("status", "changed_at");

-- Для запитів на історію фінансових транзакцій замовлення
-- Used for: SELECT ... FROM transactions WHERE order_id = ? ORDER BY transaction_date;
CREATE INDEX "idx_transactions_order" ON "orders"."transactions" ("order_id", "transaction_date");

-- Для швидкого пошуку транзакцій за типом та датою
-- Used for: SELECT ... FROM transactions WHERE transaction_type = ? ORDER BY transaction_date;
CREATE INDEX "idx_transactions_reporting" ON "orders"."transactions" ("transaction_type", "transaction_date");

ALTER TABLE "products"."images" 
   ADD CONSTRAINT "images_item_id_fkey" 
   FOREIGN KEY ("item_id") REFERENCES "products"."items" ("id");

ALTER TABLE "auth"."user_favorites" 
   ADD CONSTRAINT "user_favorites_user_id_fkey" 
   FOREIGN KEY ("user_id") REFERENCES "auth"."users" ("id");

ALTER TABLE "auth"."user_favorites" 
   ADD CONSTRAINT "user_favorites_item_id_fkey" 
   FOREIGN KEY ("item_id") REFERENCES "products"."items" ("id");

ALTER TABLE "orders"."orders" 
   ADD CONSTRAINT "orders_user_id_fkey" 
   FOREIGN KEY ("user_id") REFERENCES "auth"."users" ("id");

ALTER TABLE "orders"."status_history" 
   ADD CONSTRAINT "status_history_order_id_fkey" 
   FOREIGN KEY ("order_id") REFERENCES "orders"."orders" ("id");

ALTER TABLE "orders"."transactions" 
   ADD CONSTRAINT "transactions_order_id_fkey" 
   FOREIGN KEY ("order_id") REFERENCES "orders"."orders" ("id");

ALTER TABLE "products"."item_categories" 
   ADD CONSTRAINT "item_categories_item_id_fkey" 
   FOREIGN KEY ("item_id") REFERENCES "products"."items" ("id");

ALTER TABLE "products"."item_categories" 
   ADD CONSTRAINT "item_categories_category_id_fkey" 
   FOREIGN KEY ("category_id") REFERENCES "products"."categories" ("id");

ALTER TABLE "products"."item_discounts" 
   ADD CONSTRAINT "item_discounts_item_id_fkey" 
   FOREIGN KEY ("item_id") REFERENCES "products"."items" ("id");

ALTER TABLE "products"."item_discounts" 
   ADD CONSTRAINT "item_discounts_discount_id_fkey" 
   FOREIGN KEY ("discount_id") REFERENCES "products"."discounts" ("id");

ALTER TABLE "orders"."order_items" 
   ADD CONSTRAINT "order_items_order_id_fkey" 
   FOREIGN KEY ("order_id") REFERENCES "orders"."orders" ("id");

ALTER TABLE "orders"."order_items" 
   ADD CONSTRAINT "order_items_item_id_fkey" 
   FOREIGN KEY ("item_id") REFERENCES "products"."items" ("id");

-- Процедура для скасування неоплачених замовлень після 3 днів
CREATE OR REPLACE FUNCTION check_and_update_order_status()
RETURNS VOID AS $$
DECLARE
    order_rec RECORD;
    new_status_id INTEGER;
BEGIN
    FOR order_rec IN
        SELECT o.id
        FROM "orders"."orders" o
        JOIN "orders"."status_history" s ON o.status = s.id
        WHERE s.status = 'preparing'
          AND s.changed_at < NOW() - INTERVAL '3 days'
    LOOP
        INSERT INTO "orders"."status_history" (status, changed_at, order_id)
        VALUES ('canceled', CURRENT_TIMESTAMP, order_rec.id)
        RETURNING id INTO new_status_id;

        UPDATE "orders"."orders"
        SET "status" = new_status_id
        WHERE id = order_rec.id;
    END LOOP;
END;
$$ LANGUAGE plpgsql;


-- SQL-запит з використанням транзакції для сценарію: створення нового замовлення з внесенням платежу
BEGIN;
    DO $$ 
    DECLARE
        new_order_id INTEGER;
        initial_status_id INTEGER;
        paid_status_id INTEGER;
        total_amount DECIMAL(10,2);

        is_enough_stock BOOLEAN;
    BEGIN
        -- 1. Перевіряємо наявність товарів на складі
        SELECT 
        (SELECT COUNT(*) > 0 AND MIN(left_in_stock) >= 2 
        FROM products.items 
        WHERE id = 1) AND
        (SELECT COUNT(*) > 0 AND MIN(left_in_stock) >= 1 
        FROM products.items 
        WHERE id = 2)
        INTO is_enough_stock;

        IF NOT is_enough_stock THEN
            RAISE EXCEPTION 'Insufficient stock for items or item does not exist';
        END IF;

        -- 2. Створюємо нове замовлення та додаємо товари
        INSERT INTO orders.orders (user_id)
        VALUES (1)  
        RETURNING id INTO new_order_id;

        UPDATE products.items 
        SET left_in_stock = left_in_stock - 2
        WHERE id = 1;

        INSERT INTO orders.order_items (order_id, item_id, quantity)
        VALUES (new_order_id, 1, 2);

        UPDATE products.items 
        SET left_in_stock = left_in_stock - 1
        WHERE id = 2;

        INSERT INTO orders.order_items (order_id, item_id, quantity)
        VALUES (new_order_id, 2, 1);

        -- 3. Створюємо початковий статус 'preparing'
        INSERT INTO orders.status_history (order_id, status)
        VALUES (new_order_id, 'preparing')
        RETURNING id INTO initial_status_id;

        -- 4. Оновлюємо замовлення з початковим статусом
        UPDATE orders.orders 
        SET status = initial_status_id
        WHERE id = new_order_id;

        -- 5. Рахуємо загальну суму замовлення
        SELECT SUM(i.price * oi.quantity) INTO total_amount
        FROM orders.order_items oi
        JOIN products.items i ON oi.item_id = i.id
        WHERE oi.order_id = new_order_id;

        -- 6. Додаємо статус 'paid'
        INSERT INTO orders.status_history (order_id, status)
        VALUES (new_order_id, 'paid')
        RETURNING id INTO paid_status_id;

        -- 7. Оновлюємо статус замовлення на 'paid'
        UPDATE orders.orders 
        SET status = paid_status_id
        WHERE id = new_order_id;

        -- 8. Створюємо транзакцію оплати
        INSERT INTO orders.transactions (
            order_id,
            transaction_type,
            amount
        ) VALUES (
            new_order_id,
            'payment',
            total_amount
        );

    END $$;
COMMIT;

--Seeds generation:
--Users
INSERT INTO "auth"."users" (username, email, password, created_at, updated_at)
SELECT 
    'user-' || id,
    'user-' || id || '@example.com',
    'securepassword',
    CURRENT_TIMESTAMP,
    CURRENT_TIMESTAMP
FROM 
    generate_series(1, 100) AS s(id);  

--Items
-- додаємо категорію та генеруємо товари, що до неї належать,
-- *Припускаємо, що категорія товару - обовʼязкова, зображення і знижка - не обовʼязкові
DO $$
DECLARE
    category_id INTEGER;
BEGIN
    -- Step 1: Create a new category
    INSERT INTO "products"."categories" (name)
    VALUES ('food')
    RETURNING id INTO category_id;

    INSERT INTO "products"."items" (title, description, price, left_in_stock, created_at, updated_at)
    SELECT 
        'Product ' || id,  
        'Description for product ' || id,
        ROUND((RANDOM() * 100)::numeric, 2),
        FLOOR(RANDOM() * 1000)::INTEGER,
        CURRENT_TIMESTAMP,
        CURRENT_TIMESTAMP
    FROM 
        generate_series(1, 10000) AS s(id);

    INSERT INTO "products"."item_categories" (item_id, category_id)
    SELECT 
        id, category_id 
    FROM 
        "products"."items";
    RAISE NOTICE 'Linked products to category ID: %', category_id;  

END $$;



--РІЗНИЦЯ EXPLAIN та EXPLAIN ANALYZE:
Обидві команди - EXPLAIN та EXPLAIN ANALYZE - використовуються для аналізу виконання запитів. 
Але EXPLAIN показує план виконання запиту без його запуску.
Це дозволяє оцінити ефективність запиту заздалегідь.
EXPLAIN ANALYZE виконує запит і показує реальні результати: 
час виконання, використані ресурси та інші метрики.