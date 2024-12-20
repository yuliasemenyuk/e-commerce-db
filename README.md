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
   "status" order_status NOT NULL DEFAULT 'preparing',
   "created_at" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE "orders"."status_history" (
   "id" INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   "status" order_status NOT NULL,
   "changed_at" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
   "order_id" integer NOT NULL
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

-- Для швидкого пошуку замовлень за статусом та датою створення
-- Used for: SELECT ... FROM orders WHERE status = ? ORDER BY created_at;
CREATE INDEX "idx_orders_status" ON "orders"."orders" ("status", "created_at");

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