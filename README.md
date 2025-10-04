# PostgreSQL Tutorial - Полное руководство для начинающих

## Содержание
1. [Основы PostgreSQL](#основы-postgresql)
2. [Подключение к базе данных](#подключение-к-базе-данных)
3. [Основные команды psql](#основные-команды-psql)
4. [Работа с базами данных](#работа-с-базами-данных)
5. [Переключение между базами данных](#переключение-между-базами-данных---подробный-tutorial)
6. [Работа с таблицами](#работа-с-таблицами)
7. [CRUD операции](#crud-операции)
8. [Принципы проектирования баз данных](#принципы-проектирования-баз-данных)
9. [Нормализация и денормализация](#нормализация-и-денормализация)
10. [JOIN операции - Полный гид](#join-операции---полный-гид)
11. [Типы данных](#типы-данных)
12. [Индексы и производительность](#индексы-и-производительность)
13. [Транзакции и ACID](#транзакции-и-acid)
14. [Хранимые процедуры и триггеры](#хранимые-процедуры-и-триггеры)
15. [Оконные функции](#оконные-функции)
16. [CTE и рекурсивные запросы](#cte-и-рекурсивные-запросы)
17. [JSON и NoSQL возможности](#json-и-nosql-возможности)
18. [Партиционирование](#партиционирование)
19. [Репликация и масштабирование](#репликация-и-масштабирование)
20. [Мониторинг и диагностика](#мониторинг-и-диагностика)
21. [Пользователи и права доступа](#пользователи-и-права-доступа)
22. [Полезные команды](#полезные-команды)
23. [Best Practices для Senior разработчиков](#best-practices-для-senior-разработчиков)
24. [Ссылки для дальнейшего изучения](#ссылки-для-дальнейшего-изучения)
25. [Работа с пользователями](#работа-с-пользователями)
26. [Миграции](#миграции)
## Основы PostgreSQL

PostgreSQL (часто называемый "Postgres") — это мощная объектно-реляционная база данных с открытым исходным кодом.

### Ключевые особенности:
- **ACID-совместимость** (Atomicity, Consistency, Isolation, Durability)
- **Расширяемость** (поддержка пользовательских типов данных, функций)
- **Стандарты SQL** (высокое соответствие стандартам SQL)
- **JSON поддержка** (нативная поддержка JSON и JSONB)

## Подключение к базе данных

### Через командную строку (psql):

```bash
# Подключение к базе данных postgres (по умолчанию)
psql -d postgres

# Подключение с указанием пользователя
psql -U username -d database_name

# Подключение к удаленной базе
psql -h hostname -p 5432 -U username -d database_name

# Подключение с паролем (будет запрошен)
psql -U username -d database_name -W
```

### Строка подключения (Connection String):
```
postgresql://username:password@localhost:5432/database_name
```

### Работа с пользователями

Ниже кратко показано, как создавать пользователей PostgreSQL с паролем и без, как менять пароль и как создать базу данных, принадлежащую новому пользователю. Примеры универсальны для macOS/Unix. Замените плейсхолдеры в угловых скобках на ваши значения.

Предпосылка: используйте роль с административными правами (например, `<SUPERUSER>` — это может быть `postgres` или ваш локальный пользователь, который обладает правами на создание БД/ролей). Для надёжности добавляем `-v ON_ERROR_STOP=1`, чтобы psql выходил при ошибке.

1) Пользователь БЕЗ пароля
- SQL:
```
CREATE USER appuser;
```
- Подходит, если локальная аутентификация настроена через peer/trust (см. pg_hba.conf). Иначе подключения могут требовать пароль.
- Connection string (без пароля):
```
postgres://appuser@localhost:5432/postgress_guide?schema=public
```

2) Пользователь С паролем (рекомендовано)
- Сгенерируйте пароль в переменную окружения (не печатаем его в консоль):
```
PGUSER_PASSWORD=$(openssl rand -base64 24)
```
- Создайте пользователя:
```
psql -h localhost -U <SUPERUSER> -d postgres -v ON_ERROR_STOP=1 \
  -c "CREATE USER appuser WITH LOGIN PASSWORD '${PGUSER_PASSWORD}';"
```
- Создайте базу данных и назначьте владельца (см. п.4 для альтернатив):
```
psql -h localhost -U <SUPERUSER> -d postgres -v ON_ERROR_STOP=1 \
  -c "CREATE DATABASE postgress_guide OWNER appuser;"
```
- Разрешения по умолчанию в public-схеме обычно достаточны; при необходимости:
```
psql -h localhost -U <SUPERUSER> -d postgress_guide -v ON_ERROR_STOP=1 \
  -c "GRANT ALL ON SCHEMA public TO appuser;"
```
- Connection string (с паролем):
```
postgres://appuser:{{PGUSER_PASSWORD}}@localhost:5432/postgress_guide?schema=public
```
Если в пароле есть спецсимволы (`@`, `:`, `/`, `#`, и т.д.), URL-кодируйте пароль (например, `@` → `%40`).

3) Смена пароля существующему пользователю
- Сгенерируйте новый пароль:
```
NEW_PGUSER_PASSWORD=$(openssl rand -base64 24)
```
- Измените пароль:
```
psql -h localhost -U <SUPERUSER> -d postgres -v ON_ERROR_STOP=1 \
  -c "ALTER USER appuser WITH PASSWORD '${NEW_PGUSER_PASSWORD}';"
```
- Обновите переменную окружения в `web/.env`:
```
DATABASE_URL="postgres://appuser:{{NEW_PGUSER_PASSWORD}}@localhost:5432/postgress_guide?schema=public"
```

4) Создание базы данных от имени нового пользователя
- Вариант A (через суперпользователя; назначение владельца при создании):
```
psql -h localhost -U <SUPERUSER> -d postgres -v ON_ERROR_STOP=1 \
  -c "CREATE DATABASE postgress_guide OWNER appuser;"
```
- Вариант B (если у appuser есть право CREATEDB, создайте БД от его имени):
```
psql -h localhost -U <SUPERUSER> -d postgres -v ON_ERROR_STOP=1 \
  -c "ALTER ROLE appuser CREATEDB;"
createdb -h localhost -U appuser postgress_guide
```
Connection string для обоих вариантов тот же, т.к. владелец — `appuser`:
```
# без пароля
postgres://appuser@localhost:5432/postgress_guide?schema=public

# с паролем
postgres://appuser:{{PGUSER_PASSWORD}}@localhost:5432/postgress_guide?schema=public
```

5) Шпаргалка по connection string
- Общий шаблон:
```
postgres://USERNAME[:PASSWORD]@HOST:PORT/DBNAME?schema=public
```
- Примеры:
```
# Локально без пароля (peer/trust)
postgres://maximus@localhost:5432/postgress_guide?schema=public

# Локально с паролем (пароль храните в .env)
postgres://pguser:{{PGUSER_PASSWORD}}@localhost:5432/postgress_guide?schema=public
```

6) Удаление пользователя
Удалять пользователя в PostgreSQL безопаснее по шагам, чтобы не остались «висящие» объекты и активные сессии.

- Остановить активные сессии пользователя (подключён к любой БД):
```
psql -h localhost -U <SUPERUSER> -d postgres -v ON_ERROR_STOP=1 \
  -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE usename = 'appuser';"
```

- Переназначить (или удалить) объекты, принадлежащие пользователю, в каждой нужной базе данных:
```
# Переназначить все объекты на суперпользователя (или другого владельца)
psql -h localhost -U <SUPERUSER> -d postgress_guide -v ON_ERROR_STOP=1 \
  -c "REASSIGN OWNED BY appuser TO <SUPERUSER>;"

# Удалить все привилегии и объекты, связанные с ролью в текущей БД
psql -h localhost -U <SUPERUSER> -d postgress_guide -v ON_ERROR_STOP=1 \
  -c "DROP OWNED BY appuser;"
```
Примечание: REASSIGN/DROP OWNED работают в рамках конкретной базы. Выполните эти команды в каждой БД, где у пользователя есть объекты/привилегии.

- Если пользователь владеет базой данных, перед удалением роли смените владельца базы (или удалите базу):
```
# Сменить владельца БД
psql -h localhost -U <SUPERUSER> -d postgres -v ON_ERROR_STOP=1 \
  -c "ALTER DATABASE postgress_guide OWNER TO <SUPERUSER>;"

# Либо удалить БД, если она больше не нужна
psql -h localhost -U <SUPERUSER> -d postgres -v ON_ERROR_STOP=1 \
  -c "DROP DATABASE postgress_guide;"
```

- Удалить роль:
```
psql -h localhost -U <SUPERUSER> -d postgres -v ON_ERROR_STOP=1 \
  -c "DROP ROLE appuser;"
```

Советы безопасности:
- Сначала убедитесь, что роль не используется приложениями (остановите сервисы или смените креды),
- Сделайте бэкап важных данных,
- Выполняйте REASSIGN OWNED в каждой базе, где роль что-то создавала.

## Основные команды psql

### Meta-команды (начинаются с \):

```sql
-- Показать все базы данных
\l

-- Подключиться к другой базе данных
\c database_name

-- Показать все таблицы в текущей базе
\dt

-- Показать схему таблицы
\d table_name

-- Показать все схемы
\dn

-- Показать всех пользователей
\du

-- Показать текущее подключение
\conninfo

-- Выйти из psql
\q

-- Справка по командам
\?

-- Справка по SQL
\h

-- Справка по конкретной команде
\h CREATE TABLE
```

## Работа с базами данных

## Переключение между базами данных - Подробный Tutorial

Этот раздел покажет пошагово, как переключаться между базами данных в PostgreSQL.

### Шаг 1: Подключение к PostgreSQL

Сначала подключимся к PostgreSQL. По умолчанию подключаемся к системной базе `postgres`:

```bash
# Подключение к системной базе данных
psql -d postgres

# Или с указанием пользователя (если требуется)
psql -U username -d postgres
```

**Результат:**
```
psql (15.4)
Type "help" for help.

postgres=#
```

### Шаг 2: Просмотр всех доступных баз данных

Посмотрим какие базы данных существуют:

```sql
-- Meta-команда для просмотра баз
\l

-- Или SQL запрос
SELECT datname, datowner, encoding FROM pg_database;
```

**Пример результата:**
```
                              List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    
-----------+----------+----------+-------------+-------------
 myapp     | username | UTF8     | en_US.UTF-8 | en_US.UTF-8
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8
 testdb    | username | UTF8     | en_US.UTF-8 | en_US.UTF-8
```

### Шаг 3: Создание тестовых баз данных для практики

Давайте создадим несколько баз данных для практики переключения:

```sql
-- Создаем базы данных для демонстрации
CREATE DATABASE shop;
CREATE DATABASE blog;
CREATE DATABASE analytics;
```

**Результат:**
```
CREATE DATABASE
CREATE DATABASE
CREATE DATABASE
```

### Шаг 4: Переключение между базами данных - Основные способы

#### Способ 1: Использование \c команды (рекомендуется)

```sql
-- Переключиться на базу shop
\c shop

-- Переключиться на базу blog с указанием пользователя
\c blog username

-- Переключиться на базу analytics на другом хосте
\c analytics username hostname
```

**Пример успешного переключения:**
```
postgres=# \c shop
You are now connected to database "shop" as user "username".
shop=#
```

#### Способ 2: Выход и новое подключение

```bash
# Выход из текущего подключения
\q

# Подключение к нужной базе данных
psql -d shop
```

### Шаг 5: Проверка текущего подключения

Всегда можно проверить к какой базе мы подключены:

```sql
-- Показать информацию о текущем подключении
\conninfo

-- Или SQL запрос
SELECT current_database(), current_user, inet_server_addr(), inet_server_port();
```

**Пример результата:**
```
You are connected to database "shop" as user "username" via socket in "/tmp" at port "5432".
```

### Шаг 6: Практическая демонстрация с созданием таблиц

Давайте создадим разные таблицы в разных базах, чтобы увидеть изоляцию данных:

#### В базе shop:
```sql
-- Убедимся что мы в базе shop
\c shop

-- Создаем таблицу товаров
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Добавляем тестовые данные
INSERT INTO products (name, price) VALUES 
    ('iPhone 15', 999.99),
    ('MacBook Pro', 2499.99);

-- Проверяем данные
SELECT * FROM products;
```

#### В базе blog:
```sql
-- Переключаемся на blog
\c blog

-- Создаем таблицу постов
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content TEXT,
    published_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Добавляем тестовые данные
INSERT INTO posts (title, content) VALUES 
    ('Первый пост', 'Содержание первого поста'),
    ('Второй пост', 'Содержание второго поста');

-- Проверяем данные
SELECT * FROM posts;
```

#### В базе analytics:
```sql
-- Переключаемся на analytics
\c analytics

-- Создаем таблицу событий
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_name VARCHAR(50) NOT NULL,
    user_id INTEGER,
    event_data JSONB,
    occurred_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Добавляем тестовые данные
INSERT INTO events (event_name, user_id, event_data) VALUES 
    ('page_view', 123, '{"page": "/home", "duration": 30}'),
    ('button_click', 456, '{"button": "buy_now", "product_id": 789}');

-- Проверяем данные
SELECT * FROM events;
```

### Шаг 7: Проверка изоляции данных между базами

Теперь убедимся, что таблицы изолированы между базами:

```sql
-- В базе shop
\c shop
\dt  -- Покажет только таблицу products

-- В базе blog  
\c blog
\dt  -- Покажет только таблицу posts

-- В базе analytics
\c analytics
\dt  -- Покажет только таблицу events
```

### Шаг 8: Переключение с сохранением истории команд

По умолчанию psql сохраняет историю команд отдельно для каждой базы:

```sql
-- В базе shop выполним команду
\c shop
SELECT COUNT(*) FROM products;

-- Переключимся на blog и выполним команду
\c blog
SELECT COUNT(*) FROM posts;

-- Вернемся в shop и нажмем стрелку вверх
\c shop
-- Стрелка вверх покажет предыдущие команды для базы shop
```

### Шаг 9: Переключение в скриптах и автоматизация

#### Создание скрипта с переключением баз:

```sql
-- Файл switch_databases.sql
\echo 'Подключаемся к базе shop'
\c shop
SELECT 'Товары в shop:', COUNT(*) FROM products;

\echo 'Подключаемся к базе blog'
\c blog  
SELECT 'Посты в blog:', COUNT(*) FROM posts;

\echo 'Подключаемся к базе analytics'
\c analytics
SELECT 'События в analytics:', COUNT(*) FROM events;
```

#### Запуск скрипта:
```bash
psql -d postgres -f switch_databases.sql
```

### Шаг 10: Полезные команды для работы с множественными базами

#### Получение информации о всех базах:
```sql
-- Размеры всех баз данных
SELECT 
    datname as database_name,
    pg_size_pretty(pg_database_size(datname)) as size,
    (SELECT count(*) FROM pg_stat_activity WHERE datname = pg_database.datname) as connections
FROM pg_database 
WHERE datistemplate = false
ORDER BY pg_database_size(datname) DESC;
```

#### Подключение к нескольким базам одновременно:
```bash
# Открыть несколько терминалов с разными базами
psql -d shop      # Терминал 1
psql -d blog      # Терминал 2  
psql -d analytics # Терминал 3
```

### Шаг 11: Распространенные ошибки и решения

#### Ошибка: database "dbname" does not exist
```sql
-- Проверьте существующие базы
\l

-- Создайте базу если нужно
CREATE DATABASE dbname;
```

#### Ошибка: permission denied for database
```sql
-- Проверьте права доступа
\l

-- Дайте права пользователю
GRANT ALL PRIVILEGES ON DATABASE dbname TO username;
```

#### База данных используется другими пользователями:
```sql
-- Посмотреть активные подключения
SELECT pid, usename, datname, state, query_start 
FROM pg_stat_activity 
WHERE datname = 'database_name';

-- Завершить подключения (осторожно!)
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE datname = 'database_name' AND pid <> pg_backend_pid();
```

### Шаг 12: Лучшие практики

1. **Используйте осмысленные имена** для баз данных:
   ```sql
   CREATE DATABASE ecommerce_prod;
   CREATE DATABASE ecommerce_test;
   CREATE DATABASE analytics_warehouse;
   ```

2. **Проверяйте текущее подключение** перед выполнением важных операций:
   ```sql
   \conninfo
   SELECT current_database();
   ```

3. **Используйте разные пользователи** для разных баз:
   ```sql
   CREATE USER shop_user WITH PASSWORD 'shop_pass';
   CREATE USER blog_user WITH PASSWORD 'blog_pass';
   GRANT ALL PRIVILEGES ON DATABASE shop TO shop_user;
   GRANT ALL PRIVILEGES ON DATABASE blog TO blog_user;
   ```

4. **Создавайте резервные копии** перед переключением:
   ```bash
   pg_dump -U username -d database_name > backup_$(date +%Y%m%d).sql
   ```

### Краткая справка команд переключения:

| Команда | Описание | Пример |
|---------|----------|--------|
| `\c dbname` | Переключиться на базу | `\c shop` |
| `\c dbname user` | Переключиться с другим пользователем | `\c shop admin` |
| `\c dbname user host` | Переключиться на другой хост | `\c shop user localhost` |
| `\conninfo` | Показать текущее подключение | `\conninfo` |
| `\l` | Показать все базы | `\l` |
| `SELECT current_database();` | Текущая база (SQL) | Возвращает имя базы |

---

### Создание базы данных:
```sql
-- Создать базу данных
CREATE DATABASE my_database;

-- Создать базу данных с параметрами
CREATE DATABASE my_database
    OWNER = username
    ENCODING = 'UTF8'
    LOCALE = 'en_US.UTF-8';
```

### Удаление базы данных:
```sql
-- Удалить базу данных
DROP DATABASE my_database;

-- Удалить только если существует
DROP DATABASE IF EXISTS my_database;
```

### Просмотр баз данных:
```sql
-- Показать все базы данных
SELECT datname FROM pg_database WHERE datistemplate = false;
```

## Работа с таблицами

### Создание таблицы:
```sql
-- Простое создание таблицы
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    age INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Создание таблицы с внешним ключом
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    total DECIMAL(10,2),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Изменение таблицы:
```sql
-- Добавить столбец
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Удалить столбец
ALTER TABLE users DROP COLUMN phone;

-- Изменить тип столбца
ALTER TABLE users ALTER COLUMN age TYPE BIGINT;

-- Переименовать столбец
ALTER TABLE users RENAME COLUMN name TO full_name;

-- Переименовать таблицу
ALTER TABLE users RENAME TO customers;
```

### Удаление таблицы:
```sql
-- Удалить таблицу
DROP TABLE users;

-- Удалить только если существует
DROP TABLE IF EXISTS users;

-- Удалить с каскадным удалением зависимостей
DROP TABLE users CASCADE;
```

## CRUD операции

### CREATE (Создание/Вставка):
```sql
-- Вставка одной записи
INSERT INTO users (name, email, age) 
VALUES ('John Doe', 'john@example.com', 30);

-- Вставка нескольких записей
INSERT INTO users (name, email, age) VALUES 
    ('Jane Smith', 'jane@example.com', 25),
    ('Bob Johnson', 'bob@example.com', 35),
    ('Alice Brown', 'alice@example.com', 28);

-- Вставка с возвратом данных
INSERT INTO users (name, email, age) 
VALUES ('Mike Wilson', 'mike@example.com', 32)
RETURNING id, created_at;
```

### READ (Чтение/Выборка):
```sql
-- Простая выборка всех записей
SELECT * FROM users;

-- Выборка конкретных столбцов
SELECT name, email FROM users;

-- Выборка с условием
SELECT * FROM users WHERE age > 25;

-- Выборка с сортировкой
SELECT * FROM users ORDER BY age DESC;

-- Выборка с ограничением
SELECT * FROM users LIMIT 5;

-- Выборка с пропуском записей (пагинация)
SELECT * FROM users LIMIT 5 OFFSET 10;

-- Выборка с группировкой
SELECT age, COUNT(*) FROM users GROUP BY age;

-- Выборка с условием для групп
SELECT age, COUNT(*) FROM users 
GROUP BY age 
HAVING COUNT(*) > 1;

-- Выборка с соединением таблиц
SELECT u.name, o.total 
FROM users u 
JOIN orders o ON u.id = o.user_id;
```

### UPDATE (Обновление):
```sql
-- Обновление одного поля
UPDATE users SET age = 31 WHERE id = 1;

-- Обновление нескольких полей
UPDATE users 
SET name = 'John Smith', age = 31 
WHERE id = 1;

-- Обновление с условием
UPDATE users SET age = age + 1 WHERE age < 30;

-- Обновление с возвратом данных
UPDATE users 
SET age = 32 
WHERE id = 1 
RETURNING name, age;
```

### DELETE (Удаление):
```sql
-- Удаление конкретной записи
DELETE FROM users WHERE id = 1;

-- Удаление по условию
DELETE FROM users WHERE age < 18;

-- Удаление всех записей (осторожно!)
DELETE FROM users;

-- Удаление с возвратом данных
DELETE FROM users 
WHERE id = 1 
RETURNING name, email;
```

## Принципы проектирования баз данных

Правильное проектирование базы данных - основа успешного приложения. Рассмотрим ключевые принципы для senior разработчиков.

### 1. Определение требований и анализ предметной области

#### Шаги анализа:
```sql
-- 1. Определить сущности (entities)
-- Примеры: User, Order, Product, Category, Review

-- 2. Определить атрибуты каждой сущности
-- User: id, name, email, password_hash, created_at, last_login
-- Order: id, user_id, total, status, created_at, shipped_at
-- Product: id, name, description, price, category_id, stock_quantity

-- 3. Определить связи между сущностями
-- User 1:N Order (один пользователь - много заказов)
-- Order N:M Product (заказ содержит много товаров, товар может быть в разных заказах)
-- Category 1:N Product (одна категория - много товаров)
```

### 2. Принципы именования

#### Лучшие практики:
```sql
-- Таблицы: множественное число, snake_case
CREATE TABLE users (...);           -- ✅ Хорошо
CREATE TABLE user (...);            -- ❌ Плохо
CREATE TABLE UserAccounts (...);    -- ❌ Плохо

-- Столбцы: единственное число, snake_case
CREATE TABLE users (
    id SERIAL PRIMARY KEY,           -- ✅
    full_name VARCHAR(255),          -- ✅
    email_address VARCHAR(255),      -- ✅
    created_at TIMESTAMP,            -- ✅
    firstName VARCHAR(255)           -- ❌ camelCase плохо
);

-- Внешние ключи: название_таблицы_id
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),     -- ✅
    customer_id INTEGER REFERENCES users(id)  -- ❌ неясное название
);

-- Индексы: idx_таблица_столбцы
CREATE INDEX idx_users_email ON users(email);              -- ✅
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at); -- ✅
```

### 3. Выбор типов данных

#### Оптимальные типы для частых случаев:
```sql
CREATE TABLE users (
    -- Первичные ключи
    id BIGSERIAL PRIMARY KEY,        -- Для больших таблиц
    uuid UUID DEFAULT gen_random_uuid(), -- Для распределенных систем
    
    -- Строки
    name VARCHAR(255) NOT NULL,      -- Ограниченная длина лучше TEXT
    description TEXT,                -- Для длинного текста
    slug VARCHAR(100) UNIQUE,        -- URL-friendly идентификаторы
    
    -- Числа
    price DECIMAL(10,2),             -- Точные вычисления (деньги)
    rating NUMERIC(3,2),             -- 0.00 - 5.00
    view_count BIGINT DEFAULT 0,     -- Счетчики
    
    -- Логические значения
    is_active BOOLEAN DEFAULT true,
    is_verified BOOLEAN DEFAULT false,
    
    -- Дата и время
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMPTZ,          -- Soft delete
    
    -- Перечисления
    status VARCHAR(20) CHECK (status IN ('draft', 'published', 'archived')),
    -- Или лучше использовать ENUM
    priority priority_enum DEFAULT 'medium'
);

-- Создание enum типа
CREATE TYPE priority_enum AS ENUM ('low', 'medium', 'high', 'urgent');
```

### 4. Ограничения и валидация на уровне БД

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    
    -- Валидация email
    email VARCHAR(255) UNIQUE NOT NULL 
        CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    
    -- Валидация возраста
    age INTEGER CHECK (age >= 0 AND age <= 150),
    
    -- Валидация телефона
    phone VARCHAR(20) CHECK (phone ~ '^\+[1-9]\d{10,14}$'),
    
    -- Валидация статуса
    status VARCHAR(20) NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'inactive', 'suspended', 'deleted')),
    
    -- Валидация дат
    birth_date DATE CHECK (birth_date < CURRENT_DATE),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    
    -- Составные ограничения
    CONSTRAINT valid_name_length CHECK (length(trim(full_name)) >= 2),
    CONSTRAINT future_dates CHECK (updated_at >= created_at)
);
```

### 5. Принципы проектирования связей

#### One-to-Many (1:N):
```sql
-- Пользователи и их заказы
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    total DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending'
);

-- Индекс для быстрых запросов
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

#### Many-to-Many (N:M):
```sql
-- Товары и категории (товар может быть в нескольких категориях)
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL
);

CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    parent_id BIGINT REFERENCES categories(id)
);

-- Промежуточная таблица
CREATE TABLE product_categories (
    product_id BIGINT REFERENCES products(id) ON DELETE CASCADE,
    category_id BIGINT REFERENCES categories(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, category_id)
);

-- Индексы для быстрых поисков
CREATE INDEX idx_product_categories_product ON product_categories(product_id);
CREATE INDEX idx_product_categories_category ON product_categories(category_id);
```

#### Self-referencing (самосвязь):
```sql
-- Комментарии с ответами
CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    parent_id BIGINT REFERENCES comments(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    
    -- Предотвращаем слишком глубокую вложенность
    level INTEGER DEFAULT 0 CHECK (level >= 0 AND level <= 5)
);

-- Индексы для иерархических запросов
CREATE INDEX idx_comments_parent ON comments(parent_id);
CREATE INDEX idx_comments_user ON comments(user_id);
```

### 6. Паттерны проектирования таблиц

#### Audit Trail (журнал изменений):
```sql
-- Основная таблица
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    version INTEGER DEFAULT 1
);

-- Таблица истории изменений
CREATE TABLE products_history (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL,
    name VARCHAR(255),
    price DECIMAL(10,2),
    changed_by BIGINT REFERENCES users(id),
    changed_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    change_type VARCHAR(10) CHECK (change_type IN ('INSERT', 'UPDATE', 'DELETE')),
    old_values JSONB,
    new_values JSONB
);
```

#### Soft Delete:
```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    deleted_at TIMESTAMPTZ NULL,
    
    -- Индекс для быстрого поиска активных записей
    CHECK (deleted_at IS NULL OR deleted_at > created_at)
);

-- Частичный индекс только для активных записей
CREATE UNIQUE INDEX idx_users_email_active 
ON users(email) 
WHERE deleted_at IS NULL;

-- View для активных пользователей
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL;
```

## Нормализация и денормализация

### Формы нормализации

#### 1NF (Первая нормальная форма):
```sql
-- ❌ Нарушение 1NF - множественные значения в одном поле
CREATE TABLE users_bad (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    phones VARCHAR(255)  -- "123-456, 789-012, 345-678"
);

-- ✅ Соблюдение 1NF - атомарные значения
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE user_phones (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    phone VARCHAR(20),
    phone_type VARCHAR(10) -- 'mobile', 'home', 'work'
);
```

#### 2NF (Вторая нормальная форма):
```sql
-- ❌ Нарушение 2NF - неполная зависимость от составного ключа
CREATE TABLE order_items_bad (
    order_id INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    product_name VARCHAR(255),  -- зависит только от product_id
    product_price DECIMAL(10,2), -- зависит только от product_id
    PRIMARY KEY (order_id, product_id)
);

-- ✅ Соблюдение 2NF - выделение зависимостей
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10,2)
);

CREATE TABLE order_items (
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER,
    price_at_time DECIMAL(10,2), -- цена на момент заказа
    PRIMARY KEY (order_id, product_id)
);
```

#### 3NF (Третья нормальная форма):
```sql
-- ❌ Нарушение 3NF - транзитивная зависимость
CREATE TABLE employees_bad (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    department_id INTEGER,
    department_name VARCHAR(255),    -- зависит от department_id
    department_location VARCHAR(255) -- зависит от department_id
);

-- ✅ Соблюдение 3NF - устранение транзитивных зависимостей
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    location VARCHAR(255)
);

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    department_id INTEGER REFERENCES departments(id)
);
```

### Когда денормализовать

#### Таблица для отчетов (денормализация для производительности):
```sql
-- Нормализованная структура
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    order_id BIGINT REFERENCES orders(id),
    product_id BIGINT REFERENCES products(id),
    quantity INTEGER,
    price DECIMAL(10,2)
);

-- Денормализованная таблица для быстрых отчетов
CREATE TABLE order_summary (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT,
    user_id BIGINT,
    user_name VARCHAR(255),
    total_amount DECIMAL(10,2),
    items_count INTEGER,
    order_date DATE,
    
    -- Индексы для быстрых запросов отчетов
    INDEX idx_order_summary_date (order_date),
    INDEX idx_order_summary_user (user_id),
    INDEX idx_order_summary_amount (total_amount)
);

-- Поддерживаем актуальность через триггеры или ETL процессы
```

#### Кэширование агрегированных данных:
```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10,2),
    
    -- Денормализованные поля для производительности
    total_sales BIGINT DEFAULT 0,
    avg_rating DECIMAL(3,2) DEFAULT 0,
    review_count INTEGER DEFAULT 0,
    
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Обновляем денормализованные данные через триггеры
CREATE OR REPLACE FUNCTION update_product_stats()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE products SET
        total_sales = (
            SELECT COALESCE(SUM(oi.quantity), 0)
            FROM order_items oi
            WHERE oi.product_id = NEW.product_id
        ),
        avg_rating = (
            SELECT COALESCE(AVG(r.rating), 0)
            FROM reviews r
            WHERE r.product_id = NEW.product_id
        ),
        review_count = (
            SELECT COUNT(*)
            FROM reviews r
            WHERE r.product_id = NEW.product_id
        ),
        updated_at = CURRENT_TIMESTAMP
    WHERE id = NEW.product_id;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## JOIN операции - Полный гид

JOIN - это мощный инструмент для связывания данных из разных таблиц. Рассмотрим все типы JOIN с практическими примерами.

### Подготовка тестовых данных:

```sql
-- Создание тестовых таблиц
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    city VARCHAR(100)
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    order_date DATE,
    total DECIMAL(10,2),
    status VARCHAR(20)
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    category VARCHAR(50)
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    price DECIMAL(10,2)
);

-- Вставка тестовых данных
INSERT INTO customers (name, email, city) VALUES
    ('Alice Johnson', 'alice@example.com', 'New York'),
    ('Bob Smith', 'bob@example.com', 'Los Angeles'),
    ('Carol Davis', 'carol@example.com', 'Chicago'),
    ('David Wilson', 'david@example.com', 'Houston'),
    ('Eva Brown', 'eva@example.com', NULL); -- Клиент без города

INSERT INTO orders (customer_id, order_date, total, status) VALUES
    (1, '2024-01-15', 150.00, 'completed'),
    (1, '2024-02-20', 89.50, 'shipped'),
    (2, '2024-01-10', 299.99, 'completed'),
    (3, '2024-03-05', 45.00, 'pending'),
    (NULL, '2024-03-10', 199.99, 'completed'); -- Заказ без клиента

INSERT INTO products (name, price, category) VALUES
    ('Laptop', 999.99, 'Electronics'),
    ('Mouse', 25.99, 'Electronics'),
    ('Book', 12.99, 'Books'),
    ('Headphones', 79.99, 'Electronics');

INSERT INTO order_items (order_id, product_id, quantity, price) VALUES
    (1, 1, 1, 999.99),
    (1, 2, 2, 25.99),
    (2, 3, 3, 12.99),
    (3, 4, 1, 79.99),
    (4, 2, 1, 25.99);
```

### 1. INNER JOIN

Возвращает только записи, которые имеют соответствие в обеих таблицах.

```sql
-- Базовый INNER JOIN
SELECT 
    c.name AS customer_name,
    c.email,
    o.order_date,
    o.total,
    o.status
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;

-- Результат: только клиенты с заказами
```

#### Множественные INNER JOIN:
```sql
-- Связываем клиентов, заказы, товары в заказах и сами товары
SELECT 
    c.name AS customer_name,
    o.order_date,
    p.name AS product_name,
    oi.quantity,
    oi.price,
    (oi.quantity * oi.price) AS subtotal
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id
ORDER BY c.name, o.order_date;
```

### 2. LEFT JOIN (LEFT OUTER JOIN)

Возвращает все записи из левой таблицы и соответствующие из правой.

```sql
-- Показать всех клиентов, включая тех, у кого нет заказов
SELECT 
    c.name AS customer_name,
    c.email,
    o.order_date,
    o.total,
    CASE 
        WHEN o.id IS NULL THEN 'No orders'
        ELSE o.status
    END AS order_status
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
ORDER BY c.name;
```

#### Практический пример - найти клиентов без заказов:
```sql
SELECT 
    c.id,
    c.name,
    c.email
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;
```

### 3. RIGHT JOIN (RIGHT OUTER JOIN)

Возвращает все записи из правой таблицы и соответствующие из левой.

```sql
-- Показать все заказы, включая те, у которых нет клиента
SELECT 
    COALESCE(c.name, 'Unknown Customer') AS customer_name,
    o.order_date,
    o.total,
    o.status
FROM customers c
RIGHT JOIN orders o ON c.id = o.customer_id
ORDER BY o.order_date;
```

### 4. FULL OUTER JOIN

Возвращает все записи из обеих таблиц.

```sql
-- Показать всех клиентов и все заказы
SELECT 
    COALESCE(c.name, 'Unknown Customer') AS customer_name,
    COALESCE(c.email, 'No email') AS email,
    COALESCE(o.order_date::text, 'No order date') AS order_date,
    COALESCE(o.total, 0) AS total
FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id
ORDER BY c.name NULLS LAST, o.order_date NULLS LAST;
```

### 5. CROSS JOIN

Возвращает декартово произведение - каждая запись из первой таблицы со всеми записями из второй.

```sql
-- Пример: все возможные комбинации клиентов и товаров
SELECT 
    c.name AS customer_name,
    p.name AS product_name,
    p.price
FROM customers c
CROSS JOIN products p
WHERE c.city = 'New York' -- Ограничим результат
ORDER BY c.name, p.name;
```

### 6. SELF JOIN

Соединение таблицы с самой собой.

```sql
-- Добавим таблицу сотрудников для примера
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER,
    department VARCHAR(50)
);

INSERT INTO employees (name, manager_id, department) VALUES
    ('John CEO', NULL, 'Management'),
    ('Alice Manager', 1, 'Sales'),
    ('Bob Employee', 2, 'Sales'),
    ('Carol Employee', 2, 'Sales'),
    ('David Manager', 1, 'IT'),
    ('Eva Employee', 5, 'IT');

-- Показать сотрудников с их менеджерами
SELECT 
    e.name AS employee_name,
    e.department,
    COALESCE(m.name, 'No Manager') AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id
ORDER BY e.department, e.name;
```

### 7. Продвинутые техники JOIN

#### JOIN с условиями:
```sql
-- Показать клиентов и их заказы только за определенный период
SELECT 
    c.name,
    o.order_date,
    o.total
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id 
    AND o.order_date >= '2024-02-01'
    AND o.status = 'completed'
ORDER BY o.order_date DESC;
```

#### JOIN с агрегацией:
```sql
-- Статистика по клиентам
SELECT 
    c.id,
    c.name,
    c.email,
    COUNT(o.id) AS total_orders,
    COALESCE(SUM(o.total), 0) AS total_spent,
    COALESCE(AVG(o.total), 0) AS avg_order_value,
    MAX(o.order_date) AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.email
HAVING COUNT(o.id) > 0  -- Только клиенты с заказами
ORDER BY total_spent DESC;
```

#### Подзапросы в JOIN:
```sql
-- Клиенты с их самым дорогим заказом
SELECT 
    c.name,
    c.email,
    max_orders.max_total,
    max_orders.order_date
FROM customers c
INNER JOIN (
    SELECT 
        customer_id,
        MAX(total) AS max_total,
        order_date
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id, order_date
    HAVING total = MAX(total)
) max_orders ON c.id = max_orders.customer_id;
```

#### JOIN с CASE WHEN:
```sql
-- Категоризация клиентов по объему покупок
SELECT 
    c.name,
    COALESCE(SUM(o.total), 0) AS total_spent,
    CASE 
        WHEN COALESCE(SUM(o.total), 0) >= 500 THEN 'VIP'
        WHEN COALESCE(SUM(o.total), 0) >= 200 THEN 'Regular'
        WHEN COALESCE(SUM(o.total), 0) > 0 THEN 'New'
        ELSE 'No purchases'
    END AS customer_tier
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.status = 'completed'
GROUP BY c.id, c.name
ORDER BY total_spent DESC;
```

### 8. Оптимизация JOIN запросов

#### Использование индексов:
```sql
-- Создание индексов для оптимизации JOIN
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- Составные индексы для часто используемых комбинаций
CREATE INDEX idx_orders_customer_status_date 
ON orders(customer_id, status, order_date);
```

#### EXPLAIN для анализа производительности:
```sql
-- Анализ плана выполнения
EXPLAIN (ANALYZE, BUFFERS) 
SELECT 
    c.name,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.city = 'New York'
GROUP BY c.id, c.name
HAVING COUNT(o.id) > 0;
```

### 9. Распространенные ошибки и их решения

#### Неправильное использование WHERE vs ON:
```sql
-- ❌ Плохо - фильтр в WHERE может превратить LEFT JOIN в INNER JOIN
SELECT c.name, o.total
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'completed';  -- Исключит клиентов без заказов

-- ✅ Хорошо - фильтр в ON сохраняет логику LEFT JOIN
SELECT c.name, o.total
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.status = 'completed';
```

#### Обработка NULL значений:
```sql
-- ✅ Правильная обработка NULL
SELECT 
    c.name,
    COALESCE(SUM(o.total), 0) AS total_spent,
    COUNT(o.id) AS order_count  -- COUNT не считает NULL
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;
```

## Транзакции и ACID

### Основы транзакций

Транзакция - это последовательность операций, которая выполняется как единое целое.

```sql
-- Базовый синтаксис транзакции
BEGIN;
    INSERT INTO accounts (name, balance) VALUES ('Alice', 1000);
    INSERT INTO accounts (name, balance) VALUES ('Bob', 500);
COMMIT;

-- Откат транзакции при ошибке
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
    UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
    -- Если что-то пошло не так
ROLLBACK;
```

### Уровни изоляции транзакций

```sql
-- Установка уровня изоляции для текущей транзакции
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- или
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Уровни изоляции:
-- READ UNCOMMITTED - самый низкий уровень
-- READ COMMITTED - по умолчанию в PostgreSQL
-- REPEATABLE READ - предотвращает non-repeatable reads
-- SERIALIZABLE - самый высокий уровень
```

### Блокировки

```sql
-- Явные блокировки
BEGIN;
    SELECT * FROM products WHERE id = 1 FOR UPDATE;  -- Блокировка на запись
    UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;

-- Блокировка на чтение
SELECT * FROM products FOR SHARE;

-- Неблокирующие блокировки
SELECT * FROM products WHERE id = 1 FOR UPDATE NOWAIT;
SELECT * FROM products WHERE id = 1 FOR UPDATE SKIP LOCKED;
```

## Хранимые процедуры и триггеры

### Функции PL/pgSQL

```sql
-- Простая функция
CREATE OR REPLACE FUNCTION calculate_total(
    order_id_param BIGINT
) RETURNS DECIMAL(10,2) AS $$
DECLARE
    total_amount DECIMAL(10,2);
BEGIN
    SELECT SUM(quantity * price) INTO total_amount
    FROM order_items 
    WHERE order_id = order_id_param;
    
    RETURN COALESCE(total_amount, 0);
END;
$$ LANGUAGE plpgsql;

-- Использование функции
SELECT calculate_total(1);
```

### Триггеры

```sql
-- Функция для триггера
CREATE OR REPLACE FUNCTION update_order_total()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE orders 
    SET total = (
        SELECT SUM(quantity * price)
        FROM order_items
        WHERE order_id = NEW.order_id
    )
    WHERE id = NEW.order_id;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Создание триггера
CREATE TRIGGER trg_update_order_total
    AFTER INSERT OR UPDATE OR DELETE ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION update_order_total();

-- Триггер для аудита
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (
        table_name, 
        operation, 
        old_values, 
        new_values, 
        changed_by,
        changed_at
    ) VALUES (
        TG_TABLE_NAME,
        TG_OP,
        CASE WHEN TG_OP = 'DELETE' THEN row_to_json(OLD) ELSE NULL END,
        CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN row_to_json(NEW) ELSE NULL END,
        current_user,
        CURRENT_TIMESTAMP
    );
    
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

## Оконные функции

Оконные функции позволяют выполнять вычисления для набора строк, связанных с текущей строкой.

```sql
-- Ранжирование клиентов по общей сумме покупок
SELECT 
    c.name,
    SUM(o.total) as total_spent,
    ROW_NUMBER() OVER (ORDER BY SUM(o.total) DESC) as rank,
    RANK() OVER (ORDER BY SUM(o.total) DESC) as dense_rank,
    PERCENT_RANK() OVER (ORDER BY SUM(o.total) DESC) as percent_rank
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;

-- Скользящее окно - средняя сумма заказа за последние 3 заказа
SELECT 
    order_date,
    total,
    AVG(total) OVER (
        ORDER BY order_date 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg
FROM orders
ORDER BY order_date;

-- Сравнение с предыдущим/следующим значением
SELECT 
    order_date,
    total,
    LAG(total) OVER (ORDER BY order_date) as prev_total,
    LEAD(total) OVER (ORDER BY order_date) as next_total,
    total - LAG(total) OVER (ORDER BY order_date) as diff_from_prev
FROM orders
ORDER BY order_date;

-- Партиционирование - ранжирование внутри групп
SELECT 
    c.name,
    o.order_date,
    o.total,
    ROW_NUMBER() OVER (
        PARTITION BY c.id 
        ORDER BY o.total DESC
    ) as rank_within_customer
FROM customers c
JOIN orders o ON c.id = o.customer_id
ORDER BY c.name, rank_within_customer;
```

## CTE и рекурсивные запросы

### Common Table Expressions (CTE)

```sql
-- Простой CTE
WITH customer_stats AS (
    SELECT 
        c.id,
        c.name,
        COUNT(o.id) as order_count,
        SUM(o.total) as total_spent
    FROM customers c
    LEFT JOIN orders o ON c.id = o.customer_id
    GROUP BY c.id, c.name
)
SELECT 
    name,
    order_count,
    total_spent,
    CASE 
        WHEN total_spent > 500 THEN 'VIP'
        WHEN total_spent > 100 THEN 'Regular'
        ELSE 'New'
    END as customer_tier
FROM customer_stats
ORDER BY total_spent DESC;

-- Множественные CTE
WITH 
high_value_customers AS (
    SELECT customer_id, SUM(total) as total_spent
    FROM orders
    GROUP BY customer_id
    HAVING SUM(total) > 200
),
recent_orders AS (
    SELECT *
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
)
SELECT 
    c.name,
    hvc.total_spent,
    COUNT(ro.id) as recent_orders_count
FROM high_value_customers hvc
JOIN customers c ON hvc.customer_id = c.id
LEFT JOIN recent_orders ro ON c.id = ro.customer_id
GROUP BY c.id, c.name, hvc.total_spent
ORDER BY hvc.total_spent DESC;
```

### Рекурсивные CTE

```sql
-- Построение иерархии сотрудников
WITH RECURSIVE employee_hierarchy AS (
    -- Базовый случай: CEO (нет менеджера)
    SELECT 
        id,
        name,
        manager_id,
        0 as level,
        name as hierarchy_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Рекурсивный случай: подчиненные
    SELECT 
        e.id,
        e.name,
        e.manager_id,
        eh.level + 1,
        eh.hierarchy_path || ' -> ' || e.name
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT 
    REPEAT('  ', level) || name as indented_name,
    level,
    hierarchy_path
FROM employee_hierarchy
ORDER BY hierarchy_path;

-- Поиск всех потомков в дереве категорий
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 as depth
    FROM categories
    WHERE id = 1  -- Начинаем с конкретной категории
    
    UNION ALL
    
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree;
```

## JSON и NoSQL возможности

### Работа с JSONB

```sql
-- Создание таблицы с JSONB полем
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255),
    metadata JSONB,
    tags TEXT[]
);

-- Вставка данных с JSON
INSERT INTO products (name, metadata, tags) VALUES 
(
    'Laptop',
    '{
        "specs": {
            "cpu": "Intel i7",
            "ram": "16GB",
            "storage": "512GB SSD"
        },
        "features": ["backlit keyboard", "fingerprint reader"],
        "warranty_years": 2
    }'::jsonb,
    '{"electronics", "computers", "portable"}'
);

-- Запросы к JSON полям
-- Извлечение значений
SELECT 
    name,
    metadata->>'warranty_years' as warranty,
    metadata->'specs'->>'cpu' as cpu,
    metadata->'specs'->>'ram' as ram
FROM products;

-- Фильтрация по JSON полям
SELECT name FROM products 
WHERE metadata->'specs'->>'cpu' LIKE '%i7%';

-- Проверка существования ключа
SELECT name FROM products 
WHERE metadata ? 'warranty_years';

-- Поиск в массиве JSON
SELECT name FROM products
WHERE metadata->'features' @> '["backlit keyboard"]';

-- Обновление JSON данных
UPDATE products 
SET metadata = jsonb_set(
    metadata, 
    '{specs,storage}', 
    '"1TB SSD"', 
    false
)
WHERE name = 'Laptop';

-- Добавление нового поля
UPDATE products
SET metadata = metadata || '{"color": "silver"}'::jsonb
WHERE name = 'Laptop';

-- Удаление поля
UPDATE products
SET metadata = metadata - 'color'
WHERE name = 'Laptop';
```

### Индексы для JSON

```sql
-- GIN индекс для быстрого поиска в JSONB
CREATE INDEX idx_products_metadata ON products USING gin(metadata);

-- Индекс для конкретного пути в JSON
CREATE INDEX idx_products_cpu 
ON products USING btree((metadata->'specs'->>'cpu'));

-- Индекс для поиска по массиву тегов
CREATE INDEX idx_products_tags ON products USING gin(tags);
```

## Партиционирование

Партиционирование разделяет большие таблицы на меньшие части для улучшения производительности.

### Range партиционирование

```sql
-- Создание партиционированной таблицы
CREATE TABLE orders_partitioned (
    id BIGSERIAL,
    customer_id BIGINT,
    order_date DATE NOT NULL,
    total DECIMAL(10,2),
    status VARCHAR(20),
    PRIMARY KEY (id, order_date)
) PARTITION BY RANGE (order_date);

-- Создание партиций
CREATE TABLE orders_2024_q1 PARTITION OF orders_partitioned
FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders_partitioned
FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

CREATE TABLE orders_2024_q3 PARTITION OF orders_partitioned
FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');

CREATE TABLE orders_2024_q4 PARTITION OF orders_partitioned
FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

-- Индексы на партиции
CREATE INDEX idx_orders_2024_q1_customer 
ON orders_2024_q1 (customer_id);

CREATE INDEX idx_orders_2024_q2_customer 
ON orders_2024_q2 (customer_id);
```

### Hash партиционирование

```sql
-- Партиционирование по хешу для равномерного распределения
CREATE TABLE users_partitioned (
    id BIGSERIAL,
    name VARCHAR(255),
    email VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
) PARTITION BY HASH (id);

-- Создание партиций по хешу
CREATE TABLE users_partition_0 PARTITION OF users_partitioned
FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE users_partition_1 PARTITION OF users_partitioned
FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE users_partition_2 PARTITION OF users_partitioned
FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE users_partition_3 PARTITION OF users_partitioned
FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

## Best Practices для Senior разработчиков

### 1. Проектирование схемы

```sql
-- ✅ Всегда используйте NOT NULL где возможно
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    bio TEXT, -- может быть NULL
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- ✅ Используйте CHECK constraints для валидации
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    status VARCHAR(20) NOT NULL DEFAULT 'active' 
        CHECK (status IN ('active', 'inactive', 'discontinued'))
);

-- ✅ Используйте ENUM для предопределенных значений
CREATE TYPE order_status AS ENUM (
    'pending', 'processing', 'shipped', 'delivered', 'cancelled'
);
```

### 2. Индексация

```sql
-- ✅ Создавайте индексы для внешних ключей
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- ✅ Составные индексы для часто используемых WHERE условий
CREATE INDEX idx_orders_status_date 
ON orders(status, order_date) 
WHERE status IN ('pending', 'processing');

-- ✅ Частичные индексы для фильтрации
CREATE INDEX idx_active_users_email 
ON users(email) 
WHERE deleted_at IS NULL;

-- ✅ Уникальные частичные индексы
CREATE UNIQUE INDEX idx_users_email_active 
ON users(email) 
WHERE deleted_at IS NULL;
```

### 3. Запросы и производительность

```sql
-- ✅ Используйте EXPLAIN для анализа производительности
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT c.name, COUNT(o.id) as order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.created_at >= '2024-01-01'
GROUP BY c.id, c.name
HAVING COUNT(o.id) > 5;

-- ✅ Избегайте SELECT *
SELECT id, name, email FROM users WHERE active = true;

-- ✅ Используйте LIMIT для больших наборов данных
SELECT id, name FROM products 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 0;

-- ✅ Предпочитайте EXISTS вместо IN для подзапросов
SELECT c.name 
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.id 
    AND o.total > 1000
);
```

### 4. Безопасность

```sql
-- ✅ Создавайте отдельных пользователей для приложений
CREATE USER app_user WITH PASSWORD 'strong_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- ✅ Используйте Row Level Security
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_orders_policy ON orders
    FOR ALL TO app_user
    USING (customer_id = current_setting('app.current_user_id')::bigint);

-- ✅ Храните пароли правильно (в приложении, не в БД)
-- Никогда не храните пароли в открытом виде
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL, -- bcrypt hash
    salt VARCHAR(255) NOT NULL
);
```

### 5. Мониторинг и логирование

```sql
-- ✅ Включите логирование медленных запросов
-- В postgresql.conf:
-- log_min_duration_statement = 1000
-- log_statement = 'all'
-- log_duration = on

-- ✅ Мониторинг активных запросов
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY query_start;

-- ✅ Проверка размеров таблиц
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    pg_total_relation_size(schemaname||'.'||tablename) as size_bytes
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY size_bytes DESC;
```

### 6. Резервное копирование и восстановление

```bash
# ✅ Автоматическое резервное копирование
# Создание скрипта backup.sh
#!/bin/bash
DATESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"
DB_NAME="your_database"

# Создание дампа
pg_dump -h localhost -U postgres -d $DB_NAME -Fc > \
  $BACKUP_DIR/db_backup_$DATESTAMP.dump

# Сжатие старых бэкапов
find $BACKUP_DIR -name "*.dump" -mtime +7 -delete

# ✅ Восстановление
pg_restore -h localhost -U postgres -d $DB_NAME -c backup.dump

# ✅ Точечное восстановление (Point-in-time recovery)
# Включить в postgresql.conf:
# archive_mode = on
# archive_command = 'cp %p /archive/%f'
```

### 7. Оптимизация для продакшена

```sql
-- ✅ Регулярное обновление статистики
ANALYZE;

-- ✅ Очистка и дефрагментация
VACUUM ANALYZE;

-- ✅ Перестроение индексов при необходимости
REINDEX INDEX idx_orders_customer_id;

-- ✅ Настройка автовакуума
-- В postgresql.conf:
-- autovacuum = on
-- autovacuum_max_workers = 3
-- autovacuum_naptime = 1min

-- ✅ Проверка неиспользуемых индексов
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY schemaname, tablename;
```

## Типы данных

### Числовые типы:
```sql
-- Целые числа
SMALLINT    -- -32,768 to 32,767
INTEGER     -- -2,147,483,648 to 2,147,483,647
BIGINT      -- -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807
SERIAL      -- Auto-incrementing integer
BIGSERIAL   -- Auto-incrementing bigint

-- Десятичные числа
DECIMAL(precision, scale)  -- Точные десятичные числа
NUMERIC(precision, scale)  -- То же что DECIMAL
REAL                       -- 6 decimal digits precision
DOUBLE PRECISION          -- 15 decimal digits precision
```

### Строковые типы:
```sql
CHAR(n)          -- Фиксированная длина
VARCHAR(n)       -- Переменная длина с ограничением
TEXT             -- Неограниченная длина
```

### Дата и время:
```sql
DATE             -- Только дата (YYYY-MM-DD)
TIME             -- Только время (HH:MM:SS)
TIMESTAMP        -- Дата и время
TIMESTAMPTZ      -- Дата и время с часовым поясом
INTERVAL         -- Временной интервал
```

### Другие полезные типы:
```sql
BOOLEAN          -- true/false
JSON             -- JSON данные
JSONB            -- Бинарный JSON (рекомендуется)
UUID             -- Уникальный идентификатор
ARRAY            -- Массивы любого типа
```

### Пример использования типов:
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2),
    in_stock BOOLEAN DEFAULT true,
    tags TEXT[],
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Индексы

### Создание индексов:
```sql
-- Простой индекс
CREATE INDEX idx_users_email ON users(email);

-- Составной индекс
CREATE INDEX idx_users_name_age ON users(name, age);

-- Уникальный индекс
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Частичный индекс
CREATE INDEX idx_active_users ON users(name) WHERE active = true;

-- B-tree индекс (по умолчанию)
CREATE INDEX idx_users_age ON users USING btree(age);

-- Hash индекс
CREATE INDEX idx_users_status ON users USING hash(status);

-- GIN индекс для JSONB
CREATE INDEX idx_products_metadata ON products USING gin(metadata);
```

### Удаление индексов:
```sql
DROP INDEX idx_users_email;
```

### Просмотр индексов:
```sql
-- Показать все индексы таблицы
\d users

-- SQL запрос для просмотра индексов
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'users';
```

## Пользователи и права доступа

### Создание пользователей:
```sql
-- Создать пользователя
CREATE USER username WITH PASSWORD 'password';

-- Создать роль (пользователя) с правами
CREATE ROLE developer WITH LOGIN PASSWORD 'dev_password' CREATEDB;

-- Создать суперпользователя
CREATE USER admin WITH PASSWORD 'admin_pass' SUPERUSER;
```

### Управление правами:
```sql
-- Дать права на базу данных
GRANT ALL PRIVILEGES ON DATABASE my_database TO username;

-- Дать права на таблицу
GRANT SELECT, INSERT, UPDATE ON users TO username;

-- Дать права на все таблицы в схеме
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO username;

-- Отозвать права
REVOKE ALL PRIVILEGES ON users FROM username;

-- Сделать пользователя владельцем базы данных
ALTER DATABASE my_database OWNER TO username;
```

### Просмотр пользователей и ролей:
```sql
-- Показать всех пользователей
\du

-- SQL запрос
SELECT rolname, rolsuper, rolcreatedb, rolcanlogin FROM pg_roles;
```

## Полезные команды

### Информация о базе данных:
```sql
-- Размер базы данных
SELECT pg_size_pretty(pg_database_size('database_name'));

-- Размер таблицы
SELECT pg_size_pretty(pg_total_relation_size('table_name'));

-- Информация о таблицах
SELECT 
    table_name,
    pg_size_pretty(pg_total_relation_size(table_name::regclass)) as size
FROM information_schema.tables 
WHERE table_schema = 'public'
ORDER BY pg_total_relation_size(table_name::regclass) DESC;
```

### Работа с последовательностями (Sequences):
```sql
-- Создать последовательность
CREATE SEQUENCE my_sequence START 1;

-- Получить следующее значение
SELECT nextval('my_sequence');

-- Получить текущее значение
SELECT currval('my_sequence');

-- Сбросить последовательность
ALTER SEQUENCE my_sequence RESTART WITH 1;
```

### Транзакции:
```sql
-- Начать транзакцию
BEGIN;

-- Выполнить операции
INSERT INTO users (name, email) VALUES ('Test User', 'test@example.com');
UPDATE users SET age = 25 WHERE name = 'Test User';

-- Подтвердить изменения
COMMIT;

-- Или отменить изменения
ROLLBACK;
```

### Резервное копирование и восстановление:
```bash
# Создать дамп базы данных
pg_dump -U username -d database_name > backup.sql

# Создать дамп в сжатом формате
pg_dump -U username -d database_name -Fc > backup.dump

# Восстановить из SQL файла
psql -U username -d database_name < backup.sql

# Восстановить из дампа
pg_restore -U username -d database_name backup.dump
```

## Мониторинг и диагностика

### Активные подключения:
```sql
-- Показать активные подключения
SELECT pid, usename, application_name, client_addr, state, query_start, query
FROM pg_stat_activity
WHERE state = 'active';

-- Количество подключений по базам
SELECT datname, count(*) 
FROM pg_stat_activity 
GROUP BY datname;
```

### Статистика таблиц:
```sql
-- Статистика использования таблиц
SELECT schemaname, tablename, n_tup_ins, n_tup_upd, n_tup_del, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;
```

### Медленные запросы:
```sql
-- Включить логирование медленных запросов (в postgresql.conf)
-- log_min_duration_statement = 1000  # логировать запросы дольше 1 секунды

-- Посмотреть план выполнения запроса
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 25;
```

### Блокировки:
```sql
-- Показать текущие блокировки
SELECT 
    pg_stat_activity.pid,
    pg_stat_activity.usename,
    pg_locks.mode,
    pg_locks.locktype,
    pg_locks.relation::regclass
FROM pg_locks
JOIN pg_stat_activity ON pg_locks.pid = pg_stat_activity.pid
WHERE NOT pg_locks.granted;
```

## Полезные функции и операторы

### Строковые функции:
```sql
-- Конкатенация строк
SELECT 'Hello' || ' ' || 'World';
SELECT CONCAT('Hello', ' ', 'World');

-- Длина строки
SELECT LENGTH('Hello World');

-- Поиск подстроки
SELECT POSITION('World' IN 'Hello World');

-- Замена подстроки
SELECT REPLACE('Hello World', 'World', 'PostgreSQL');

-- Приведение к верхнему/нижнему регистру
SELECT UPPER('hello'), LOWER('WORLD');
```

### Функции даты и времени:
```sql
-- Текущая дата и время
SELECT NOW(), CURRENT_DATE, CURRENT_TIME;

-- Извлечение частей даты
SELECT EXTRACT(YEAR FROM NOW()), EXTRACT(MONTH FROM NOW());

-- Форматирование даты
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD HH24:MI:SS');

-- Арифметика с датами
SELECT NOW() + INTERVAL '1 day';
SELECT NOW() - INTERVAL '1 month';
```

### Агрегатные функции:
```sql
SELECT 
    COUNT(*),           -- Количество записей
    AVG(age),          -- Среднее значение
    SUM(age),          -- Сумма
    MIN(age),          -- Минимальное значение
    MAX(age),          -- Максимальное значение
    STDDEV(age)        -- Стандартное отклонение
FROM users;
```

### Работа с JSON:
```sql
-- Создание таблицы с JSON
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    data JSONB
);

-- Вставка JSON данных
INSERT INTO products (data) VALUES 
    ('{"name": "iPhone", "price": 999, "specs": {"storage": "128GB", "color": "black"}}'),
    ('{"name": "Samsung", "price": 799, "specs": {"storage": "256GB", "color": "white"}}');

-- Запросы к JSON полям
SELECT data->>'name' as product_name FROM products;
SELECT * FROM products WHERE data->>'name' = 'iPhone';
SELECT * FROM products WHERE (data->'specs'->>'storage') = '128GB';

-- Обновление JSON данных
UPDATE products 
SET data = jsonb_set(data, '{price}', '899')
WHERE data->>'name' = 'iPhone';
```

## Миграции

Зачем нужны миграции
- Синхронизация схемы между средами (local/staging/prod) и разработчиками.
- История эволюции БД под контролем версий: удобно ревьюить, катить вперёд/назад.
- Повторяемость и автоматизация в CI/CD: предсказуемые деплои.
- Аудит и соответствие требованиям (кто и что менял в схеме).

Подходы к миграциям
- Чистый SQL: вы пишете .sql-файлы, применяете их последовательно (psql -f ...). Максимальный контроль и совместимость.
- Инструменты ORM (например, Prisma): генерация SQL на основе schema.prisma, удобные команды (migrate dev/deploy), встроенная таблица миграций.
- Комбинированно: DDL через миграции, сложные data-migrations отдельными скриптами/процедурами.

Пошаговый процесс (expand/contract)
1) Спланировать изменение и обратимость.
   - Для больших изменений используйте стратегию expand-and-contract: сначала расширяем схему, обеспечиваем обратную совместимость, затем сужаем (удаляем старое).
2) Создать миграцию.
   - SQL-подход: создайте файл в migrations/, например migrations/2025-10-04_add_users_table.sql.
   - Prisma: измените prisma/schema.prisma и выполните:
     ```bash
     npx prisma migrate dev -n "add_users_table"
     ```
3) Пишем DDL с учётом блокировок.
   - По возможности оборачивайте изменения в транзакцию:
     ```sql
     BEGIN;
     ALTER TABLE users ADD COLUMN bio text;
     COMMIT;
     ```
   - Исключения: операции CONCURRENTLY (индексы) нельзя внутри транзакции.
4) Обеспечиваем совместимость кода.
   - Разворачиваем код, который работает и со старой, и с новой схемой (feature flag/двойная запись/чтение из нового поля и fallback).
5) Прогоняем локально, затем в staging.
   - Настройте lock_timeout/statement_timeout, чтобы не «повесить» прод:
     ```sql
     SET lock_timeout = '5s';
     SET statement_timeout = '5min';
     ```
6) Ревью и деплой.
   - В проде применяйте минимальные по времени, безопасные миграции, вне пиковых часов.
7) Наблюдаем, затем завершаем контракт (contract-этап):
   - Удаляем старые поля/индексы, включаем NOT NULL и т.д., когда убеждены, что код полностью перешёл на новую схему.

Примеры практических миграций (SQL)
- Добавить NOT NULL столбец в большую таблицу (без долгой блокировки):
  ```sql
  -- 1) Добавить nullable столбец
  ALTER TABLE orders ADD COLUMN customer_phone text;

  -- 2) Постепенно заполнить (батчами из приложения или временным скриптом)
  -- UPDATE orders SET customer_phone = ... WHERE customer_phone IS NULL LIMIT 10000;

  -- 3) Задать DEFAULT (для новых строк)
  ALTER TABLE orders ALTER COLUMN customer_phone SET DEFAULT '';

  -- 4) Сделать NOT NULL (только когда все строки заполнены)
  ALTER TABLE orders ALTER COLUMN customer_phone SET NOT NULL;
  ```

- Индекс без простоя:
  ```sql
  -- Создание индекса без долгого ACCESS EXCLUSIVE lock
  CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_created_at ON orders (created_at);

  -- Удаление индекса также без простоя
  DROP INDEX CONCURRENTLY IF EXISTS idx_orders_old;
  ```

- Внешний ключ «без боли»:
  ```sql
  -- Сначала индекс на referenced-колонку
  CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_user_id ON orders (user_id);

  -- Добавляем constraint без немедленной проверки
  ALTER TABLE orders
    ADD CONSTRAINT orders_user_id_fkey
    FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;

  -- Затем валидируем отдельно (короче блокировки)
  ALTER TABLE orders VALIDATE CONSTRAINT orders_user_id_fkey;
  ```

- Переименование/перенос данных (expand/contract):
  ```sql
  -- expand: добавляем новое поле
  ALTER TABLE users ADD COLUMN full_name text;
  -- бэкуем/заливаем данные (из first_name/last_name)
  -- затем код читает full_name при наличии, иначе старые поля

  -- contract: когда код обновлён и данные перенесены
  ALTER TABLE users DROP COLUMN first_name;
  ALTER TABLE users DROP COLUMN last_name;
  ```

Таблица учёта миграций (для SQL-подхода)
- Создайте таблицу schema_migrations:
  ```sql
  CREATE TABLE IF NOT EXISTS schema_migrations (
    version text PRIMARY KEY,
    applied_at timestamptz NOT NULL DEFAULT now()
  );
  ```
- Каждая миграция имеет версию (метку времени/хэш). После успешного применения — вставляйте запись.

Работа с Prisma миграциями
- Локально:
  ```bash
  npx prisma migrate dev -n "feature_name"
  ```
  Эта команда создаст папку prisma/migrations/* с SQL и применит миграции к вашей БД.
- Прод:
  ```bash
  npx prisma migrate deploy
  ```
  Применяет уже сгенерированные миграции без интерактива. Рекомендуется для CI/CD.
- Проверяйте сгенерированный SQL и применяемые блокировки. Для сложных кейсов допускается вручную редактировать SQL миграции (но осторожно!).
- Data-migrations делайте отдельными скриптами или в отдельных «безопасных» шагах (batch updates).

Частые ошибки
- Долгие блокировки из-за отсутствия CONCURRENTLY/NOT VALID/VALIDATE.
- Тяжёлые миграции в пиковые часы.
- ALTER TABLE с NOT NULL сразу по гигантской таблице без предварительного backfill.
- Миграции без транзакций (или, наоборот, попытка завернуть CONCURRENTLY внутрь транзакции).
- Отсутствие бэкапа/плана отката.
- Отсутствие тестов/прогона на staging.
- Непроверенные изменения типов (без USING или без промежуточной колонки).

Лучшие практики
- Декомпозируйте: одна миграция — одно логическое изменение.
- Будьте идемпотентны: используйте IF EXISTS/IF NOT EXISTS, проверяйте состояние перед изменением.
- Для FK: сначала индекс, затем CONSTRAINT ... NOT VALID, потом VALIDATE.
- Для индексов: CREATE/DROP INDEX CONCURRENTLY.
- Для больших таблиц: batched backfill, ANALYZE, контроль времени и нагрузки.
- Используйте lock_timeout, statement_timeout в проде.
- Следуйте стратегии expand-and-contract с фичефлагами.
- Ревью SQL, тестирование на staging, мониторинг после выката.

Шпаргалка команд
- Чистый SQL:
  ```bash
  # применить миграцию из файла
  psql -h localhost -U <SUPERUSER> -d <DB> -v ON_ERROR_STOP=1 -f migrations/2025-10-04_add_feature.sql
  ```
- Prisma:
  ```bash
  npx prisma migrate dev -n "add_feature"
  npx prisma migrate deploy
  npx prisma migrate resolve
  ```

## Ссылки для дальнейшего изучения

### Официальная документация:
- **[PostgreSQL Documentation](https://www.postgresql.org/docs/)** - Основная документация
- **[SQL Commands Reference](https://www.postgresql.org/docs/current/sql-commands.html)** - Справочник по SQL командам
- **[Data Types](https://www.postgresql.org/docs/current/datatype.html)** - Типы данных
- **[Functions and Operators](https://www.postgresql.org/docs/current/functions.html)** - Функции и операторы

### Производительность и оптимизация:
- **[Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)** - Советы по производительности
- **[EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)** - Анализ планов выполнения запросов
- **[Indexes](https://www.postgresql.org/docs/current/indexes.html)** - Работа с индексами

### Администрирование:
- **[Server Administration](https://www.postgresql.org/docs/current/admin.html)** - Администрирование сервера
- **[Backup and Restore](https://www.postgresql.org/docs/current/backup.html)** - Резервное копирование
- **[User Management](https://www.postgresql.org/docs/current/user-manag.html)** - Управление пользователями

### Расширенные возможности:
- **[JSON Functions](https://www.postgresql.org/docs/current/functions-json.html)** - Работа с JSON
- **[Full Text Search](https://www.postgresql.org/docs/current/textsearch.html)** - Полнотекстовый поиск
- **[Window Functions](https://www.postgresql.org/docs/current/functions-window.html)** - Оконные функции
- **[Common Table Expressions (CTE)](https://www.postgresql.org/docs/current/queries-with.html)** - Обычные табличные выражения

### Обучающие ресурсы:
- **[PostgreSQL Tutorial](https://www.postgresqltutorial.com/)** - Подробные туториалы
- **[PGExercises](https://pgexercises.com/)** - Практические упражнения
- **[PostgreSQL Wiki](https://wiki.postgresql.org/wiki/Main_Page)** - Community wiki

### Инструменты:
- **[pgAdmin](https://www.pgadmin.org/)** - Графический интерфейс администрирования
- **[psql](https://www.postgresql.org/docs/current/app-psql.html)** - Командная строка
- **[DBeaver](https://dbeaver.io/)** - Универсальный клиент для баз данных

---

## Заключение

Этот tutorial покрывает основы работы с PostgreSQL. Для углубленного изучения рекомендуется:

1. **Практиковаться** с реальными данными
2. **Изучить планы выполнения** запросов через EXPLAIN
3. **Освоить расширенные функции** JSON, полнотекстового поиска
4. **Изучить администрирование** для продакшен окружений
5. **Следить за производительностью** и оптимизацией запросов

Удачного изучения PostgreSQL! 🐘