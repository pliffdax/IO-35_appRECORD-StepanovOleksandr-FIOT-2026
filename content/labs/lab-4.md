## 1. Тема, мета, посилання

### 1.1 Тема
«Розширені можливості Node.js-додатків: логування, завантаження файлів, моніторинг продуктивності».

### 1.2 Мета
Розширити серверну частину проєкту Helpdesk / Ticket System засобами логування, обробки файлових завантажень і базового моніторингу продуктивності. У межах роботи потрібно фіксувати події та помилки в логах, реалізувати завантаження одного й кількох файлів із валідацією, а також додати endpoint для перегляду стану Node.js-процесу.

### 1.3 Посилання
- Репозиторій власного веб-застосунку (GitHub): [посилання](https://github.com/pliffdax/Helpdesk)
- Репозиторій звітного HTML-документа (GitHub): [посилання](https://github.com/pliffdax/IO-35_appRECORD-StepanovOleksandr-FIOT-2026)
- Звітний HTML-документ (Жива сторінка): [посилання](https://pliffdax.github.io/IO-35_appRECORD-StepanovOleksandr-FIOT-2026/)

---

## 2. Короткі теоретичні відомості

### 2.1 Логування у Node.js-застосунках
Логування використовується для фіксації важливих подій у серверному застосунку: HTTP-запитів, помилок, дій користувачів, часу відповіді та службових повідомлень. Для навчального проєкту це дає змогу швидко перевірити, які маршрути викликалися, з яким статусом завершився запит і чи виникали помилки під час роботи API.

У поточному проєкті використано два рівні логування. Fastify logger виводить HTTP-події в консоль під час розробки, а Winston записує структуровані події у файли `app.log` та `error.log`. Такий підхід відповідає ідеї Morgan/Winston із завдання лабораторної роботи, але адаптований до вже реалізованого backend на Fastify.

### 2.2 Завантаження файлів
Файли з браузера або Postman передаються на сервер у форматі `multipart/form-data`. Звичайний JSON-парсер не обробляє такий формат, тому для Fastify використано плагін `@fastify/multipart`. Він дає змогу читати файл як stream, перевіряти його тип і розмір, а потім зберігати на диск.

У межах роботи дозволено завантажувати файли форматів `jpg`, `png` та `pdf`. Для запобігання надто великим запитам встановлено обмеження розміру файлу.

### 2.3 Моніторинг продуктивності
Базовий моніторинг Node.js-застосунку можна реалізувати через вбудований об'єкт `process`. Він дозволяє отримати:
- `process.uptime()` — час роботи процесу;
- `process.memoryUsage()` — використання оперативної пам'яті;
- `process.cpuUsage()` — використання CPU процесом.

Окремо для кожного HTTP-запиту вимірюється час обробки. Це допомагає побачити, які запити виконуються швидко, а які можуть потребувати оптимізації.

---

## 3. Реалізований функціонал Lab 4

### 3.1 Основні сценарії
У межах лабораторної роботи реалізовано такі можливості:
- файлове логування подій через Winston;
- запис помилок у файл `error.log`;
- вимірювання часу відповіді кожного HTTP-запиту;
- endpoint `GET /status` для перегляду uptime, memoryUsage та cpuUsage;
- endpoint `POST /upload` для завантаження одного файлу;
- endpoint `POST /upload-multiple` для завантаження кількох файлів;
- валідація типу файлу;
- обмеження розміру файлу;
- підготовка PM2-конфігурації для запуску API як окремого процесу.

### 3.2 Адаптація під існуючий проєкт
У завданні лабораторної роботи наведено приклади для Express, Morgan, Winston і Multer. Поточний Helpdesk-проєкт уже побудований на Fastify, тому реалізацію виконано без переписування backend-архітектури:
- замість Morgan використано стандартне HTTP-логування Fastify;
- для файлових логів використано Winston;
- замість Multer використано `@fastify/multipart`;
- моніторинг реалізовано через стандартний API Node.js `process`.

Такий підхід дозволяє виконати вимоги лабораторної роботи, не створюючи окремий демонстраційний сервер і не дублюючи вже існуючу структуру застосунку.

---

## 4. Реалізація backend-частини

### 4.1 Підключення multipart і middleware логування
Основні підключення виконано у файлі `apps/api/src/app.ts`. До Fastify-застосунку додано плагін `@fastify/multipart`, а також hooks `onRequest` та `onResponse`, які вимірюють час обробки кожного запиту.

```ts
app.register(multipart, {
  limits: {
    fileSize: 5 * 1024 * 1024,
    files: 5,
  },
});

app.addHook("onRequest", async (request) => {
  requestStartTimes.set(request, Date.now());
});

app.addHook("onResponse", async (request, reply) => {
  const startedAt = requestStartTimes.get(request) ?? Date.now();
  logRequestDuration({
    method: request.method,
    url: request.url,
    statusCode: reply.statusCode,
    durationMs: Date.now() - startedAt,
  });
});
```

Після кожного завершеного запиту у файл логів потрапляє HTTP-метод, URL, статус відповіді та тривалість обробки в мілісекундах.

### 4.2 Файловий логер Winston
Для логування створено окремий модуль `apps/api/src/lib/logger.ts`. Він створює директорію `apps/api/logs` і записує події у два файли:
- `app.log` — загальні події та інформаційні повідомлення;
- `error.log` — помилки.

```ts
export const appLogger = winston.createLogger({
  level: "info",
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json(),
  ),
  transports: [
    new winston.transports.File({
      filename: path.join(logDir, "error.log"),
      level: "error",
    }),
    new winston.transports.File({
      filename: path.join(logDir, "app.log"),
    }),
  ],
});
```

У разі помилки загальний error handler додатково записує її повідомлення та stack trace у Winston.

### 4.3 Моніторинг стану сервера
Маршрут `GET /status` реалізовано у файлі `apps/api/src/routes/monitoring.ts`. Він повертає поточний стан Node.js-процесу.

```ts
app.get("/status", async () => {
  const memoryUsage = process.memoryUsage();
  const cpuUsage = process.cpuUsage();

  return {
    status: "ok",
    uptime: process.uptime(),
    memoryUsage,
    cpuUsage,
    timestamp: new Date().toISOString(),
  };
});
```

Цей endpoint зручно використовувати для швидкої перевірки того, що сервер працює, і для демонстрації базових метрик продуктивності.

### 4.4 Завантаження одного файлу
Для завантаження одного файлу реалізовано маршрут `POST /upload`. Він очікує `multipart/form-data` із полем `file`, перевіряє файл і зберігає його у директорію `apps/api/uploads`.

```ts
app.post("/upload", async (request, reply) => {
  const file = await request.file({
    limits: {
      fileSize: maxFileSizeBytes,
      files: 1,
    },
  });

  if (!file) {
    return reply.status(400).send({ message: "file is required" });
  }
});
```

Після успішного завантаження API повертає JSON із назвою збереженого файлу, початковою назвою, MIME-типом, розміром і шляхом на сервері.

### 4.5 Завантаження кількох файлів
Маршрут `POST /upload-multiple` приймає кілька файлів, послідовно обробляє їх через async iterator і повертає масив збережених файлів.

```ts
for await (const file of request.files({
  limits: {
    fileSize: maxFileSizeBytes,
    files: 5,
  },
})) {
  savedFiles.push(await saveMultipartFile(file));
}
```

Максимальна кількість файлів у межах одного запиту — 5.

### 4.6 Валідація файлів
Перед збереженням перевіряється MIME-тип файлу. У лабораторній версії дозволено:
- `image/jpeg`;
- `image/png`;
- `application/pdf`.

```ts
if (!allowedMimeTypes.has(file.mimetype)) {
  file.file.resume();
  throw new UploadValidationError("only jpg, png and pdf files are allowed");
}
```

Якщо файл має непідтримуваний тип, API повертає статус `400` і повідомлення про помилку.

### 4.7 PM2-конфігурація
Для демонстрації запуску через менеджер процесів додано файл `ecosystem.config.cjs`.

```text
module.exports = {
  apps: [
    {
      name: "helpdesk-api",
      cwd: __dirname,
      script: "apps/api/dist/server.js",
      env: {
        NODE_ENV: "production",
      },
    },
  ],
};
```

Після збірки backend можна запустити командою `pm2 start ecosystem.config.cjs`.

---

## 5. Postman-колекція для перевірки

Для лабораторної роботи підготовлено окрему Postman-колекцію:
- `postman/lab-4/helpdesk_lab4_monitoring_upload_postman_collection.json`;
- `postman/lab-4/helpdesk_lab4_monitoring_upload_postman_environment.json`.

Колекція містить такі запити:
- `GET /health`;
- `GET /status`;
- `POST /upload - single file`;
- `POST /upload-multiple - several files`;
- `POST /upload - invalid file type`;
- `POST /upload - missing multipart body`.

Файлові запити потребують ручного вибору файлів у полі `form-data`. Для успішних сценаріїв потрібно вибрати `.jpg`, `.png` або `.pdf`, а для помилки валідації — файл іншого типу, наприклад `.txt`.

---

## 6. Команди для запуску

### 6.1 Встановлення залежностей
```bash
pnpm install
```

### 6.2 Запуск API у режимі розробки
```bash
pnpm dev:api
```

Після запуску API доступний за адресою `http://localhost:3001`.

### 6.3 Перевірка збірки
```bash
pnpm --filter @repo/api lint
pnpm --filter @repo/api build
```

### 6.4 Перегляд логів
```bash
tail -n 20 apps/api/logs/app.log
tail -n 20 apps/api/logs/error.log
```

### 6.5 Запуск через PM2
```bash
pnpm --filter @repo/api build
pm2 start ecosystem.config.cjs
pm2 list
pm2 logs helpdesk-api
pm2 restart helpdesk-api
```

---

## 7. Результати виконання

![Успішний запуск backend-застосунку Helpdesk API](/assets/labs/lab-4/api_dev_server_running.png)
**Рис. 1 – Успішний запуск backend-застосунку Helpdesk API.**

![Перевірка маршруту GET status у Postman](/assets/labs/lab-4/postman_status_metrics.png)
**Рис. 2 – Перевірка маршруту `GET /status` з uptime, memoryUsage та cpuUsage.**

![Успішне завантаження одного PDF-файлу через Postman](/assets/labs/lab-4/postman_upload_single_success.png)
**Рис. 3 – Успішне завантаження одного PDF-файлу через `POST /upload`.**

![Помилка валідації типу файлу у Postman](/assets/labs/lab-4/postman_upload_invalid_type.png)
**Рис. 4 – Помилка валідації під час завантаження файлу непідтримуваного типу.**

![Успішне завантаження кількох файлів через Postman](/assets/labs/lab-4/postman_upload_multiple_success.png)
**Рис. 5 – Успішне завантаження кількох файлів через `POST /upload-multiple`.**

![Вміст файлу app log з часом відповіді HTTP-запитів](/assets/labs/lab-4/app_log_request_duration.png)
**Рис. 6 – Вміст файлу `app.log` з HTTP-запитами та часом відповіді.**

![Вміст файлу error log після помилкового запиту](/assets/labs/lab-4/error_log_failed_request.png)
**Рис. 7 – Вміст файлу `error.log` після помилкового запиту.**

![Процес helpdesk-api у PM2](/assets/labs/lab-4/pm2_helpdesk_api_process.png)
**Рис. 8 – Запуск API через PM2 та перегляд процесу `helpdesk-api`.**

---

## 8. Висновки

У межах лабораторної роботи серверну частину Helpdesk / Ticket System розширено можливостями логування, завантаження файлів і моніторингу стану процесу. Для файлових логів використано Winston, який записує інформаційні події в `app.log`, а помилки — в `error.log`. Додатково для кожного HTTP-запиту вимірюється час відповіді, що дозволяє контролювати базову продуктивність API.

Для роботи з файлами реалізовано два маршрути: завантаження одного файлу та завантаження кількох файлів. Додано перевірку MIME-типу й обмеження розміру файлу, тому сервер приймає лише дозволені формати та повертає зрозумілу помилку у випадку некоректного запиту. Маршрут `GET /status` повертає uptime, використання пам'яті та CPU, що демонструє базовий моніторинг Node.js-застосунку. Також підготовлено конфігурацію PM2 для запуску backend як окремого керованого процесу.

---

## 9. Перелік використаних джерел
1. Документація Node.js (`process`, `stream`, `fs`).  
2. Документація Fastify.  
3. Документація `@fastify/multipart`.  
4. Документація Winston.  
5. Документація PM2.
