## 1. Тема, мета, посилання

### 1.1 Тема
«Реєстрація та авторизація користувачів у веб-застосунку. Захист маршрутів. Робота з токеном доступу та профілем користувача».

### 1.2 Мета
Реалізувати в проєкті Helpdesk / Ticket System модуль автентифікації та базової авторизації: створення облікового запису, вхід у систему, отримання даних поточного користувача, оновлення профілю, зміну пароля, вихід із системи, а також захист окремих маршрутів на frontend і backend.

### 1.3 Посилання
- Репозиторій власного веб-застосунку (GitHub): [посилання](https://github.com/pliffdax/Helpdesk)
- Репозиторій звітного HTML-документа (GitHub): [посилання](https://github.com/pliffdax/IO-35_appRECORD-StepanovOleksandr-FIOT-2026)
- Звітний HTML-документ (Жива сторінка): [посилання](https://pliffdax.github.io/IO-35_appRECORD-StepanovOleksandr-FIOT-2026/)

---

## 2. Короткі теоретичні відомості

### 2.1 Автентифікація та авторизація
У web-застосунках необхідно відрізняти дві базові задачі. **Автентифікація** відповідає на питання, хто саме працює із системою, а **авторизація** визначає, які саме дії цей користувач має право виконувати. Для helpdesk-системи це критично, оскільки звичайний користувач, агент підтримки та адміністратор мають різні ролі й різні сценарії роботи.

### 2.2 Токенний підхід
У межах цієї лабораторної роботи використано підхід з **access token**, який видається після реєстрації або входу. Далі клієнт передає цей токен у заголовку `Authorization: Bearer <token>` під час звернення до захищених маршрутів API. Це дозволяє відокремити frontend-частину від backend-логіки перевірки доступу.

### 2.3 Захист паролів
Пароль не повинен зберігатися у відкритому вигляді. У реалізації проєкту для цього використано хешування пароля перед збереженням у базі даних. Такий підхід зменшує ризики витоку чутливих даних і є базовою вимогою для будь-якої системи з обліковими записами.

### 2.4 Захищені маршрути
Окрім перевірки токена на рівні API, у frontend-частині важливо не дозволяти неавторизованому користувачу потрапляти на сторінки профілю. Для цього застосовується middleware-перевірка, яка перенаправляє користувача на сторінку входу або, навпаки, не дозволяє вже авторизованому користувачу повторно заходити на сторінки `/login` та `/register`.

---

## 3. Реалізований функціонал Lab 3

### 3.1 Основні сценарії
У межах лабораторної роботи реалізовано такі основні сценарії:
- реєстрація нового користувача;
- вхід у систему за email і паролем;
- отримання інформації про поточного користувача;
- редагування імені та email профілю;
- зміна пароля;
- вихід із системи;
- захист сторінки профілю та перевірка токена на backend.

### 3.2 Ролі користувачів
У моделі даних уже використовується перелік ролей:
- `USER`
- `AGENT`
- `ADMIN`

На поточному етапі основна увага приділена базовому механізму ідентифікації користувача й передачі його ролі всередині токена. Це створює основу для подальшого розмежування прав доступу до окремих дій у системі.

### 3.3 Захищені маршрути API
Для модуля автентифікації реалізовано окремі маршрути:
- `POST /api/auth/register`
- `POST /api/auth/login`
- `GET /api/auth/me`
- `PATCH /api/auth/profile`
- `POST /api/auth/change-password`
- `POST /api/auth/logout`

Маршрути `me`, `profile`, `change-password` і `logout` є захищеними та вимагають коректний Bearer token.

---

## 4. Реалізація backend-частини

### 4.1 Структура auth-модуля
Основна логіка Lab 3 у backend-частині зосереджена в таких файлах:
- `apps/api/src/routes/auth.ts` — HTTP-маршрути для реєстрації, входу, профілю, зміни пароля та виходу;
- `apps/api/src/lib/auth.ts` — створення та перевірка токена, хешування пароля, middleware перевірки доступу;
- `apps/api/prisma/schema.prisma` — розширена модель користувача з полем для хешу пароля;
- `apps/api/src/app.ts` — підключення auth-маршрутів і налаштування заголовка `Authorization`.

### 4.2 Розширення моделі користувача
Для підтримки автентифікації в Prisma-схему користувача додано поле `passwordHash`, а також службові поля дат створення та оновлення.

```prisma
model User {
  id           Int      @id @default(autoincrement())
  name         String
  email        String   @unique
  passwordHash String?  @map("password_hash")
  role         Role     @default(USER)
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")

  @@map("users")
}
```

Наявність `@unique` для `email` гарантує, що система не створить двох користувачів з однаковою адресою електронної пошти.

### 4.3 Хешування пароля
Перед збереженням пароля використовується функція хешування. У реалізації для цього застосовано `scrypt`, а до результату додається випадкова сіль.

```ts
export async function hashPassword(password: string) {
  const salt = randomBytes(16).toString("hex");
  const derivedKey = (await scrypt(password, salt, 64)) as Buffer;
  return `${salt}:${derivedKey.toString("hex")}`;
}
```

Під час входу в систему пароль не порівнюється як звичайний текстовий рядок. Замість цього виконується повторне обчислення хешу та безпечне порівняння збереженого значення.

### 4.4 Формування access token
Після успішної реєстрації або входу backend формує токен доступу, який містить ідентифікатор користувача, роль, email, ім’я та час завершення дії.

```ts
export function createAccessToken(user: {
  id: number;
  role: Role;
  email: string;
  name: string;
}) {
  const header = toBase64Url(JSON.stringify({ alg: "HS256", typ: "JWT" }));
  const payload = toBase64Url(
    JSON.stringify({
      sub: user.id,
      role: user.role,
      email: user.email,
      name: user.name,
      exp: Math.floor(Date.now() / 1000) + env.authTokenTtlSeconds,
      type: "access",
    }),
  );

  const signature = signTokenParts(header, payload);
  return `${header}.${payload}.${signature}`;
}
```

У проєкті використано власну реалізацію підпису токена через HMAC SHA-256. Для навчальної лабораторної роботи це дозволяє явно показати принцип побудови та перевірки токена без прихованої логіки сторонніх auth-фреймворків.

### 4.5 Реєстрація користувача
Маршрут `POST /api/auth/register` перевіряє коректність полів `name`, `email`, `password`, `passwordConfirmation`, після чого створює нового користувача в базі та повертає токен разом з даними профілю.

```ts
app.post("/auth/register", async (request, reply) => {
  const body = request.body as {
    name?: unknown;
    email?: unknown;
    password?: unknown;
    passwordConfirmation?: unknown;
  };

  if (!isNonEmptyString(body?.name)) {
    return reply.status(400).send({ message: "name is required" });
  }

  if (!isEmail(body?.email)) {
    return reply.status(400).send({ message: "valid email is required" });
  }

  if (!isStrongPassword(body?.password)) {
    return reply.status(400).send({ message: "password must be at least 8 characters" });
  }
});
```

Окремо обробляється ситуація з дублюванням email через Prisma error handler у `app.ts`, який повертає статус `409` для конфлікту унікального значення.

### 4.6 Вхід у систему
Після входу за маршрутом `POST /api/auth/login` виконується пошук користувача за email і перевірка пароля. У разі успіху API повертає токен і короткі відомості про користувача, а у випадку помилки — статус `401`.

```ts
app.post("/auth/login", async (request, reply) => {
  const user = await prisma.user.findUnique({
    where: { email: body.email.trim().toLowerCase() },
  });

  if (!user || !(await verifyPassword(body.password, user.passwordHash))) {
    return reply.status(401).send({ message: "invalid email or password" });
  }
});
```

### 4.7 Перевірка токена на захищених маршрутах
Для захищених запитів використовується `requireAuth`. Функція читає Bearer token із заголовка, перевіряє підпис, строк дії та заповнює `request.authUser`.

```ts
export async function requireAuth(request: FastifyRequest, reply: FastifyReply) {
  const token = getBearerToken(request);

  if (!token) {
    return reply.status(401).send({ message: "authentication required" });
  }

  const payload = verifyAccessToken(token);

  if (!payload) {
    return reply.status(401).send({ message: "invalid or expired token" });
  }

  request.authUser = {
    id: payload.sub,
    role: payload.role,
    email: payload.email,
    name: payload.name,
  };
}
```

### 4.8 Робота з профілем і паролем
Після автентифікації користувач може:
- отримати власні дані через `GET /api/auth/me`;
- оновити ім’я та email через `PATCH /api/auth/profile`;
- змінити пароль через `POST /api/auth/change-password`;
- завершити сесію через `POST /api/auth/logout`.

З технічного погляду `logout` у цій реалізації не зберігає стан сесії на сервері, а лише завершує клієнтське використання токена. Для базового лабораторного сценарію цього достатньо, оскільки основний акцент зроблено на механізмі видачі, передавання і перевірки access token.

---

## 5. Реалізація frontend-частини

### 5.1 Сторінки автентифікації
У web-частині додано окремі маршрути:
- `/login`
- `/register`
- `/profile`

Сторінки `login` і `register` використовують спільний компонент форми `AuthForms`, який працює в двох режимах.

```tsx
export function AuthForms({ mode }: { mode: "login" | "register" }) {
  async function onSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();

    const session =
      mode === "login"
        ? await loginUser({
            email: form.email,
            password: form.password,
          })
        : await registerUser({
            name: form.name,
            email: form.email,
            password: form.password,
            passwordConfirmation: form.passwordConfirmation,
          });

    storeSession(session);
    router.push("/profile");
  }
}
```

### 5.2 Збереження сесії на клієнті
Після успішного входу або реєстрації клієнт зберігає сесію в `localStorage`, а також дублює ключові дані в cookie для middleware-перевірки.

```ts
export function storeSession(session: AuthSession) {
  window.localStorage.setItem(AUTH_STORAGE_KEY, JSON.stringify(session));
  setCookie(AUTH_COOKIE_NAME, session.accessToken, 60 * 60 * 8);
  setCookie(AUTH_ROLE_COOKIE_NAME, session.user.role, 60 * 60 * 8);
  emitAuthChange();
}
```

Такий підхід дозволяє:
- використовувати токен у запитах до API;
- швидко визначати факт авторизації на клієнті;
- виконувати редиректи ще до повного завантаження захищеної сторінки.

### 5.3 Захист сторінок через middleware
На рівні Next.js застосовано middleware, яке не допускає неавторизованого користувача до сторінки профілю та перенаправляє авторизованого користувача зі сторінок входу/реєстрації назад у профіль.

```ts
export function middleware(request: NextRequest) {
  const token = request.cookies.get(AUTH_COOKIE_NAME)?.value;
  const { pathname } = request.nextUrl;

  if (!token && pathname.startsWith("/profile")) {
    const loginUrl = new URL("/login", request.url);
    return NextResponse.redirect(loginUrl);
  }

  if (token && (pathname.startsWith("/login") || pathname.startsWith("/register"))) {
    const profileUrl = new URL("/profile", request.url);
    return NextResponse.redirect(profileUrl);
  }
}
```

### 5.4 Сторінка профілю
Сторінка `/profile` реалізована як захищений інтерфейс, у якому:
- відображаються основні дані користувача;
- доступна форма оновлення профілю;
- доступна форма зміни пароля;
- є кнопка виходу з системи.

Frontend під час завантаження сторінки додатково викликає `GET /api/auth/me`, щоб переконатися, що токен дійсний. Якщо токен відсутній або недійсний, локальна сесія очищається, а користувач повертається на `/login`.

### 5.5 API-функції клієнта
Для взаємодії з backend винесено окремі функції:
- `registerUser`
- `loginUser`
- `getCurrentUser`
- `updateProfile`
- `changePassword`
- `logoutUser`

Це робить код frontend-частини більш структурованим і дозволяє централізовано працювати з HTTP-запитами та обробкою помилок.

---

## 6. Перевірка роботи функціоналу

### 6.1 Типові сценарії перевірки
Функціонал Lab 3 перевіряється як через frontend, так і через API-запити:
- реєстрація нового користувача;
- спроба реєстрації з помилкою підтвердження пароля;
- успішний вхід;
- помилка входу з неправильним паролем;
- запит до захищеного маршруту без токена;
- запит до захищеного маршруту з валідним токеном;
- оновлення профілю;
- зміна пароля;
- вихід і повторна перевірка захищеного маршруту.

### 6.2 Очікувана поведінка
Успішні сценарії повинні:
- повертати коректний HTTP-статус (`200`, `201` або `204`);
- повертати об’єкт користувача там, де це потрібно;
- видавати токен після реєстрації та входу;
- перенаправляти користувача на сторінку профілю після успішної автентифікації.

Негативні сценарії повинні:
- повертати `400` у разі помилки валідації, зокрема коли `passwordConfirmation` не збігається з `password`;
- повертати `401` у разі відсутності токена або неправильних облікових даних.

---

## 7. Команди для запуску

### 7.1 Запуск інфраструктури бази даних
```bash
pnpm db:up
pnpm db:check
```

### 7.2 Підготовка Prisma
```bash
pnpm prisma:generate
pnpm prisma:push
pnpm prisma:seed
```

### 7.3 Запуск backend і frontend
```bash
pnpm dev:api
pnpm dev:web
```

Після цього frontend зазвичай доступний на `http://localhost:3000`, а backend API — на `http://localhost:3001`.

---

## 8. Скріншоти результатів

![Сторінка входу](/assets/labs/lab-3/login_page.png)  
**Рис. 1 – Сторінка входу `/login`.**

![Сторінка реєстрації](/assets/labs/lab-3/register_page.png)  
**Рис. 2 – Сторінка реєстрації `/register`.**

![Успішна авторизація та профіль](/assets/labs/lab-3/profile_page.png)  
**Рис. 3 – Захищена сторінка профілю `/profile` після входу.**

![Оновлення профілю](/assets/labs/lab-3/profile_update_success.png)  
**Рис. 4 – Приклад успішного оновлення профілю користувача.**

![Зміна пароля](/assets/labs/lab-3/change_password_success.png)  
**Рис. 5 – Приклад успішної зміни пароля.**

![Register success у Postman](/assets/labs/lab-3/postman_register_success.png)  
**Рис. 6 – Успішна реєстрація через `POST /api/auth/register`.**

![Register validation error у Postman](/assets/labs/lab-3/postman_register_validation_error.png)  
**Рис. 7 – Помилка валідації під час `POST /api/auth/register`, коли підтвердження пароля не збігається.**

![Login success у Postman](/assets/labs/lab-3/postman_login_success.png)  
**Рис. 8 – Успішний вхід через `POST /api/auth/login`.**

![Login invalid credentials у Postman](/assets/labs/lab-3/postman_login_invalid_credentials.png)  
**Рис. 9 – Помилка входу з неправильними обліковими даними.**

![Protected route without token](/assets/labs/lab-3/postman_me_unauthorized.png)  
**Рис. 10 – Відмова доступу до `GET /api/auth/me` без токена.**

![Protected route with token](/assets/labs/lab-3/postman_me_success.png)  
**Рис. 11 – Успішний доступ до `GET /api/auth/me` з Bearer token.**

---

## 9. Висновки
У межах лабораторної роботи реалізовано базовий модуль автентифікації та авторизації для веб-застосунку Helpdesk / Ticket System. На backend-частині додано маршрути для реєстрації, входу, отримання даних поточного користувача, редагування профілю, зміни пароля та виходу із системи. Паролі зберігаються у вигляді хешу, а для доступу до захищених маршрутів використовується підписаний токен доступу.

На frontend-частині реалізовано сторінки входу, реєстрації та профілю, збереження сесії в `localStorage` і cookie, а також middleware-перевірку захищених маршрутів. Отриманий результат формує основу для подальшого розширення системи: повноцінного role-based access control, керування сесіями, refresh token-механізму та детальнішого розмежування прав доступу між користувачем, агентом підтримки й адміністратором.

---

## 10. Перелік використаних джерел
1. Документація Fastify.  
2. Документація Prisma ORM.  
3. Документація Next.js (App Router, Middleware).  
4. Документація Node.js (`crypto`, `scrypt`, HMAC).  
5. Документація PostgreSQL.
