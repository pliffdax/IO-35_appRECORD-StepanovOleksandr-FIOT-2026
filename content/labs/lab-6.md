## 1. Тема, мета, посилання

### 1.1 Тема

«Документування API за допомогою Swagger. Деплой Node.js-додатку. Підсумковий проєкт: REST API з базою даних та Swagger документацією».

### 1.2 Мета

Інтегрувати Swagger/OpenAPI-документацію в існуючий backend Helpdesk / Ticket System, описати основні endpoint-и REST API, перевірити API через Swagger UI та підготувати застосунок до production-запуску і деплою.

### 1.3 Посилання

- Репозиторій власного веб-застосунку (GitHub): [посилання](https://github.com/pliffdax/Helpdesk)
- Репозиторій звітного HTML-документа (GitHub): [посилання](https://github.com/pliffdax/IO-35_appRECORD-StepanovOleksandr-FIOT-2026)
- Звітний HTML-документ (Жива сторінка): [посилання](https://pliffdax.github.io/IO-35_appRECORD-StepanovOleksandr-FIOT-2026/)
- Вебзастосунок, розгорнутий на Render (Жива сторінка): [посилання](https://helpdesk-ffxi.onrender.com/)

---

## 2. Короткі теоретичні відомості

### 2.1 REST API

REST API — це спосіб організації взаємодії між клієнтом і сервером через HTTP-методи. У проєкті Helpdesk API використовується для роботи з користувачами, категоріями, заявками, авторизацією, завантаженням файлів і моніторингом.

Основні HTTP-методи:

- `GET` — отримання даних;
- `POST` — створення нового ресурсу;
- `PATCH` — часткове оновлення;
- `DELETE` — видалення.

### 2.2 Swagger та OpenAPI

OpenAPI Specification — це стандарт опису REST API. Він дозволяє формально описати маршрути, параметри, body, відповіді, схеми даних і авторизацію. Swagger UI використовує OpenAPI-документ і на його основі створює web-інтерфейс для перегляду та тестування API.

Для лабораторної роботи використано пакети:

- `@fastify/swagger` — генерація та підключення OpenAPI-документа;
- `@fastify/swagger-ui` — web-інтерфейс Swagger UI.

### 2.3 Деплой Node.js-застосунку

Деплой — це публікація backend-застосунку на сервері або хмарній платформі. Для Node.js API потрібно підготувати production-збірку, змінні середовища, підключення до бази даних і команду старту.

У поточному monorepo-проєкті backend уже має production-команди:

- `pnpm --filter @repo/api build`;
- `pnpm --filter @repo/api start`.

Також API читає порт із `PORT`, що потрібно для Render, Railway або іншої PaaS-платформи.

---

## 3. Реалізований функціонал Lab 6

### 3.1 Основні сценарії

У межах лабораторної роботи реалізовано:

- OpenAPI 3.0.3 документ для Helpdesk API;
- Swagger UI за маршрутом `GET /docs`;
- JSON-специфікацію за маршрутом `GET /docs/json`;
- опис основних endpoint-ів API;
- опис моделей `User`, `Category`, `Ticket`, `AuthResponse`, `ErrorResponse`;
- Bearer token security scheme для захищених маршрутів;
- Postman-колекцію для перевірки OpenAPI JSON і documented endpoints;
- production-перевірку через `build` і `start`.

### 3.2 Адаптація під існуючий проєкт

У завданні лабораторної роботи приклади наведено для Express і MySQL. У поточному проєкті backend реалізовано на Fastify, а база даних — PostgreSQL через Prisma ORM. Це не змінює суті лабораторної роботи: API вже є REST API з CRUD-операціями, базою даних і авторизацією, тому Swagger інтегровано без переписування серверної частини.

---

## 4. Реалізація Swagger/OpenAPI

### 4.1 OpenAPI-документ

OpenAPI-специфікацію винесено в окремий файл `apps/api/src/docs/openapi.ts`. У документі описано загальну інформацію про API, сервер, теги, схеми даних і маршрути.

```ts
export const openApiDocument = {
  openapi: "3.0.3",
  info: {
    title: "Helpdesk API",
    version: "1.0.0",
    description:
      "REST API for the Helpdesk / Ticket System laboratory project.",
  },
  servers: [
    {
      url: "http://localhost:3001",
      description: "Local development API",
    },
  ],
};
```

Документ містить 10 основних paths, серед яких `/health`, `/status`, `/api/auth/register`, `/api/auth/login`, `/api/auth/me`, `/api/users`, `/api/categories`, `/api/tickets` і `/api/tickets/{id}`.

### 4.2 Підключення Swagger до Fastify

У файлі `apps/api/src/app.ts` підключено `@fastify/swagger` у static mode та `@fastify/swagger-ui`.

```ts
app.register(swagger, {
  mode: "static",
  specification: {
    document: openApiDocument,
  },
});

app.register(swaggerUi, {
  routePrefix: "/docs",
  uiConfig: {
    docExpansion: "list",
    deepLinking: true,
  },
});
```

Після запуску backend документація доступна за адресами:

- `http://localhost:3001/docs`;
- `http://localhost:3001/docs/json`.

### 4.3 Опис моделей даних

У OpenAPI-документі описано основні моделі:

- `User` — користувач системи;
- `Category` — категорія заявки;
- `Ticket` — заявка helpdesk-системи;
- `AuthResponse` — відповідь після реєстрації або входу;
- `ErrorResponse` — стандартна помилка API.

```ts
User: {
  type: "object",
  properties: {
    id: { type: "integer", example: 1 },
    name: { type: "string", example: "Olena Support" },
    email: { type: "string", example: "olena@example.com" },
    role: { type: "string", enum: ["USER", "AGENT", "ADMIN"] },
  },
}
```

### 4.4 Авторизація у Swagger

Для захищених маршрутів додано схему `bearerAuth`.

```ts
securitySchemes: {
  bearerAuth: {
    type: "http",
    scheme: "bearer",
    bearerFormat: "JWT",
  },
}
```

Маршрут `GET /api/auth/me` у документації позначено як захищений і він очікує Bearer token.

---

## 5. Перевірка через Swagger UI

Після запуску API потрібно відкрити `http://localhost:3001/docs`. Swagger UI показує список груп endpoint-ів:

- Health;
- Auth;
- Users;
- Categories;
- Tickets;
- Monitoring.

Через Swagger UI можна виконати запит `GET /health`, переглянути JSON-відповідь і перевірити HTTP-код `200`. Також можна переглянути схеми body для `POST /api/auth/login`, `POST /api/users`, `POST /api/tickets` і параметри пагінації для `GET /api/tickets`.

---

## 6. Postman-колекція для перевірки

Для лабораторної роботи підготовлено окрему Postman-колекцію:

- `postman/lab-6/helpdesk_lab6_swagger_collection.json`;
- `postman/lab-6/helpdesk_lab6_swagger_environment.json`.

Колекція містить такі запити:

- `GET /docs/json - OpenAPI spec`;
- `GET /health - documented health endpoint`;
- `GET /api/categories`;
- `GET /api/tickets?page=1&limit=2`;
- `POST /api/auth/register`;
- `POST /api/auth/login`;
- `GET /api/auth/me - Bearer token`.

Після login колекція автоматично записує `accessToken` в environment, щоб наступний запит `GET /api/auth/me` виконався з Bearer token.

---

## 7. Production-запуск і деплой

### 7.1 Production-збірка

Backend збирається командою:

```bash
pnpm --filter @repo/api build
```

Після цього API можна запустити командою:

```bash
pnpm --filter @repo/api start
```

### 7.2 Змінні середовища

Для production-середовища потрібні:

- `PORT`;
- `HOST`;
- `DATABASE_URL`;
- `CORS_ORIGIN`;
- `AUTH_TOKEN_SECRET`;
- `AUTH_TOKEN_TTL_SECONDS`;
- `REFRESH_TOKEN_TTL_SECONDS`.

Файл `apps/api/.env.example` уже містить приклад потрібних змінних. Для Render або Railway потрібно перенести ці змінні в налаштування сервісу й підключити PostgreSQL database URL.

### 7.3 Типові налаштування Render

Для деплою backend API на Render можна використати такі значення:

- Build Command: `pnpm install && pnpm --filter @repo/api build`;
- Start Command: `pnpm --filter @repo/api start`;
- Root Directory: `app`;
- Environment: `Node`;
- Database: PostgreSQL через `DATABASE_URL`.

---

## 8. Команди для запуску

### 8.1 Запуск інфраструктури

```bash
pnpm db:up
pnpm db:check
```

### 8.2 Підготовка бази

```bash
pnpm prisma:generate
pnpm prisma:push
pnpm prisma:seed
```

### 8.3 Запуск API

```bash
pnpm dev:api
```

### 8.4 Перевірка API, тестів і збірки

```bash
pnpm --filter @repo/api lint
pnpm --filter @repo/api test
pnpm --filter @repo/api build
```

---

## 9. Результати виконання

![Успішний запуск API для Swagger документації](/assets/labs/lab-6/api_dev_server_running.png)
**Рис. 1 – Успішний запуск backend-застосунку Helpdesk API для перевірки Swagger.**

![Swagger UI Helpdesk API](/assets/labs/lab-6/swagger_ui_opened.png)
**Рис. 2 – Swagger UI за адресою `http://localhost:3001/docs` зі списком endpoint-ів.**

![OpenAPI JSON документ](/assets/labs/lab-6/swagger_openapi_json.png)
**Рис. 3 – JSON-специфікація OpenAPI за маршрутом `GET /docs/json`.**

![Try it out для health endpoint у Swagger UI](/assets/labs/lab-6/swagger_try_health_success.png)
**Рис. 4 – Виконання `GET /health` через Swagger UI та відповідь сервера.**

![Auth login schema у Swagger UI](/assets/labs/lab-6/swagger_auth_login_schema.png)
**Рис. 5 – Опис request body для `POST /api/auth/login` у Swagger UI.**

![Перевірка OpenAPI JSON у Postman](/assets/labs/lab-6/postman_openapi_json_success.png)
**Рис. 6 – Перевірка `GET /docs/json` через Postman.**

![Перевірка захищеного маршруту auth me через Postman](/assets/labs/lab-6/postman_auth_me_success.png)
**Рис. 7 – Успішний запит `GET /api/auth/me` з Bearer token через Postman.**

![Production build і start API](/assets/labs/lab-6/api_build_start_success.png)
**Рис. 8 – Production-збірка та запуск backend API через `build` і `start`.**

![Налаштування деплою Render](/assets/labs/lab-6/render_deploy_settings.png)
**Рис. 9 – Приклад налаштувань деплою Node.js API на Render.**

---

## 10. Висновки

У межах лабораторної роботи до Helpdesk / Ticket System інтегровано Swagger/OpenAPI-документацію. Для API створено OpenAPI 3.0.3 документ із описом основних маршрутів, моделей даних, параметрів, body-запитів, відповідей і Bearer token авторизації. Swagger UI доступний за маршрутом `/docs`, а JSON-специфікація — за `/docs/json`.

Документація охоплює основні частини підсумкового backend-проєкту: health check, авторизацію, користувачів, категорії, заявки, upload і моніторинг. API можна перевіряти як через Swagger UI, так і через Postman-колекцію. Окремо описано production-збірку, запуск через `start` і базові налаштування деплою для Render/Railway.

Отриманий результат завершує серію лабораторних робіт: проєкт має базу даних, CRUD API, авторизацію, логування, upload, моніторинг, кешування, тести та Swagger-документацію.

---

## 11. Перелік використаних джерел

1. Документація OpenAPI Specification.
2. Документація Swagger UI.
3. Документація `@fastify/swagger`.
4. Документація `@fastify/swagger-ui`.
5. Документація Fastify.
6. Документація Render/Railway для деплою Node.js-застосунків.
