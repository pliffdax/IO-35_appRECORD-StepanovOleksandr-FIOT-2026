## 1. Тема, мета, посилання

### 1.1 Тема
«Безпека та продуктивність серверних додатків. Безпека Node.js-додатків. Оптимізація запитів і кешування. Тестування API».

### 1.2 Мета
Розширити backend-частину Helpdesk / Ticket System засобами базового захисту, обмеження кількості запитів, кешування, оптимізації REST API та автоматизованого тестування. У межах роботи потрібно показати, як сервер захищається від типових небезпечних сценаріїв, як зменшується навантаження на повторні запити та як перевіряється працездатність API.

### 1.3 Посилання
- Репозиторій власного веб-застосунку (GitHub): [посилання](https://github.com/pliffdax/Helpdesk)
- Репозиторій звітного HTML-документа (GitHub): [посилання](https://github.com/pliffdax/IO-35_appRECORD-StepanovOleksandr-FIOT-2026)
- Звітний HTML-документ (Жива сторінка): [посилання](https://pliffdax.github.io/IO-35_appRECORD-StepanovOleksandr-FIOT-2026/)

---

## 2. Короткі теоретичні відомості

### 2.1 Безпека Node.js API
Для backend-застосунків важливо захищати HTTP-рівень, перевіряти вхідні дані та обмежувати надмірну кількість запитів. У проєкті вже реалізовано хешування паролів, токенний доступ і валідацію полів, а в межах цієї лабораторної роботи додано захисні HTTP-заголовки та rate limiting.

Helmet встановлює набір HTTP-заголовків, які зменшують ризики XSS, MIME sniffing та інших типових web-атак. Rate limiting обмежує кількість запитів від одного клієнта за певний проміжок часу, що допомагає зменшити ризик brute-force або простого перевантаження API.

### 2.2 Оптимізація API
Оптимізація REST API включає зменшення обсягу відповіді, скорочення кількості звернень до бази даних і використання індексів. Для списку заявок додано пагінацію через параметри `page` та `limit`, тому API може повертати не весь список одразу, а лише потрібну сторінку.

Також у Prisma-схему додано індекси для полів, які часто використовуються у фільтрах або сортуванні: `status`, `priority`, `createdAt`.

### 2.3 Кешування
Кешування зберігає результат запиту на короткий час, щоб повторний запит не звертався до бази даних. Для лабораторної роботи використано простий in-memory cache. Він не потребує окремого Redis-сервера, але добре демонструє сам принцип кешування.

У відповідях API використовується заголовок `x-cache`:
- `MISS` — дані отримано з бази та записано в кеш;
- `HIT` — дані повернено з кешу.

Після створення або зміни сутності відповідний кеш очищається.

### 2.4 Тестування API
Автоматизовані тести дозволяють швидко перевірити, що базова поведінка API не зламалася після змін. Для цього використано Vitest і Fastify `inject`, який дозволяє виконувати HTTP-запити до застосунку без запуску окремого сервера на порту.

---

## 3. Реалізований функціонал Lab 5

### 3.1 Основні сценарії
У межах лабораторної роботи реалізовано:
- захисні HTTP-заголовки через `@fastify/helmet`;
- обмеження частоти запитів через `@fastify/rate-limit`;
- збереження існуючої валідації даних користувача;
- in-memory cache для списку категорій;
- cache headers `x-cache: MISS` та `x-cache: HIT`;
- пагінацію для `GET /api/tickets`;
- індекси в Prisma-схемі для фільтрів і сортування заявок;
- автоматизовані тести для безпеки, валідації та кешу;
- Postman-колекцію для ручної перевірки результатів.

### 3.2 Адаптація під існуючий проєкт
У завданні лабораторної роботи наведено приклади для Express, Redis і Jest/Supertest. Поточний проєкт уже побудовано на Fastify, Prisma і PostgreSQL, тому реалізацію виконано без переписування backend-частини:
- замість Express middleware використано Fastify plugins;
- замість Redis для навчальної демонстрації використано in-memory cache;
- замість Supertest використано Fastify `inject`;
- існуюча JWT-like автентифікація, refresh token і хешування паролів залишилися без дублювання.

---

## 4. Реалізація backend-частини

### 4.1 Helmet і rate limiting
У файлі `apps/api/src/app.ts` підключено два Fastify-плагіни:

```ts
app.register(helmet);

app.register(rateLimit, {
  max: 100,
  timeWindow: "1 minute",
});
```

Після цього відповіді API містять захисні заголовки, наприклад `X-Content-Type-Options: nosniff`, а також заголовки rate limit: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.

### 4.2 In-memory cache
Для кешування створено модуль `apps/api/src/lib/cache.ts`. Він зберігає дані в `Map`, має TTL за замовчуванням 60 секунд і дозволяє очищати записи за префіксом.

```ts
const cache = new Map<string, CacheEntry<unknown>>();
const defaultTtlMs = 60 * 1000;

export function getCached<T>(key: string) {
  const entry = cache.get(key);

  if (!entry) {
    return null;
  }

  if (entry.expiresAt <= Date.now()) {
    cache.delete(key);
    return null;
  }

  return entry.value as T;
}
```

Такий кеш достатній для лабораторної демонстрації, бо дозволяє побачити різницю між першим і повторним запитом без підключення додаткової інфраструктури.

### 4.3 Кешування категорій
Маршрут `GET /api/categories` спочатку перевіряє кеш. Якщо запис знайдено, API повертає дані з `x-cache: HIT`; якщо ні — отримує дані з PostgreSQL, записує їх у кеш і повертає `x-cache: MISS`.

```ts
const cached = getCached(categoriesCacheKey);

if (cached) {
  reply.header("x-cache", "HIT");
  return cached;
}

reply.header("x-cache", "MISS");
setCached(categoriesCacheKey, response);
```

Після створення нової категорії кеш очищається:

```ts
deleteCachedByPrefix("categories:");
```

### 4.4 Пагінація списку заявок
Для маршруту `GET /api/tickets` додано параметри `page` і `limit`. Значення за замовчуванням:
- `page = 1`;
- `limit = 10`;
- максимальний `limit = 50`.

```ts
const page = parsePositiveInteger(query.page, 1);
const limit = Math.min(parsePositiveInteger(query.limit, 10), 50);
```

Запит до Prisma використовує `skip` і `take`, а відповідь містить службову інформацію:

```ts
meta: {
  total,
  page,
  limit,
  totalPages,
  source: "database",
}
```

Це зменшує обсяг відповіді та робить API зручнішим для frontend-таблиць і списків.

### 4.5 Індекси в Prisma
Для оптимізації фільтрації та сортування заявок у `schema.prisma` додано індекси:

```prisma
@@index([status])
@@index([priority])
@@index([createdAt])
```

Ці поля використовуються в маршруті списку заявок для фільтрації за статусом, пріоритетом і сортування за датою створення.

---

## 5. Автоматизоване тестування

### 5.1 Тести API
Для перевірки API додано файл `apps/api/src/app.test.ts`. Він перевіряє:
- наявність security headers;
- наявність rate limit headers;
- валідацію `POST /api/users` без звернення до бази даних.

```ts
const response = await app.inject({
  method: "GET",
  url: "/health",
});

expect(response.statusCode).toBe(200);
expect(response.headers["x-content-type-options"]).toBe("nosniff");
expect(response.headers["x-ratelimit-limit"]).toBeDefined();
```

### 5.2 Тести кешу
Окремо додано `apps/api/src/lib/cache.test.ts`, де перевіряється запис, читання та очищення кешу за префіксом.

```ts
setCached("lab5:test:one", { source: "database" });

expect(getCached("lab5:test:one")).toEqual({
  source: "database",
});
```

Тести запускаються командою:

```bash
pnpm --filter @repo/api test
```

---

## 6. Postman-колекція для перевірки

Для лабораторної роботи підготовлено окрему Postman-колекцію:
- `postman/lab-5/helpdesk_lab5_security_performance_collection.json`;
- `postman/lab-5/helpdesk_lab5_security_performance_environment.json`.

Колекція містить такі запити:
- `GET /health - Helmet and rate limit headers`;
- `POST /api/users - validation error`;
- `GET /api/tickets?page=1&limit=2 - paginated tickets`;
- `GET /api/categories - cache MISS`;
- `GET /api/categories - cache HIT`;
- `POST /api/categories - invalidate cache`;
- `GET /api/categories - cache MISS after create`.

Для коректної демонстрації `MISS` і `HIT` бажано запускати запити по порядку після перезапуску API.

---

## 7. Команди для запуску

### 7.1 Запуск інфраструктури
```bash
pnpm db:up
pnpm db:check
```

### 7.2 Підготовка бази даних
```bash
pnpm prisma:generate
pnpm prisma:push
pnpm prisma:seed
```

### 7.3 Запуск API
```bash
pnpm dev:api
```

### 7.4 Перевірка типів, тестів і збірки
```bash
pnpm --filter @repo/api lint
pnpm --filter @repo/api test
pnpm --filter @repo/api build
```

---

## 8. Результати виконання

![Успішний запуск API для лабораторної роботи 5](/assets/labs/lab-5/api_dev_server_running.png)
**Рис. 1 – Успішний запуск backend-застосунку Helpdesk API для перевірки Lab 5.**

![Security headers і rate limit headers у Postman](/assets/labs/lab-5/postman_security_headers.png)
**Рис. 2 – Перевірка `GET /health`: захисні заголовки Helmet і заголовки rate limiting.**

![Помилка валідації користувача у Postman](/assets/labs/lab-5/postman_user_validation_error.png)
**Рис. 3 – Помилка валідації під час `POST /api/users` з порожнім body.**

![Пагінація заявок у Postman](/assets/labs/lab-5/postman_tickets_pagination.png)
**Рис. 4 – Перевірка пагінації `GET /api/tickets?page=1&limit=2`.**

![Перший запит категорій з cache MISS](/assets/labs/lab-5/postman_categories_cache_miss.png)
**Рис. 5 – Перший запит `GET /api/categories` із заголовком `x-cache: MISS`.**

![Повторний запит категорій з cache HIT](/assets/labs/lab-5/postman_categories_cache_hit.png)
**Рис. 6 – Повторний запит `GET /api/categories` із заголовком `x-cache: HIT`.**

![Очищення кешу після створення категорії](/assets/labs/lab-5/postman_categories_cache_invalidation.png)
**Рис. 7 – Створення нової категорії через `POST /api/categories` для очищення кешу.**

![Запуск автоматизованих тестів Vitest](/assets/labs/lab-5/vitest_api_tests_success.png)
**Рис. 8 – Успішний запуск автоматизованих тестів через `pnpm --filter @repo/api test`.**

![Перевірка TypeScript і збірки API](/assets/labs/lab-5/api_lint_build_success.png)
**Рис. 9 – Успішна перевірка типів і production-збірка backend API.**

---

## 9. Висновки

У межах лабораторної роботи backend-частину Helpdesk / Ticket System розширено механізмами безпеки, кешування, оптимізації та тестування. Через `@fastify/helmet` додано захисні HTTP-заголовки, а через `@fastify/rate-limit` — обмеження частоти запитів. Існуюча валідація даних залишилася частиною захисту API, оскільки некоректні запити відхиляються до звернення до бази даних.

Для покращення продуктивності реалізовано in-memory cache для довідника категорій і пагінацію для списку заявок. Повторні запити до категорій можуть повертатися з кешу, що видно за заголовком `x-cache: HIT`, а після створення нової категорії кеш очищується. Для списку заявок додано `page` і `limit`, а також індекси в Prisma-схемі для полів, які використовуються у фільтрації та сортуванні.

Окремо підготовлено автоматизовані тести на Vitest. Вони перевіряють security headers, rate limit headers, базову валідацію та роботу кешу. Це дає мінімальний, але корисний набір перевірок для подальшого розвитку API.

---

## 10. Перелік використаних джерел
1. Документація Fastify.  
2. Документація `@fastify/helmet`.  
3. Документація `@fastify/rate-limit`.  
4. Документація Prisma ORM.  
5. Документація Vitest.  
6. Документація Node.js.
