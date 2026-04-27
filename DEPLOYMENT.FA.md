# راهنمای Build، اجرا و Deploy

این راهنما مراحل مربوط به **پایگاه‌داده، بک‌اند، فرانت‌اند، گردش‌کار Docker و استقرار (Deployment)** را توضیح می‌دهد.

---

# پیش‌نیازها

- Node.js نسخه `24.12.0` (طبق فایل `.nvmrc`)
- npm
- Docker و Docker Compose (برای اجرای کانتینری در محیط محلی یا مشابه production)
- PostgreSQL (در صورتی که دیتابیس را خارج از Docker اجرا می‌کنید)

---

# بخش‌های پروژه

- **Frontend:** ریشه مخزن (Repository root)، برنامه Vite  
- **Backend:** پوشه `backend/`، شامل Node + Express + Prisma  
- **Database:** PostgreSQL که با متغیر `DATABASE_URL` تنظیم می‌شود  

---

# فایل‌های محیطی (Environment Files)

## Backend

فایل زیر را ایجاد کنید:

```
backend/.env
```

محتوا:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/me?schema=public"
PORT=4000
JWT_SECRET="replace_with_a_long_random_secret"
NODE_ENV=development
```

برای **محیط production** از دیتابیس واقعی و یک `JWT_SECRET` قوی استفاده کنید:

```env
DATABASE_URL="postgresql://USER:PASSWORD@HOST:5432/DB_NAME?schema=public"
PORT=4000
JWT_SECRET="very_long_random_secret"
NODE_ENV=production
```

---

## Frontend

فایل `.env.example` در ریشه پروژه شامل این مقدار است:

```env
VITE_API_URL=https://api.dabirgress.runflare.run
```

---

# راه‌اندازی دیتابیس بدون Docker

ابتدا PostgreSQL را به صورت محلی اجرا کنید و یک دیتابیس با نام `me` بسازید.

مثال با `psql`:

```bash
createdb me
```

سپس در فایل `backend/.env` رشته اتصال را تنظیم کنید:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/me?schema=public"
```

وابستگی‌ها را از ریشه پروژه نصب کنید:

```bash
npm install
```

اگر هنگام نصب **Cypress** خطا داشتید (به دلیل ناسازگاری نسخه Node)، از دستور زیر استفاده کنید یا ورژن نود خود را به ورژنی قبلتر از 20 تغییر دهید:

```bash
npm install --force
```

سپس Prisma client را ایجاد کنید:

```bash
npm run prisma:generate
```

برای ایجاد یا بروزرسانی جدول‌های دیتابیس از schema پریزما:

```bash
npm run prisma:db:push
```

در صورت نیاز می‌توانید داده اولیه (seed) اضافه کنید:

```bash
cd backend
npm run seed
```

---

# اجرای Backend بدون Docker

از ریشه پروژه:

```bash
npm install
npm run prisma:generate
npm run prisma:db:push
npm run dev:backend
```

یا از داخل پوشه `backend`:

```bash
cd backend
npm install
npm run prisma:generate
npm run prisma:db:push
npm run dev
```

اجرای به سبک production:

```bash
cd backend
NODE_ENV=production npm start
```

بک‌اند روی پورتی که در `PORT` تعریف شده اجرا می‌شود، معمولاً:

```
http://localhost:4000
```

---

# اجرای Frontend بدون Docker

از ریشه پروژه:

```bash
npm install
npm run dev
```

سرور توسعه Vite در این آدرس اجرا می‌شود:

```
http://127.0.0.1:5173
```

پروکسی توسعه در فایل `vite.config.js` مسیر `/api` را به این آدرس هدایت می‌کند:

```
http://127.0.0.1:4001
```

اگر از این proxy استفاده می‌کنید، مطمئن شوید **پورت بک‌اند با target پروکسی یکی باشد** یا آن را تغییر دهید.

---

# Build کردن Frontend

از ریشه پروژه:

```bash
npm install
npm run build
```

خروجی build در پوشه زیر ساخته می‌شود:

```
dist/
```

برای مشاهده نسخه production به صورت محلی:

```bash
npm run preview
```

سرور preview در آدرس زیر اجرا می‌شود:

```
http://localhost:5173
```

---

# اجرای Docker: دیتابیس + بک‌اند

یک تنظیم Docker برای **بک‌اند و PostgreSQL** در پروژه وجود دارد.

از ریشه پروژه اجرا کنید:

```bash
docker compose up --build
```

این دستور این سرویس‌ها را اجرا می‌کند:

- `db` → کانتینر PostgreSQL  
- `backend` → کانتینر Node backend  

کانتینر بک‌اند قبل از اجرا دیتابیس را تنظیم می‌کند:

```bash
npm run prisma:db:push && npm start
```

بک‌اند در این آدرس در دسترس خواهد بود:

```
http://localhost:4000
```

متوقف کردن کانتینرها:

```bash
docker compose down
```

متوقف کردن و حذف volume دیتابیس:

```bash
docker compose down -v
```

---

# ساخت Docker فقط برای Backend

از ریشه پروژه:

```bash
docker build -f backend/Dockerfile -t me-backend .
```

اجرای ایمیج به صورت دستی:

```bash
docker run --rm -p 4000:4000 \
  -e DATABASE_URL="postgresql://postgres:postgres@host.docker.internal:5432/me?schema=public" \
  -e JWT_SECRET="replace_with_a_long_random_secret" \
  -e PORT=4000 \
  me-backend
```

اجرای migration یا schema push در یک کانتینر موقت:

```bash
docker run --rm \
  -e DATABASE_URL="postgresql://postgres:postgres@host.docker.internal:5432/me?schema=public" \
  me-backend \
  npm run prisma:db:push
```

---

# چک‌لیست Deploy در Production

## Database

1. یک PostgreSQL مدیریت‌شده ایجاد کنید.  
2. URL اتصال دیتابیس را کپی کنید.  
3. متغیر `DATABASE_URL` را در محیط بک‌اند تنظیم کنید.  
4. schema دیتابیس را اجرا کنید:

```bash
npm run prisma:db:push
```

برای production بهتر است به جای `db push` از **Prisma migrations** استفاده شود.

---

## Backend

1. ایمیج Docker را بسازید:

```bash
docker build -f backend/Dockerfile -t me-backend .
```

2. آن را روی پلتفرم هاست یا container platform خود deploy کنید.  
3. متغیرهای محیطی را تنظیم کنید:

```env
DATABASE_URL="postgresql://USER:PASSWORD@HOST:5432/DB_NAME?schema=public"
PORT=4000
JWT_SECRET="very_long_random_secret"
NODE_ENV=production
```

4. بک‌اند را پشت HTTPS در دسترس قرار دهید.  
5. بررسی کنید که دامنه بک‌اند با مسیرهای `/api` درست کار می‌کند.

---

## Frontend

1. مطمئن شوید URLهای API فرانت‌اند به دامنه بک‌اند deploy شده اشاره می‌کنند.  
2. فرانت‌اند را build کنید:

```bash
npm run build
```

3. پوشه `dist/` را روی هاست استاتیک deploy کنید (مثل Netlify، Vercel، Cloudflare Pages، Nginx یا S3/CloudFront).  
4. برای SPA تنظیم کنید که در صورت routeهای کلاینت، فایل `index.html` سرو شود.

---

# دستورات رایج در محیط محلی

نصب همه وابستگی‌ها:

```bash
npm install
```

اجرای backend:

```bash
npm run dev:backend
```

اجرای frontend:

```bash
npm run dev
```

Build کردن frontend:

```bash
npm run build
```

ساخت Prisma client:

```bash
npm run prisma:generate
```

اعمال schema Prisma به دیتابیس:

```bash
npm run prisma:db:push
```

اجرای Docker برای backend + DB:

```bash
docker compose up --build
```

---

# رفع مشکلات (Troubleshooting)

## اتصال Backend به دیتابیس برقرار نمی‌شود

`DATABASE_URL`، میزبان دیتابیس، نام کاربری، رمز عبور، نام دیتابیس و اجرای PostgreSQL را بررسی کنید.

---

## خطای Prisma client

Prisma client را دوباره بسازید:

```bash
npm run prisma:generate
```

---

## جدول‌های دیتابیس وجود ندارند

schema پریزما را به دیتابیس push کنید:

```bash
npm run prisma:db:push
```

---