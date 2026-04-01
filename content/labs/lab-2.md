## 1. Тема, мета, посилання

### 1.1 Тема
«Створення бази даних у PostgreSQL. Підключення Node.js до PostgreSQL. Робота з ORM Prisma». 

### 1.2 Мета
Побудувати реляційну базу даних для власного web-застосунку Helpdesk / Ticket System, підключити backend на Node.js до PostgreSQL, реалізувати роботу з даними через Prisma ORM, виконати CRUD-операції для основних сутностей та окремо показати виконання прямих SQL-запитів із серверної частини.

### 1.3 Посилання
- Репозиторій власного веб-застосунку (GitHub): [посилання](https://github.com/pliffdax/Helpdesk)
- Репозиторій звітного HTML-документа (GitHub): [посилання](https://github.com/pliffdax/IO-35_appRECORD-StepanovOleksandr-FIOT-2026)
- Звітний HTML-документ (Жива сторінка): [посилання](https://pliffdax.github.io/IO-35_appRECORD-StepanovOleksandr-FIOT-2026/)

---

## 2. Короткі теоретичні відомості

### 2.1 PostgreSQL
Для цієї роботи обрано PostgreSQL як реляційну систему керування базами даних. Вона добре підходить для helpdesk-сценарію, де є окремі таблиці користувачів, категорій і заявок, а також зовнішні ключі між ними. У такій структурі зручно підтримувати цілісність даних і розширювати модель без переробки всієї серверної частини.

### 2.2 Node.js та Fastify
Серверну логіку реалізовано на Node.js, а HTTP API побудовано на Fastify. Така зв’язка дає просту й достатньо легку основу для маршрутизації, валідації та обробки запитів. У межах лабораторної роботи цього достатньо, щоб окремо винести маршрути для `users`, `categories` і `tickets` та перевірити їх як через Postman, так і через web-інтерфейс.

### 2.3 Prisma ORM
Prisma використано як ORM-прошарок між TypeScript-кодом і PostgreSQL. Схема таблиць описується в `schema.prisma`, після чого на її основі генерується Prisma Client. Далі в коді вже можна працювати через методи на кшталт `findMany`, `findUnique`, `create`, `update`, `delete`, не виписуючи вручну більшість типових SQL-запитів.

### 2.4 Прямі SQL-запити
Окремо від ORM у роботі залишено демонстраційний скрипт на базі пакета `pg`. Він потрібен не для основного API, а для того, щоб показати пряме виконання `SELECT`, `INSERT`, `UPDATE` і `DELETE` із Node.js. Такий сценарій зручний саме для лабораторної перевірки, бо наочно показує різницю між ORM-рівнем і роботою з SQL напряму.

---

## 3. Проєктування бази даних

### 3.1 Предметна область
У якості прикладної області взято helpdesk-систему для обліку звернень користувачів. Кожна заявка має автора, категорію, опис, статус і пріоритет. На практиці це дає мінімальний, але вже робочий набір сутностей, достатній для демонстрації зв’язків між таблицями та CRUD-операцій.

### 3.2 Основні сутності
У лабораторній версії проєкту використано три основні сутності:
- `User` — користувач системи;
- `Category` — категорія, до якої належить заявка;
- `Ticket` — саме звернення з назвою, описом, статусом і пріоритетом.

### 3.3 Зв’язки між таблицями
У моделі реалізовано два зв’язки типу **One-to-Many**:
- один користувач може створити багато заявок (`User -> Ticket`);
- одна категорія може містити багато заявок (`Category -> Ticket`).

### 3.4 Prisma-схема
Нижче наведено скорочений фрагмент схеми даних, яка використовується в проєкті:

```prisma
enum Role {
  USER
  AGENT
  ADMIN
}

enum Status {
  OPEN
  IN_PROGRESS
  RESOLVED
  CLOSED
}

enum Priority {
  LOW
  MEDIUM
  HIGH
}

model User {
  id      Int      @id @default(autoincrement())
  name    String
  email   String   @unique
  role    Role     @default(USER)
  tickets Ticket[]

  @@map("users")
}

model Category {
  id      Int      @id @default(autoincrement())
  name    String   @unique
  tickets Ticket[]

  @@map("categories")
}

model Ticket {
  id          Int      @id @default(autoincrement())
  title       String
  description String
  status      Status   @default(OPEN)
  priority    Priority @default(MEDIUM)
  creatorId   Int      @map("creator_id")
  categoryId  Int      @map("category_id")
  creator     User     @relation(fields: [creatorId], references: [id])
  category    Category @relation(fields: [categoryId], references: [id])

  @@map("tickets")
}
```

---

## 4. Реалізація backend-частини

### 4.1 Структура серверної частини
Серверна частина розташована в `apps/api` monorepo-проєкту. Основні файли розподілено так:
- `src/server.ts` — запуск сервера;
- `src/app.ts` — створення Fastify-застосунку та реєстрація плагінів;
- `src/routes/` — маршрути для користувачів, категорій і заявок;
- `src/lib/prisma.ts` — ініціалізація Prisma Client;
- `prisma/schema.prisma` — опис структури таблиць;
- `prisma/seed.ts` — початкове наповнення тестовими даними;
- `src/scripts/sql-demo.ts` — демонстрація прямих SQL-запитів.

### 4.2 Підключення до PostgreSQL через Prisma
Prisma Client винесено в окремий модуль, після чого він використовується всередині маршрутів API.

```ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma?: PrismaClient;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: ["warn", "error"],
  });
```

### 4.3 Конфігурація Fastify
У `app.ts` створюється сервер, підключається CORS і реєструються маршрути. Для лабораторної перевірки цього вистачає, щоб frontend на `localhost:3000` працював із backend на `localhost:3001`.

```ts
import Fastify from "fastify";
import cors from "@fastify/cors";

const app = Fastify({ logger: true });

app.register(cors, {
  origin: env.corsOrigin,
  methods: "GET,POST,PATCH,DELETE,OPTIONS",
  allowedHeaders: ["Content-Type"],
  preflight: true,
  optionsSuccessStatus: 204,
});

app.register(healthRoutes);
app.register(async (api) => {
  api.register(userRoutes);
  api.register(categoryRoutes);
  api.register(ticketRoutes);
}, { prefix: "/api" });
```

### 4.4 CRUD-маршрути
Для сутності `Ticket` реалізовано повний набір основних операцій:
- `GET /api/tickets` — отримання списку заявок;
- `GET /api/tickets/:id` — отримання однієї заявки;
- `POST /api/tickets` — створення нової заявки;
- `PATCH /api/tickets/:id` — оновлення заявки;
- `DELETE /api/tickets/:id` — видалення заявки.

Аналогічно винесено маршрути для `users` і `categories`, щоб перевіряти не лише одну таблицю, а й увесь мінімальний набір сутностей, з якими працює система.

### 4.5 Початкові дані
Щоб одразу після запуску не вводити все вручну, реалізовано `prisma/seed.ts`. Скрипт додає кількох користувачів, набір категорій і стартові заявки. Завдяки цьому після `pnpm prisma:seed` можна одразу відкривати сторінки `/tickets`, `/users` і `/categories`, не готуючи вручну базу для демонстрації.

---

## 5. Прямі SQL-запити з Node.js

### 5.1 Підключення через `pg`
Для окремої демонстрації ручної роботи з SQL використано пакет `pg` і скрипт `src/scripts/sql-demo.ts`.

```ts
import { Pool } from "pg";

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});
```

### 5.2 Демонстрація CRUD без ORM
У скрипті послідовно виконуються `SELECT`, `INSERT`, `UPDATE`, `DELETE`. Такий підхід потрібен саме для лабораторної частини, бо показує, що сервер може працювати не тільки через ORM, а й напряму з SQL-запитами.

```ts
const insert = await pool.query(
  `INSERT INTO tickets (title, description, status, priority, creator_id, category_id, created_at, updated_at)
   VALUES ($1, $2, $3, $4, $5, $6, NOW(), NOW())
   RETURNING id, title, status`,
  [
    "SQL demo ticket",
    "Created from pg client script",
    "OPEN",
    "LOW",
    1,
    1,
  ],
);
```

Після вставки запис оновлюється та видаляється, а завершальний `SELECT` показує, що тимчасові зміни були прибрані коректно.

---

## 6. Інтеграція з web-інтерфейсом

### 6.1 Сторінка зі списком заявок
Маршрут `/tickets` отримує список заявок із backend API і відображає їх у таблиці. Додатково на цій сторінці є фільтрація за статусом, пріоритетом і пошуковим рядком, тому перевірити маршрут `GET /api/tickets` можна не лише через Postman, а й безпосередньо через UI.

### 6.2 Детальна сторінка заявки
Маршрут `/tickets/[id]` показує дані однієї заявки та форму керування нею. Через цю сторінку можна змінити назву, опис, статус, пріоритет і категорію, а також видалити запис. Таким чином сторінка виступає інтерфейсом для перевірки `PATCH` і `DELETE` без окремих зовнішніх інструментів.

### 6.3 Форма створення нової заявки
Сторінка `/tickets/new` містить форму створення звернення з вибором автора, категорії та пріоритету. Після надсилання даних виконується `POST /api/tickets`, а новий запис з’являється в системі без ручного втручання в базу.

### 6.4 Довідники користувачів і категорій
Окремі сторінки `/users` та `/categories` дозволяють переглядати відповідні таблиці й додавати нові записи через UI. Для лабораторної роботи це зручно тим, що всі основні таблиці можна продемонструвати як через API-запити, так і через клієнтську частину.

---

## 7. Команди для запуску

Після переходу в корінь репозиторію середовище розгортається через `pnpm` і root-скрипти.

### 7.1 Встановлення залежностей
```bash
pnpm install
```

### 7.2 Запуск PostgreSQL у Docker
```bash
pnpm db:up
pnpm db:ps
pnpm db:check
```

### 7.3 Підготовка Prisma
```bash
pnpm prisma:generate
pnpm prisma:push
pnpm prisma:seed
```

### 7.4 Запуск API та web-частини
```bash
pnpm dev:api
pnpm dev:web
```

### 7.5 Запуск демонстрації SQL
```bash
pnpm sql:demo
```

---

## 8. Результати виконання

![](/assets/labs/lab-2/postgres_start_check_logs.png)
**Запуск і перевірка контейнера PostgreSQL**  
Команди `pnpm db:up`, `pnpm db:ps`, `pnpm db:check` та `pnpm db:logs` підтверджують старт контейнера і готовність СУБД до підключення.

![](/assets/labs/lab-2/prisma_generate_success.png)
**Генерація Prisma Client**  
Команда `pnpm prisma:generate` створює клієнт для роботи з PostgreSQL через ORM Prisma.

![](/assets/labs/lab-2/prisma_push_success.png)
**Синхронізація схеми Prisma з базою даних**  
Команда `pnpm prisma:push` створює або оновлює структуру таблиць відповідно до опису в `schema.prisma`.

![](/assets/labs/lab-2/prisma_seed_success.png)
**Заповнення бази тестовими даними**  
Команда `pnpm prisma:seed` додає початкові записи до таблиць користувачів, категорій і заявок.

![](/assets/labs/lab-2/web_api_running_tmux.png)
**Одночасний запуск frontend і backend**  
У tmux-сесії видно окремі процеси для `pnpm dev:web` та `pnpm dev:api`, що підтверджує спільну роботу клієнтської і серверної частин.

![](/assets/labs/lab-2/health_check.png)
**Перевірка доступності сервера**  
Маршрут `GET /health` підтверджує, що backend-застосунок запущений і готовий приймати запити.

![](/assets/labs/lab-2/users_get_all.png)
**Отримання списку користувачів**  
Запит `GET /api/users` повертає всі наявні записи з базовими даними користувачів helpdesk-системи.

![](/assets/labs/lab-2/users_post_success.png)
**Створення нового користувача**  
Маршрут `POST /api/users` успішно додає нового користувача та повертає його ідентифікатор, ім’я, email і роль.

![](/assets/labs/lab-2/categories_get_all.png)
**Отримання списку категорій**  
Запит `GET /api/categories` повертає перелік категорій разом із кількістю пов’язаних заявок.

![](/assets/labs/lab-2/categories_post_success.png)
**Створення нової категорії**  
Маршрут `POST /api/categories` успішно додає новий запис до таблиці категорій і повертає створений об’єкт.

![](/assets/labs/lab-2/tickets_get_all.png)
**Отримання списку заявок**  
Маршрут `GET /api/tickets` повертає всі заявки з інформацією про автора та категорію.

![](/assets/labs/lab-2/tickets_filter_status_open.png)
**Фільтрація заявок за статусом**  
Запит `GET /api/tickets?status=OPEN` відбирає заявки у відкритому стані та повертає їх разом із пов’язаними даними.

![](/assets/labs/lab-2/sql_demo_crud_execution.png)
**Виконання SQL-операцій без ORM**  
Скрипт `pnpm sql:demo` демонструє послідовне виконання команд `SELECT`, `INSERT`, `UPDATE` і `DELETE` безпосередньо через SQL-запити до PostgreSQL.

![](/assets/labs/lab-2/frontend_tickets_page_full.png)
**Сторінка перегляду всіх заявок**  
На сторінці `Заявки` відображено повний список звернень із колонками ідентифікатора, назви, категорії, статусу, пріоритету, автора та дати створення.

![](/assets/labs/lab-2/frontend_tickets_filtered_open.png)
**Фільтрація заявок у web-інтерфейсі**  
На сторінці списку застосовано фільтр за статусом `Open`, у результаті чого відображено лише один запис, що відповідає заданій умові.

![](/assets/labs/lab-2/frontend_ticket_create_form.png)
**Форма створення нової заявки**  
Сторінка `Нова заявка` дозволяє ввести назву, вибрати категорію, автора, пріоритет і опис перед відправленням даних до Fastify API.

![](/assets/labs/lab-2/frontend_ticket_detail_update_success.png)
**Редагування заявки через web-інтерфейс**  
На сторінці конкретної заявки показано форму для зміни назви, опису, статусу, пріоритету та категорії, а також повідомлення про успішне збереження змін.

![](/assets/labs/lab-2/frontend_categories_create_filled.png)
**Створення нової категорії через UI**  
На сторінці `Категорії` показано заповнену форму створення нового запису та оновлений список категорій після додавання нового елемента.

![](/assets/labs/lab-2/frontend_users_create_filled.png)
**Створення нового користувача через UI**  
На сторінці `Користувачі` показано заповнену форму створення користувача та таблицю, до якої додано новий запис після успішного POST-запиту.

---

## 9. Висновки

У межах лабораторної роботи для helpdesk-проєкту побудовано реляційну базу даних PostgreSQL і підключено її до серверної частини на Node.js. Структуру даних описано через Prisma ORM, після чого реалізовано маршрути для користувачів, категорій і заявок та перевірено повний набір базових CRUD-операцій. Окремо підготовлено скрипт для прямого виконання SQL-запитів через пакет `pg`, щоб показати роботу без ORM-рівня. Додатково backend інтегровано з web-інтерфейсом, тому створення, перегляд, фільтрація, редагування і видалення записів можна демонструвати не лише через Postman, а й безпосередньо через UI. Отримана реалізація вже є робочою основою для подальшого розвитку системи — додавання авторизації, історії статусів, коментарів і ролей доступу.

---

## 10. Перелік використаних джерел
1. Документація PostgreSQL.  
2. Документація Node.js.  
3. Документація Fastify.  
4. Документація Prisma ORM.  
5. Документація `pg` для Node.js.  
6. Документація Next.js.
