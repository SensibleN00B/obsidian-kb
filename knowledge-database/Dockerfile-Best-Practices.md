# 🐋 Повний гайд з Best Practices для Dockerfile

> Комплексний посібник з найкращих практик створення Dockerfile на основі офіційної документації Docker

## 📑 Зміст

1. [Багатоетапні збірки (Multi-stage Builds)](#багатоетапні-збірки-multi-stage-builds)
2. [Оптимізація кешування шарів](#оптимізація-кешування-шарів)
3. [Безпека Docker-образів](#безпека-docker-образів)
4. [Вибір базових образів](#вибір-базових-образів)
5. [Робота з інструкціями COPY та ADD](#робота-з-інструкціями-copy-та-add)
6. [CMD vs ENTRYPOINT](#cmd-vs-entrypoint)
7. [ARG vs ENV](#arg-vs-env)
8. [Файл .dockerignore](#файл-dockerignore)
9. [Практичні приклади](#практичні-приклади)

---

## Багатоетапні збірки (Multi-stage Builds)

### 🎯 Навіщо потрібні?

Багатоетапні збірки дозволяють:
- **Зменшити розмір фінального образу** - виключити інструменти збірки
- **Покращити безпеку** - зменшити поверхню атаки
- **Оптимізувати процес розробки** - різні етапи для dev/test/production

### 📝 Базовий приклад для Node.js

```dockerfile
# syntax=docker/dockerfile:1

# ========================================
# Етап 1: Збірка додатку
# ========================================
FROM node:lts AS build
WORKDIR /app

# Копіюємо файли залежностей
COPY package.json yarn.lock ./
RUN yarn install

# Копіюємо вихідний код та збираємо
COPY public ./public
COPY src ./src
RUN yarn run build

# ========================================
# Етап 2: Production образ
# ========================================
FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
```

### 🔧 Розширений приклад з окремими етапами

```dockerfile
# syntax=docker/dockerfile:1

ARG NODE_VERSION=20-alpine

# ========================================
# Етап: Базові залежності
# ========================================
FROM node:${NODE_VERSION} AS base
WORKDIR /app

# Встановлюємо системні залежності
RUN apk add --no-cache \
    dumb-init \
    && rm -rf /var/cache/apk/*

# ========================================
# Етап: Production залежності
# ========================================
FROM base AS deps
COPY package.json package-lock.json ./
RUN npm ci --only=production --ignore-scripts

# ========================================
# Етап: Збірка
# ========================================
FROM base AS build
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts

COPY . .
RUN npm run build

# ========================================
# Етап: Development
# ========================================
FROM base AS development
ENV NODE_ENV=development

COPY --from=deps /app/node_modules ./node_modules
COPY . .

USER node
EXPOSE 3000
CMD ["npm", "run", "dev"]

# ========================================
# Етап: Production
# ========================================
FROM base AS production
ENV NODE_ENV=production \
    NODE_OPTIONS="--max-old-space-size=256"

# Створюємо непривілейованого користувача
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

# Копіюємо тільки необхідне
COPY --from=deps --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist
COPY --chown=nodejs:nodejs package.json ./

USER nodejs
EXPOSE 3000
CMD ["dumb-init", "node", "dist/index.js"]
```

---

## Оптимізація кешування шарів

### 🎯 Принцип роботи кешу

Docker кешує кожен шар (layer). Якщо файл змінився, всі наступні шари перебудовуються.

### ⚠️ Погана практика

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .                    # ❌ Копіюємо все відразу
RUN npm install            # ❌ Перевстановлюється кожного разу
RUN npm run build
```

### ✅ Хороша практика

```dockerfile
FROM node:20-alpine
WORKDIR /app

# 1. Спочатку копіюємо файли залежностей
COPY package.json package-lock.json ./
RUN npm ci --only=production

# 2. Потім копіюємо код (змінюється частіше)
COPY . .
RUN npm run build
```

### 📊 Правильний порядок шарів

```dockerfile
# 1️⃣ Базовий образ та системні залежності (змінюються рідко)
FROM node:20-alpine
RUN apk add --no-cache dumb-init

# 2️⃣ Робочий каталог
WORKDIR /app

# 3️⃣ Залежності проєкту (змінюються рідко)
COPY package*.json ./
RUN npm ci --only=production

# 4️⃣ Вихідний код (змінюється часто)
COPY . .

# 5️⃣ Збірка та конфігурація
RUN npm run build
CMD ["node", "dist/index.js"]
```

### 💡 Поради щодо кешування

```dockerfile
# ✅ Об'єднуйте команди RUN для зменшення шарів
RUN apt-get update && \
    apt-get install -y nginx redis-server && \
    rm -rf /var/lib/apt/lists/*

# ❌ Не створюйте зайві шари
RUN apt-get update
RUN apt-get install -y nginx
RUN apt-get install -y redis-server
RUN rm -rf /var/lib/apt/lists/*
```

---

## Безпека Docker-образів

### 🔒 Основні принципи безпеки

#### 1. Завжди використовуйте непривілейованого користувача

> [!CAUTION]
> Запуск контейнера як root - це серйозна загроза безпеці!

```dockerfile
# ✅ Створення непривілейованого користувача (Alpine)
FROM node:20-alpine
WORKDIR /app

RUN addgroup -g 1001 -S appuser && \
    adduser -S appuser -u 1001 -G appuser

COPY --chown=appuser:appuser . .

USER appuser
CMD ["node", "index.js"]
```

```dockerfile
# ✅ Створення непривілейованого користувача (Debian/Ubuntu)
FROM python:3.11-slim
WORKDIR /app

RUN groupadd -r appuser && \
    useradd --no-log-init -r -g appuser appuser

COPY --chown=appuser:appuser . .

USER appuser
CMD ["python", "app.py"]
```

#### 2. Ніколи не зберігайте секрети в ARG або ENV

```dockerfile
# ❌ НЕБЕЗПЕЧНО - секрети потрапляють в image
FROM ubuntu
ARG AWS_SECRET_ACCESS_KEY=super-secret-key
ENV DATABASE_PASSWORD=my-password
```

```dockerfile
# ✅ Використовуйте Docker secrets або build secrets
FROM ubuntu
RUN --mount=type=secret,id=aws_key \
    aws configure set aws_secret_access_key $(cat /run/secrets/aws_key)
```

#### 3. Мінімізуйте поверхню атаки

```dockerfile
# ✅ Використовуйте distroless або alpine образи
FROM gcr.io/distroless/nodejs20-debian12
COPY --from=build /app/dist /app
USER nonroot
WORKDIR /app
CMD ["/app/index.js"]
```

#### 4. Регулярно скануйте образи на вразливості

```bash
# Використовуйте Docker Scout
docker scout cves my-image:latest

# Або Trivy
trivy image my-image:latest
```

#### 5. Використовуйте конкретні версії базових образів

```dockerfile
# ❌ Погано - непередбачувані оновлення
FROM node:latest

# ✅ Добре - фіксована версія
FROM node:20.11.0-alpine3.19

# ✅ Ще краще - з хешем
FROM node:20.11.0-alpine3.19@sha256:abc123...
```

---

## Вибір базових образів

### 📦 Типи базових образів

#### 1. **Alpine** - найменший розмір

```dockerfile
FROM node:20-alpine  # ~40MB
FROM python:3.11-alpine  # ~50MB
```

**Переваги:**
- ✅ Мінімальний розмір
- ✅ Швидке завантаження
- ✅ Менша поверхня атаки

**Недоліки:**
- ❌ Використовує musl замість glibc (можливі проблеми сумісності)
- ❌ Менше pre-installed пакетів

#### 2. **Slim** - баланс розміру та функціональності

```dockerfile
FROM python:3.11-slim  # ~120MB
FROM node:20-slim  # ~180MB
```

**Переваги:**
- ✅ Базується на Debian
- ✅ Краща сумісність
- ✅ Менший за повний образ

#### 3. **Distroless** - максимальна безпека

```dockerfile
FROM gcr.io/distroless/nodejs20-debian12
FROM gcr.io/distroless/python3-debian12
```

**Переваги:**
- ✅ Немає shell (неможливо виконати команди в контейнері)
- ✅ Мінімальна поверхня атаки
- ✅ Тільки runtime залежності

**Недоліки:**
- ❌ Складніше налагоджувати
- ❌ Не можна встановлювати пакети

#### 4. **Повний образ** - для складних випадків

```dockerfile
FROM node:20  # ~900MB
FROM python:3.11  # ~800MB
```

### 🎯 Рекомендації щодо вибору

| Випадок використання | Рекомендація |
|---------------------|--------------|
| Production (Node.js/Python) | `alpine` або `slim` |
| Максимальна безпека | `distroless` |
| Складні нативні залежності | `slim` або повний образ |
| Development | Повний образ |
| Мікросервіси | `alpine` або `distroless` |

---

## Робота з інструкціями COPY та ADD

### 📋 COPY vs ADD - коли що використовувати?

> [!IMPORTANT]
> За замовчуванням завжди використовуйте `COPY`, якщо вам не потрібні додаткові можливості `ADD`

### COPY - рекомендований вибір

```dockerfile
# ✅ Простий та зрозумілий
COPY package.json /app/
COPY src/ /app/src/

# ✅ З встановленням прав власності
COPY --chown=appuser:appuser . /app

# ✅ З конкретними файлами
COPY package.json package-lock.json ./
```

### ADD - використовуйте обережно

```dockerfile
# ✅ Автоматичне розпакування tar
ADD myarchive.tar.gz /app/

# ✅ Завантаження з URL (не рекомендується)
ADD https://example.com/file.txt /app/

# ❌ Краще використовувати COPY для звичайних файлів
ADD file.txt /app/  # Замість цього: COPY file.txt /app/
```

### 💡 Best Practices

```dockerfile
# ✅ Використовуйте конкретні шляхи
COPY package.json package-lock.json ./

# ❌ Не копіюйте все підряд
COPY . .  # Тільки якщо .dockerignore правильно налаштовано

# ✅ Копіюйте в правильному порядку для кешування
COPY package.json ./
RUN npm install
COPY src/ ./src/

# ✅ Використовуйте --chown для встановлення прав
COPY --chown=node:node . /app
```

---

## CMD vs ENTRYPOINT

### 🎯 Різниця та правила використання

#### CMD - команда за замовчуванням

```dockerfile
# Форма shell (запускається через /bin/sh -c)
CMD npm start

# Exec форма (рекомендована)
CMD ["npm", "start"]

# Можна перевизначити при запуску
docker run my-image python app.py  # CMD ігнорується
```

#### ENTRYPOINT - точка входу

```dockerfile
# Exec форма (рекомендована)
ENTRYPOINT ["python", "app.py"]

# Використовується завжди, навіть з аргументами
docker run my-image --debug  # Виконає: python app.py --debug
```

### 📊 Комбінування ENTRYPOINT та CMD

```dockerfile
# ✅ Найкраща практика - ENTRYPOINT + CMD
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]

# Запуск за замовчуванням: python app.py --port 8000
docker run my-image

# З custom аргументами: python app.py --port 9000 --debug
docker run my-image --port 9000 --debug
```

### 🎯 Практичні приклади

#### Приклад 1: Node.js application

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .

# ENTRYPOINT - executable
ENTRYPOINT ["node"]

# CMD - default arguments
CMD ["index.js"]

# Використання:
# docker run my-app               -> node index.js
# docker run my-app server.js     -> node server.js
```

#### Приклад 2: Python з dumb-init

```dockerfile
FROM python:3.11-slim
WORKDIR /app

RUN apt-get update && \
    apt-get install -y dumb-init && \
    rm -rf /var/lib/apt/lists/*

COPY . .

ENTRYPOINT ["dumb-init", "--"]
CMD ["python", "app.py"]

# dumb-init правильно обробляє сигнали
```

#### Приклад 3: Універсальний wrapper

```dockerfile
FROM ubuntu:22.04
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["default-command"]
```

### 💡 Рекомендації

| Ситуація | Використовуйте |
|----------|----------------|
| Контейнер як executable | `ENTRYPOINT` |
| Аргументи за замовчуванням | `CMD` |
| Гнучкість для користувача | `CMD` |
| Фіксована команда | `ENTRYPOINT` |
| Комбінація обох | `ENTRYPOINT` + `CMD` |

---

## ARG vs ENV

### 🎯 Основна різниця

- **ARG** - доступні тільки під час збірки (build-time)
- **ENV** - доступні під час збірки та виконання (build-time + runtime)

### ARG - Build-time змінні

```dockerfile
# Оголошення ARG
ARG NODE_VERSION=20
ARG BUILD_DATE
ARG ENVIRONMENT=production

# Використання ARG
FROM node:${NODE_VERSION}-alpine

RUN echo "Build date: ${BUILD_DATE}"
RUN echo "Environment: ${ENVIRONMENT}"

# Передача значень при збірці
# docker build --build-arg NODE_VERSION=18 --build-arg BUILD_DATE=$(date) .
```

> [!WARNING]
> ARG значення зберігаються в image metadata. Не використовуйте ARG для секретів!

### ENV - Runtime змінні

```dockerfile
# Встановлення ENV
ENV NODE_ENV=production \
    PORT=3000 \
    LOG_LEVEL=info

# Використання ENV
RUN echo "Node env: ${NODE_ENV}"

# ENV доступні в контейнері при виконанні
ENTRYPOINT echo "Running on port: ${PORT}"
```

### 🔄 Комбінування ARG та ENV

```dockerfile
# Паттерн: ARG -> ENV
ARG NODE_VERSION=20
ARG APP_ENV=production

# Перетворюємо ARG в ENV для runtime
ENV NODE_VERSION=${NODE_VERSION} \
    APP_ENV=${APP_ENV}

FROM node:${NODE_VERSION}-alpine

# Тепер обидві змінні доступні під час виконання
```

### 📊 Практичні приклади

#### Приклад 1: Версії залежностей

```dockerfile
ARG NODE_VERSION=20.11.0
ARG ALPINE_VERSION=3.19

FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION}

ARG NPM_VERSION=10.2.4
RUN npm install -g npm@${NPM_VERSION}
```

#### Приклад 2: Multi-stage з ARG

```dockerfile
ARG BUILD_ENV=production

FROM node:20-alpine AS base
ARG BUILD_ENV
ENV NODE_ENV=${BUILD_ENV}

FROM base AS development
ENV NODE_ENV=development
RUN npm install

FROM base AS production
ENV NODE_ENV=production
RUN npm ci --only=production

# docker build --target production --build-arg BUILD_ENV=staging .
```

#### Приклад 3: Конфігурація застосунку

```dockerfile
# Build-time
ARG APP_VERSION=1.0.0

# Runtime
ENV PORT=8000 \
    HOST=0.0.0.0 \
    LOG_LEVEL=info \
    APP_VERSION=${APP_VERSION}

LABEL version="${APP_VERSION}"

EXPOSE ${PORT}
CMD ["python", "app.py"]
```

### 🔒 Безпека ARG/ENV

```dockerfile
# ❌ НЕБЕЗПЕЧНО - секрети в ARG
ARG DATABASE_PASSWORD=secret123
ARG API_KEY=abc-xyz-123

# ❌ НЕБЕЗПЕЧНО - секрети в ENV
ENV AWS_SECRET_KEY=my-secret-key

# ✅ БЕЗПЕЧНО - використовуйте build secrets
RUN --mount=type=secret,id=db_password \
    echo "Password: $(cat /run/secrets/db_password)" > /tmp/config

# Збірка з секретом:
# docker build --secret id=db_password,src=./secrets/password.txt .
```

### 💡 Коли що використовувати?

| Випадок | Використовуйте |
|---------|----------------|
| Версія базового образу | `ARG` |
| Налаштування build process | `ARG` |
| Налаштування runtime | `ENV` |
| Порт, хост, шляхи | `ENV` |
| Секрети | `--mount=type=secret` |
| Гнучкість для користувача | `ENV` (можна перевизначити при запуску) |

---

## Файл .dockerignore

### 🎯 Навіщо потрібен .dockerignore?

- ✅ Зменшує розмір build context
- ✅ Прискорює збірку
- ✅ Захищає секрети від потрапляння в образ
- ✅ Покращує кешування

### 📝 Базовий приклад .dockerignore

```dockerignore
# Git файли
.git
.gitignore
.gitattributes

# CI/CD
.github
.gitlab-ci.yml
.travis.yml

# Documentation
README.md
CHANGELOG.md
docs/
*.md

# Dependencies
node_modules/
venv/
__pycache__/
*.pyc
*.pyo
*.pyd

# Build artifacts
dist/
build/
*.egg-info/
target/

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db

# Environment files
.env
.env.local
.env.*.local

# Logs
logs/
*.log
npm-debug.log*

# Test files
tests/
test/
*.test.js
*.spec.js
coverage/

# Docker files (не потрібні inside container)
Dockerfile*
docker-compose*.yml
.dockerignore
```

### 🎯 Приклад для Node.js проєкту

```dockerignore
# Dependencies
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Testing
coverage/
.nyc_output/
*.test.js
*.spec.js
__tests__/

# Build
dist/
build/
.next/
.nuxt/
.cache/

# Environment
.env
.env.local
.env.*.local

# Editor
.vscode/
.idea/
*.sublime-*

# OS
.DS_Store
Thumbs.db

# Git
.git/
.gitignore

# Documentation
README.md
docs/
```

### 🎯 Приклад для Python проєкту

```dockerignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
venv/
env/
ENV/
.venv

# Testing
.pytest_cache/
.coverage
htmlcov/
.tox/
tests/

# Build
build/
dist/
*.egg-info/
wheels/

# Jupyter
.ipynb_checkpoints
*.ipynb

# Environment
.env
.env.local

# IDE
.vscode/
.idea/
*.swp

# Documentation
README.md
docs/
```

### 💡 Винятки з ігнорування

```dockerignore
# Ігноруємо всі markdown
*.md

# Але включаємо конкретний файл
!README.md

# Ігноруємо node_modules
node_modules/

# Але включаємо конкретний модуль (якщо потрібно)
!node_modules/specific-package/
```

### 🎯 Оптимізація для різних етапів

```dockerfile
# Dockerfile
FROM node:20-alpine AS development
# Development потребує тестів та документації
COPY . .

FROM node:20-alpine AS production
# Production не потребує dev-файлів
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/
```

```dockerignore
# .dockerignore для production
tests/
*.test.js
*.spec.js
docs/
.git/
node_modules/
```

---

## Практичні приклади

### 🚀 Full-Stack Node.js додаток

```dockerfile
# syntax=docker/dockerfile:1

# ========================================
# Аргументи для налаштування версій
# ========================================
ARG NODE_VERSION=20.11.0
ARG ALPINE_VERSION=3.19

# ========================================
# Базовий етап
# ========================================
FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS base

# Встановлюємо системні залежності
RUN apk add --no-cache \
    dumb-init \
    curl \
    && rm -rf /var/cache/apk/*

WORKDIR /app

# Створюємо користувача
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs && \
    chown -R nodejs:nodejs /app

# ========================================
# Етап: Dependencies
# ========================================
FROM base AS dependencies

# Копіюємо package files
COPY --chown=nodejs:nodejs package.json package-lock.json ./

# Встановлюємо production dependencies
RUN npm ci --only=production --ignore-scripts && \
    npm cache clean --force

# ========================================
# Етап: Build
# ========================================
FROM base AS build

# Копіюємо package files
COPY --chown=nodejs:nodejs package.json package-lock.json ./

# Встановлюємо всі dependencies (включно з devDependencies)
RUN npm ci --ignore-scripts

# Копіюємо вихідний код
COPY --chown=nodejs:nodejs . .

# Збираємо додаток
RUN npm run build && \
    chown -R nodejs:nodejs /app

# ========================================
# Етап: Development
# ========================================
FROM base AS development

ENV NODE_ENV=development

# Копіюємо node_modules з dependencies етапу
COPY --from=dependencies --chown=nodejs:nodejs /app/node_modules ./node_modules

# Копіюємо весь код
COPY --chown=nodejs:nodejs . .

USER nodejs

# Expose ports (app + HMR)
EXPOSE 3000 5173

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s  \
    CMD node healthcheck.js || exit 1

CMD ["dumb-init", "npm", "run", "dev"]

# ========================================
# Етап: Production
# ========================================
FROM base AS production

ENV NODE_ENV=production \
    NODE_OPTIONS="--max-old-space-size=512" \
    NPM_CONFIG_LOGLEVEL=error

# Копіюємо production dependencies
COPY --from=dependencies --chown=nodejs:nodejs /app/node_modules ./node_modules

# Копіюємо package.json
COPY --chown=nodejs:nodejs package.json ./

# Копіюємо зібраний додаток
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist

USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
    CMD curl -f http://localhost:3000/health || exit 1

CMD ["dumb-init", "node", "dist/index.js"]

# ========================================
# Етап: Test
# ========================================
FROM build AS test

ENV NODE_ENV=test \
    CI=true

USER nodejs

CMD ["npm", "run", "test:ci"]
```

### 🐍 Python FastAPI додаток

```dockerfile
# syntax=docker/dockerfile:1

# ========================================
# Аргументи
# ========================================
ARG PYTHON_VERSION=3.11
ARG DEBIAN_VERSION=bookworm

# ========================================
# Базовий етап
# ========================================
FROM python:${PYTHON_VERSION}-slim-${DEBIAN_VERSION} AS base

# Встановлюємо системні залежності
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        dumb-init \
        && rm -rf /var/lib/apt/lists/*

# Python optimizations
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# Створюємо користувача
RUN groupadd -r appuser && \
    useradd --no-log-init -r -g appuser appuser && \
    chown -R appuser:appuser /app

# ========================================
# Етап: Dependencies
# ========================================
FROM base AS dependencies

# Копіюємо requirements
COPY requirements.txt ./

# Встановлюємо dependencies
RUN pip install --no-cache-dir -r requirements.txt

# ========================================
# Етап: Development
# ========================================
FROM base AS development

ENV ENVIRONMENT=development

# Копіюємо dependencies з попереднього етапу
COPY --from=dependencies /usr/local/lib/python${PYTHON_VERSION}/site-packages /usr/local/lib/python${PYTHON_VERSION}/site-packages
COPY --from=dependencies /usr/local/bin /usr/local/bin

# Копіюємо код
COPY --chown=appuser:appuser . .

USER appuser

EXPOSE 8000

CMD ["dumb-init", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# ========================================
# Етап: Production
# ========================================
FROM base AS production

ENV ENVIRONMENT=production

# Копіюємо тільки production dependencies
COPY --from=dependencies /usr/local/lib/python${PYTHON_VERSION}/site-packages /usr/local/lib/python${PYTHON_VERSION}/site-packages
COPY --from=dependencies /usr/local/bin /usr/local/bin

# Копіюємо тільки application code (без tests)
COPY --chown=appuser:appuser app/ ./app/
COPY --chown=appuser:appuser alembic/ ./alembic/
COPY --chown=appuser:appuser alembic.ini ./

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["dumb-init", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]

# ========================================
# Етап: Test
# ========================================
FROM development AS test

ENV ENVIRONMENT=test

# Встановлюємо test dependencies
COPY requirements-test.txt ./
RUN pip install --no-cache-dir -r requirements-test.txt

CMD ["pytest", "-v", "--cov=app", "--cov-report=term-missing"]
```

### ☕ Java Spring Boot додаток

```dockerfile
# syntax=docker/dockerfile:1

# ========================================
# Аргументи
# ========================================
ARG JAVA_VERSION=21

# ========================================
# Етап: Build
# ========================================
FROM eclipse-temurin:${JAVA_VERSION}-jdk-jammy AS build

WORKDIR /app

# Копіюємо Maven wrapper та pom.xml
COPY .mvn/ .mvn
COPY mvnw pom.xml ./

# Завантажуємо dependencies (кешується окремо)
RUN ./mvnw dependency:go-offline

# Копіюємо вихідний код
COPY src ./src

# Збираємо додаток
RUN ./mvnw clean package -DskipTests && \
    java -Djarmode=layertools -jar target/*.jar extract

# ========================================
# Етап: Production
# ========================================
FROM eclipse-temurin:${JAVA_VERSION}-jre-jammy AS production

WORKDIR /app

# Створюємо користувача
RUN groupadd -r spring && \
    useradd -r -g spring spring && \
    chown -R spring:spring /app

# Копіюємо layers в правильному порядку
COPY --from=build --chown=spring:spring /app/dependencies/ ./
COPY --from=build --chown=spring:spring /app/spring-boot-loader/ ./
COPY --from=build --chown=spring:spring /app/snapshot-dependencies/ ./
COPY --from=build --chown=spring:spring /app/application/ ./

USER spring

EXPOSE 8080

ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"

HEALTHCHECK --interval=30s --timeout=3s --start-period=60s \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

---

## 📊 Чеклист Best Practices

### ✅ Обов'язкові практики

- [ ] Використовуйте багатоетапні збірки для production
- [ ] Запускайте контейнер від непривілейованого користувача
- [ ] Використовуйте конкретні версії базових образів (не `latest`)
- [ ] Створіть `.dockerignore` файл
- [ ] Оптимізуйте порядок інструкцій для кешування
- [ ] Використовуйте `COPY` замість `ADD` (якщо не потрібна розпаковка)
- [ ] Об'єднуйте `RUN` команди для зменшення шарів
- [ ] Додайте `HEALTHCHECK` для production контейнерів
- [ ] Використовуйте `dumb-init` або `tini` як init процес

### 🔒 Безпека

- [ ] Не зберігайте секрети в `ARG` або `ENV`
- [ ] Використовуйте офіційні базові образи
- [ ] Регулярно скануйте образи на вразливості
- [ ] Видаляйте непотрібні пакети після встановлення
- [ ] Використовуйте `alpine` або `distroless` для production
- [ ] Встановлюйте `USER` перед `CMD`/`ENTRYPOINT`

### ⚡ Оптимізація

- [ ] Копіюйте залежності перед кодом
- [ ] Використовуйте BuildKit для швидших збірок
- [ ] Очищайте кеш пакетних менеджерів
- [ ] Використовуйте `--no-install-recommends` для apt-get
- [ ] Мінімізуйте кількість шарів
- [ ] Використовуйте `.dockerignore` для виключення зайвих файлів

### 📝 Документація та підтримка

- [ ] Додайте `LABEL` з metadata
- [ ] Документуйте `ARG` змінні
- [ ] Використовуйте коментарі для складних секцій
- [ ] Вказуйте `EXPOSE` для documented портів
- [ ] Додайте приклади збірки в README

---

## 🛠️ Корисні команди

### Збірка з оптимізаціями

```bash
# Збірка з BuildKit
DOCKER_BUILDKIT=1 docker build -t myapp:latest .

# Збірка конкретного етапу
docker build --target production -t myapp:prod .

# Збірка з build arguments
docker build --build-arg NODE_VERSION=18 -t myapp:latest .

# Збірка з секретами
docker build --secret id=npm_token,src=.npmrc -t myapp:latest .
```

### Аналіз образу

```bash
# Переглянути історію шарів
docker history myapp:latest

# Перевірити розмір образу
docker images myapp:latest

# Аналіз з dive
dive myapp:latest

# Сканування вразливостей
docker scout cves myapp:latest
trivy image myapp:latest
```

### Запуск контейнерів

```bash
# Production
docker run -d \
  --name myapp \
  --restart unless-stopped \
  -p 3000:3000 \
  -e NODE_ENV=production \
  myapp:prod

# Development з volume mounting
docker run -it \
  --name myapp-dev \
  -p 3000:3000 \
  -v $(pwd):/app \
  -v /app/node_modules \
  myapp:dev

# З обмеженням ресурсів
docker run -d \
  --name myapp \
  --memory="512m" \
  --cpus="1.0" \
  myapp:prod
```

---

## 📚 Додаткові ресурси

### Офіційна документація

- [Docker Official Docs](https://docs.docker.com/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Build Cache](https://docs.docker.com/build/cache/)

### Інструменти

- [Docker Scout](https://docs.docker.com/scout/) - аналіз безпеки
- [Trivy](https://github.com/aquasecurity/trivy) - сканер вразливостей
- [Dive](https://github.com/wagoodman/dive) - аналіз шарів образу
- [Hadolint](https://github.com/hadolint/hadolint) - лінтер для Dockerfile

### Distroless образи

- [Google Distroless](https://github.com/GoogleContainerTools/distroless)
- [Chainguard Images](https://www.chainguard.dev/chainguard-images)

---

## 🎓 Висновок

Дотримання best practices для Dockerfile допомагає:

1. **Безпека** - мінімізація ризиків та вразливостей
2. **Продуктивність** - швидші збірки та менші образи
3. **Підтримка** - зрозуміліший та легший в підтримці код
4. **Ефективність** - оптимальне використання ресурсів

> [!TIP]
> Починайте з простих патернів та поступово додавайте оптимізації. Використовуйте цей гайд як checklist при створенні нових Dockerfile.

---

**Версія гайду:** 1.0  
**Останнє оновлення:** Січень 2026  
**Базується на:** Docker Official Documentation
