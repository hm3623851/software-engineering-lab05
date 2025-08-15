# پروژه Notes - استقرار با Docker

## توضیح Dockerfile

Dockerfile این پروژه شامل مراحل زیر است:

### لایه پایه
```dockerfile
FROM python:3.11-slim as base
```
از ایمیج Python 3.11 slim به عنوان پایه استفاده می‌شود که حجم کمتری دارد.

### متغیرهای محیطی
```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app \
    DJANGO_SETTINGS_MODULE=notes.settings
```
- `PYTHONDONTWRITEBYTECODE=1`: از ایجاد فایل‌های .pyc جلوگیری می‌کند
- `PYTHONUNBUFFERED=1`: خروجی Python را بدون بافر نمایش می‌دهد
- `PYTHONPATH=/app`: مسیر پایتون را تنظیم می‌کند
- `DJANGO_SETTINGS_MODULE=notes.settings`: ماژول تنظیمات Django را مشخص می‌کند

### نصب وابستگی‌های سیستم
```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc \
        g++ \
        libpq-dev \
        default-libmysqlclient-dev \
        pkg-config \
        curl \
    && rm -rf /var/lib/apt/lists/*
```
کامپایلرها و کتابخانه‌های مورد نیاز برای PostgreSQL نصب می‌شوند.

### نصب وابستگی‌های پایتون
```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir -r requirements.txt
```
فایل requirements.txt کپی شده و وابستگی‌های پایتون نصب می‌شوند.

### کپی فایل‌های پروژه
```dockerfile
COPY . .
```
تمام فایل‌های پروژه به کانتینر کپی می‌شوند.

### جمع‌آوری فایل‌های استاتیک
```dockerfile
RUN python manage.py collectstatic --noinput
```
فایل‌های استاتیک Django جمع‌آوری می‌شوند.

### ایجاد کاربر غیر root
```dockerfile
RUN adduser --disabled-password --gecos '' appuser \
    && chown -R appuser:appuser /app
USER appuser
```
برای امنیت، کاربر غیر root ایجاد شده و مالکیت فایل‌ها به آن داده می‌شود.

### پورت و Health Check
```dockerfile
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/ || exit 1
```
پورت 8000 در معرض دید قرار گرفته و بررسی سلامت هر 30 ثانیه انجام می‌شود.

### دستور اجرا
```dockerfile
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```
سرور Django روی آدرس 0.0.0.0:8000 اجرا می‌شود.

## توضیح Docker Compose

فایل `docker-compose.yml` شامل دو سرویس اصلی است:

### سرویس پایگاه داده (db)
```yaml
db:
  image: postgres:15-alpine
  volumes:
    - postgres_data:/var/lib/postgresql/data
  environment:
    - POSTGRES_DB=notes_db
    - POSTGRES_USER=notes_user
    - POSTGRES_PASSWORD=notes_password
  ports:
    - "5432:5432"
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U notes_user -d notes_db"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**وظایف:**
- از ایمیج PostgreSQL 15 Alpine استفاده می‌کند (حجم کمتر)
- داده‌ها در volume `postgres_data` ذخیره می‌شوند
- متغیرهای محیطی برای نام دیتابیس، کاربر و رمز عبور تنظیم شده‌اند
- پورت 5432 برای اتصال خارجی در دسترس است
- Health check هر 10 ثانیه وضعیت دیتابیس را بررسی می‌کند

### سرویس وب (web)
```yaml
web:
  build: .
  command: python manage.py runserver 0.0.0.0:8000
  volumes:
    - .:/app
  ports:
    - "8000:8000"
  environment:
    - DATABASE_URL=postgresql://notes_user:notes_password@db:5432/notes_db
    - DEBUG=True
    - DJANGO_SETTINGS_MODULE=notes.settings
  depends_on:
    db:
      condition: service_healthy
  restart: unless-stopped
```

**وظایف:**
- از Dockerfile موجود در دایرکتوری فعلی build می‌شود
- سرور Django روی پورت 8000 اجرا می‌شود
- فایل‌های پروژه به صورت volume mount شده‌اند (برای توسعه)
- متغیرهای محیطی برای اتصال به دیتابیس تنظیم شده‌اند
- وابستگی به سرویس db با شرط healthy بودن
- در صورت توقف، مجدداً راه‌اندازی می‌شود

### Volume
```yaml
volumes:
  postgres_data:
```
Volume برای ذخیره دائمی داده‌های PostgreSQL تعریف شده است.

## دستورات cURL برای تست API

### مرحله 1: ایجاد کاربر user1 با رمز 1234
```bash
curl -X POST http://localhost:8000/users/create/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=user1&password=1234"
```

### مرحله 2: ورود کاربر user1
```bash
curl -X POST http://localhost:8000/users/login/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=user1&password=1234" \
  -c cookies.txt
```

### مرحله 3: ایجاد یادداشت با عنوان title1 و محتوای body1
```bash
curl -X POST http://localhost:8000/notes/create/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "title=title1&body=body1" \
  -b cookies.txt
```

### مرحله 4: ایجاد یادداشت با عنوان title2 و محتوای body2
```bash
curl -X POST http://localhost:8000/notes/create/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "title=title2&body=body2" \
  -b cookies.txt
```

### مرحله 5: دریافت تمام یادداشت‌های user1
```bash
curl -X GET http://localhost:8000/notes/ \
  -b cookies.txt
```

## پاسخ به پرسش‌ها

### 1. وظایف Dockerfile، image و container را توضیح دهید.

**Dockerfile:**
- فایل متنی شامل دستورالعمل‌های ساخت ایمیج Docker است
- شامل مراحل نصب وابستگی‌ها، کپی فایل‌ها و تنظیم محیط اجرا است
- به صورت لایه‌ای (layered) ساخته می‌شود که بهینه‌سازی کش را امکان‌پذیر می‌کند

**Image:**
- نسخه قابل حمل و مستقل از نرم‌افزار است
- شامل تمام فایل‌ها، وابستگی‌ها و تنظیمات مورد نیاز است
- از Dockerfile ساخته می‌شود و قابل اشتراک‌گذاری است
- تغییرناپذیر (immutable) است و برای هر تغییر باید image جدید ساخته شود

**Container:**
- نمونه در حال اجرا از image است
- محیط اجرای مجزا و ایزوله فراهم می‌کند
- شامل سیستم فایل، شبکه و پردازش‌های مجزا است
- قابل شروع، توقف، حذف و مدیریت است

### 2. از kubernetes برای انجام چه کارهایی می‌توان استفاده کرد؟ رابطه آن با داکر چیست؟

**کاربردهای Kubernetes:**
- **Orchestration**: مدیریت و هماهنگی کانتینرها در مقیاس بزرگ
- **Auto-scaling**: افزایش یا کاهش خودکار تعداد کانتینرها بر اساس بار
- **Load balancing**: توزیع بار بین کانتینرهای مختلف
- **Service discovery**: کشف و ارتباط بین سرویس‌های مختلف
- **Rolling updates**: به‌روزرسانی بدون توقف سرویس
- **Health monitoring**: نظارت بر سلامت کانتینرها و restart خودکار
- **Resource management**: مدیریت منابع CPU و RAM

**رابطه با Docker:**
- **Docker** برای ساخت و اجرای کانتینرهای منفرد استفاده می‌شود
- **Kubernetes** برای مدیریت و orchestration کانتینرهای Docker در مقیاس بزرگ استفاده می‌شود
- Docker کانتینرها را می‌سازد و Kubernetes آنها را مدیریت می‌کند
- Kubernetes از Docker runtime برای اجرای کانتینرها استفاده می‌کند
- Docker برای توسعه و تست مناسب است، Kubernetes برای production و مقیاس بزرگ