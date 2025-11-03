# 🏗️ AREA Project - Architecture & Defense Guide

## 📋 Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Backend Structure](#backend-structure)
4. [Frontend Structure](#frontend-structure)
5. [Database Schema](#database-schema)
6. [Workflows System](#workflows-system)
7. [OAuth Integration](#oauth-integration)
8. [Docker Setup](#docker-setup)
9. [Startup Process](#startup-process)
10. [Key Files & Locations](#key-files--locations)

---

## 1. Project Overview

**AREA** = **A**ction **RE**action **A**utomation

Un sistema tipo IFTTT/Zapier que permite crear automatizaciones entre diferentes servicios (GitHub, Discord, Spotify, Twitch, Google, Email).

### Tech Stack:
- **Backend:** Laravel 11 (PHP)
- **Frontend Web:** Next.js + React + TypeScript
- **Frontend Mobile:** React Native + Expo
- **Database:** SQLite
- **Containerization:** Docker + Docker Compose
- **Webhooks Relay:** Smee.io

---

## 2. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                  │
├─────────────────┬───────────────────────┬───────────────────────┤
│  Web (Next.js)  │  Mobile (React Native)│   Discord Bot         │
│  Port: 8081     │  Expo                 │   (optional)          │
└────────┬────────┴───────────┬───────────┴──────────┬────────────┘
         │                    │                      │
         └────────────────────┼──────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   API Gateway     │
                    │  Laravel Backend  │
                    │   Port: 8000      │
                    └─────────┬─────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    ┌────▼────┐         ┌─────▼─────┐      ┌──────▼──────┐
    │ SQLite  │         │  Smee.io  │      │ OAuth APIs  │
    │   DB    │         │  Webhooks │      │  Services   │
    └─────────┘         └───────────┘      └─────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    ┌────▼────┐         ┌─────▼─────┐      ┌──────▼──────┐
    │ GitHub  │         │  Discord  │      │   Spotify   │
    │Webhooks │         │ Webhooks  │      │   Twitch    │
    └─────────┘         └───────────┘      └─────────────┘
```

---

## 3. Backend Structure

### 📁 Directory Structure

```
apps/server/
├── app/
│   ├── Http/
│   │   ├── Controllers/          # 🎮 CONTROLADORES (Lógica de negocio)
│   │   │   ├── AuthController.php              # Login/Register básico
│   │   │   ├── DiscordAuthController.php       # OAuth Discord
│   │   │   ├── GitHubAuthController.php        # OAuth GitHub
│   │   │   ├── SpotifyAuthController.php       # OAuth Spotify
│   │   │   ├── TwitchAuthController.php        # OAuth Twitch
│   │   │   ├── GoogleAuthController.php        # OAuth Google
│   │   │   ├── WorkflowController.php          # ⚙️ CRUD de Workflows
│   │   │   ├── WorkflowWebhookController.php   # 📬 Recepción de Webhooks
│   │   │   ├── GitHubRepositoryController.php  # Lista repos de GitHub
│   │   │   └── IntegrationController.php       # Conexiones OAuth
│   │   └── Middleware/
│   │       └── Authenticate.php
│   ├── Models/                   # 🗄️ MODELOS (Base de datos)
│   │   ├── User.php              # Usuarios
│   │   ├── Workflow.php          # ⚙️ Workflows (Actions + Reactions)
│   │   ├── OAuthToken.php        # Tokens OAuth guardados
│   │   └── Integration.php       # (legacy)
│   └── Services/                 # Servicios auxiliares
│       ├── SpotifyService.php    # Polling de Spotify
│       └── TwitchService.php     # Polling de Twitch
│
├── routes/
│   ├── api.php                   # 🛣️ RUTAS DE LA API
│   └── web.php
│
├── database/
│   ├── migrations/               # 📊 ESQUEMA DE BD
│   │   ├── 2024_*_create_users_table.php
│   │   ├── 2024_*_create_oauth_tokens_table.php
│   │   └── 2024_*_create_workflows_table.php
│   └── database.sqlite           # Base de datos SQLite
│
├── config/
│   ├── services.php              # 🔑 Credenciales OAuth (Client ID/Secret)
│   └── cors.php                  # CORS para frontend
│
├── .env                          # 🔐 VARIABLES DE ENTORNO
├── docker-entrypoint.sh          # Script de inicio Docker
└── Dockerfile                    # Imagen Docker del backend
```

---

## 4. Frontend Structure

### 📁 Web Frontend (Next.js)

```
apps/web/
├── pages/
│   ├── index.tsx                 # 🏠 Landing page
│   ├── login.tsx                 # Login
│   ├── register.tsx              # Registro
│   ├── dashboard.tsx             # 📊 Dashboard principal
│   └── workflows.tsx             # Lista de workflows guardados
│
├── components/
│   └── AreaBuilder.tsx           # 🎨 EDITOR VISUAL de workflows
│                                 #    (drag & drop, conectar actions/reactions)
│
├── services/
│   └── api.ts                    # Llamadas HTTP al backend
│
└── package.json
```

---

## 5. Database Schema

### 📊 Tabla: `users`
```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE,
    password VARCHAR(255),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### 📊 Tabla: `oauth_tokens`
```sql
CREATE TABLE oauth_tokens (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,                    -- FK a users
    service VARCHAR(50),                -- 'discord', 'github', 'spotify', etc.
    access_token TEXT,
    refresh_token TEXT,
    expires_at TIMESTAMP,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**Ubicación:** `apps/server/database/migrations/2024_*_create_oauth_tokens_table.php`

### 📊 Tabla: `workflows` ⚙️ (MÁS IMPORTANTE)
```sql
CREATE TABLE workflows (
    id INTEGER PRIMARY KEY,
    user_identifier VARCHAR(255),       -- Email o ID del usuario
    name VARCHAR(255),                  -- Nombre del workflow
    
    -- ACTION (Trigger/Evento)
    action_type VARCHAR(100),           -- Tipo de acción
    action_config JSON,                 -- Configuración de la acción
    
    -- REACTION (Acción a ejecutar)
    reaction_type VARCHAR(100),         -- Tipo de reacción
    reaction_config JSON,               -- Configuración de la reacción
    
    is_active BOOLEAN DEFAULT true,     -- Activo/Pausado
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**Ubicación:** `apps/server/database/migrations/2024_*_create_workflows_table.php`

#### Ejemplo de Workflow en BD:
```json
{
  "id": 1,
  "user_identifier": "user@example.com",
  "name": "Nuevo Issue en GitHub → Discord",
  "action_type": "github_new_issue",
  "action_config": {
    "repository": "owner/repo"
  },
  "reaction_type": "send_discord_message",
  "reaction_config": {
    "guild_id": "123456789",
    "channel_id": "987654321",
    "message": "New issue created!"
  },
  "is_active": true
}
```

---

## 6. Workflows System (⚙️ CORE DEL PROYECTO)

### 🎯 Actions (Triggers) Disponibles

**Definidos en:** `apps/server/app/Models/Workflow.php`

```php
// GitHub Actions
const ACTION_GITHUB_NEW_ISSUE = 'github_new_issue';
const ACTION_GITHUB_NEW_PR = 'github_new_pr';
const ACTION_GITHUB_NEW_PUSH = 'github_new_push';
const ACTION_GITHUB_NEW_STAR = 'github_new_star';

// Discord Actions
const ACTION_DISCORD_MENTION = 'discord_mention';
const ACTION_DISCORD_KEYWORD = 'discord_keyword';
const ACTION_DISCORD_MEMBER_JOIN = 'discord_member_join';

// Spotify Actions
const ACTION_SPOTIFY_NEW_FOLLOWER = 'spotify_new_follower';
const ACTION_SPOTIFY_SAVED_TRACK = 'spotify_saved_track';

// Twitch Actions
const ACTION_TWITCH_NEW_FOLLOWER = 'twitch_new_follower';

// Timer Actions
const ACTION_TIMER_AT_TIME = 'timer_at_time';
const ACTION_TIMER_AT_DATE = 'timer_at_date';
const ACTION_TIMER_IN_X_DAYS = 'timer_in_x_days';
```

### ⚡ Reactions Disponibles

```php
const REACTION_SEND_EMAIL = 'send_email';
const REACTION_SEND_PUSH = 'send_push';
const REACTION_SEND_DISCORD_MESSAGE = 'send_discord_message';
```

### 📍 Dónde se definen las Actions/Reactions

**Backend - Endpoint que las devuelve:**
- **Ruta:** `/api/workflows/action-types`
- **Controlador:** `WorkflowController@getActionTypes()`
- **Archivo:** `apps/server/app/Http/Controllers/WorkflowController.php`

```php
public function getActionTypes()
{
    return response()->json([
        'success' => true,
        'action_types' => [
            [
                'value' => 'github_new_issue',
                'label' => 'Nuevo Issue en GitHub',
                'description' => 'Se activa cuando se crea un issue',
                'icon' => 'github',
                'config_fields' => [
                    [
                        'name' => 'repository',
                        'label' => 'Repositorio',
                        'type' => 'repository_select',
                        'required' => true,
                        'placeholder' => 'owner/repo'
                    ]
                ]
            ],
            // ... más actions
        ]
    ]);
}
```

**Frontend - Dónde se usan:**
- **Componente:** `AreaBuilder.tsx` (editor visual)
- **Ubicación:** `apps/web/components/AreaBuilder.tsx`
- **Líneas:** 79-93 (carga de action/reaction types)

---

## 7. OAuth Integration

### 🔐 Flujo de OAuth

```
1. Usuario hace clic en "Connect GitHub"
   ↓
2. Frontend redirige a: /api/auth/github/redirect
   ↓
3. Backend (GitHubAuthController) genera URL de autorización de GitHub
   ↓
4. Usuario autoriza en GitHub
   ↓
5. GitHub redirige a: /api/auth/github/callback?code=xxx
   ↓
6. Backend intercambia code por access_token
   ↓
7. Backend guarda token en tabla oauth_tokens
   ↓
8. Backend redirige a: /dashboard?connected=github
```

### 📍 Controladores OAuth

Cada servicio tiene su controlador:

| Servicio | Controlador | Ubicación |
|----------|-------------|-----------|
| GitHub | `GitHubAuthController` | `app/Http/Controllers/GitHubAuthController.php` |
| Discord | `DiscordAuthController` | `app/Http/Controllers/DiscordAuthController.php` |
| Spotify | `SpotifyAuthController` | `app/Http/Controllers/SpotifyAuthController.php` |
| Twitch | `TwitchAuthController` | `app/Http/Controllers/TwitchAuthController.php` |
| Google | `GoogleAuthController` | `app/Http/Controllers/GoogleAuthController.php` |

### 🔑 Credenciales OAuth

**Ubicación:** `apps/server/config/services.php`

```php
return [
    'github' => [
        'client_id' => env('GITHUB_CLIENT_ID'),
        'client_secret' => env('GITHUB_CLIENT_SECRET'),
        'redirect' => env('GITHUB_REDIRECT_URI'),
    ],
    'discord' => [
        'client_id' => env('DISCORD_CLIENT_ID'),
        'client_secret' => env('DISCORD_CLIENT_SECRET'),
        'redirect' => env('DISCORD_REDIRECT_URI'),
        'bot_token' => env('DISCORD_BOT_TOKEN'),
    ],
    // ... otros servicios
];
```

**Variables en `.env`:**
```env
GITHUB_CLIENT_ID=Ov23liSuMAy9WnEyiV4z
GITHUB_CLIENT_SECRET=...
DISCORD_CLIENT_ID=1422963310881968248
DISCORD_CLIENT_SECRET=...
DISCORD_BOT_TOKEN=...
```

---

## 8. Docker Setup

### 🐳 Arquitectura Docker

```
docker-compose.yml
├── backend (Laravel)
│   ├── Image: apps/server/Dockerfile
│   ├── Port: 8000
│   └── Volumes: ./apps/server → /var/www
│
├── frontend (Next.js)
│   ├── Image: apps/web/Dockerfile
│   ├── Port: 8081
│   └── Volumes: ./apps/web → /app
│
└── discord-bot (opcional)
    ├── Image: discord-bot/Dockerfile
    └── Depends on: backend
```

### 📄 `docker-compose.yml`

```yaml
version: '3.8'

services:
  backend:
    build:
      context: ./apps/server
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    volumes:
      - ./apps/server:/var/www
    environment:
      - APP_ENV=local
    networks:
      - app-network

  frontend:
    build:
      context: ./apps/web
      dockerfile: Dockerfile
    ports:
      - "8081:3000"
    depends_on:
      - backend
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### 📄 Backend `Dockerfile`

```dockerfile
FROM php:8.2-fpm

# Install dependencies
RUN apt-get update && apt-get install -y \
    git curl zip unzip sqlite3

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Install PHP extensions
RUN docker-php-ext-install pdo pdo_sqlite

WORKDIR /var/www

# Copy files
COPY . .

# Install dependencies
RUN composer install --no-dev --optimize-autoloader

# Set permissions
RUN chown -R www-data:www-data /var/www

# Expose port
EXPOSE 8000

# Start script
ENTRYPOINT ["/var/www/docker-entrypoint.sh"]
```

---

## 9. Startup Process

### 🚀 `start.sh` (Root del proyecto)

**Ubicación:** `/start.sh`

```bash
#!/bin/bash

# 1. Build Docker images
docker-compose build

# 2. Start containers
docker-compose up -d

# 3. Wait for backend
sleep 5

# 4. Run migrations
docker-compose exec backend php artisan migrate --force

# 5. Start smee.io relay
smee --url https://smee.io/ijWeSRCsn4gxUvS \
     --target http://localhost:8000/api/webhooks/smee &

echo "✅ AREA is running!"
echo "📱 Frontend: http://localhost:8081"
echo "🔧 Backend:  http://localhost:8000"
```

### 🛠️ `docker-entrypoint.sh` (Backend)

**Ubicación:** `apps/server/docker-entrypoint.sh`

```bash
#!/bin/bash

# 1. Generar clave de aplicación
php artisan key:generate --force

# 2. Crear base de datos si no existe
touch database/database.sqlite

# 3. Ejecutar migraciones
php artisan migrate --force

# 4. Limpiar caché
php artisan config:clear
php artisan cache:clear

# 5. Iniciar servidor
php artisan serve --host=0.0.0.0 --port=8000
```

**¿Qué hace?**
1. **Genera clave APP_KEY** para encriptación
2. **Crea SQLite** si no existe
3. **Ejecuta migraciones** (crea tablas)
4. **Limpia caché** de Laravel
5. **Inicia servidor** en puerto 8000

---

## 10. Key Files & Locations

### 📍 Rutas de la API

**Archivo:** `apps/server/routes/api.php`

```php
// ========== WORKFLOWS ==========
Route::get('/workflows/action-types', [WorkflowController::class, 'getActionTypes']);
Route::get('/workflows/reaction-types', [WorkflowController::class, 'getReactionTypes']);
Route::get('/workflows', [WorkflowController::class, 'index']);
Route::post('/workflows', [WorkflowController::class, 'store']);
Route::post('/workflows/{id}/toggle', [WorkflowController::class, 'toggle']);
Route::delete('/workflows/{id}', [WorkflowController::class, 'destroy']);

// ========== WEBHOOKS ==========
Route::post('/webhooks/smee', [WorkflowWebhookController::class, 'handleGitHub']);
Route::post('/discord/webhook', [DiscordWebhookController::class, 'handle']);

// ========== OAUTH ==========
Route::get('/auth/github/redirect', [GitHubAuthController::class, 'redirect']);
Route::get('/auth/github/callback', [GitHubAuthController::class, 'callback']);
// ... otros servicios OAuth
```

### 📍 Procesamiento de Webhooks

**Archivo:** `apps/server/app/Http/Controllers/WorkflowWebhookController.php`

**Flujo:**
```php
public function handleGitHub(Request $request)
{
    // 1. Detectar evento (issues, pull_request, push, watch)
    $event = $request->header('X-GitHub-Event');
    
    // 2. Mapear evento a action_type
    $actionType = $this->mapEventToActionType($event);
    
    // 3. Buscar workflows activos que coincidan
    $workflows = Workflow::where('action_type', $actionType)
        ->where('is_active', true)
        ->get();
    
    // 4. Ejecutar cada workflow
    foreach ($workflows as $workflow) {
        $this->executeWorkflow($workflow, $payload);
    }
}

private function executeWorkflow($workflow, $payload)
{
    switch ($workflow->reaction_type) {
        case 'send_discord_message':
            $this->sendDiscordMessage($workflow, $payload);
            break;
        case 'send_email':
            $this->sendEmail($workflow, $payload);
            break;
    }
}
```

### 📍 Smee.io (Webhook Relay)

**¿Qué es?**
- Servicio que reenvía webhooks de servicios externos (GitHub) a tu localhost
- GitHub → smee.io → localhost:8000

**Configuración:**
```bash
# URL pública de smee
https://smee.io/ijWeSRCsn4gxUvS

# Comando para iniciar
smee --url https://smee.io/ijWeSRCsn4gxUvS \
     --target http://localhost:8000/api/webhooks/smee
```

**Webhook de GitHub apunta a:** `https://smee.io/ijWeSRCsn4gxUvS`

---

## 🎓 Defense Key Points

### 1. **¿Cómo se crean los workflows?**
- **Frontend:** Usuario usa `AreaBuilder.tsx` (drag & drop visual)
- **API Call:** POST `/api/workflows`
- **Backend:** `WorkflowController@store()` valida y guarda en BD
- **Almacenamiento:** Tabla `workflows` en SQLite

### 2. **¿Dónde se definen actions/reactions?**
- **Código:** `Workflow.php` (constantes)
- **API:** `/api/workflows/action-types` y `/reaction-types`
- **Frontend:** Cargadas en `AreaBuilder.tsx`

### 3. **¿Cómo funcionan los webhooks?**
- **GitHub** envía evento → **smee.io** → **localhost:8000/api/webhooks/smee**
- **WorkflowWebhookController** recibe y procesa
- Busca workflows que coincidan con el evento
- Ejecuta reacciones (Discord, Email, etc.)

### 4. **¿Qué hace start.sh?**
1. Build Docker images
2. Start containers
3. Run migrations
4. Start smee relay

### 5. **¿Cómo funciona OAuth?**
- Cada servicio tiene su controlador
- Tokens guardados en `oauth_tokens`
- Usados para llamar APIs de servicios

---

## 📊 Database ER Diagram

```
┌─────────────┐
│   users     │
├─────────────┤
│ id (PK)     │
│ name        │
│ email       │
│ password    │
└──────┬──────┘
       │
       │ 1:N
       │
┌──────▼──────────┐
│  oauth_tokens   │
├─────────────────┤
│ id (PK)         │
│ user_id (FK)    │
│ service         │
│ access_token    │
│ refresh_token   │
│ expires_at      │
└─────────────────┘

┌─────────────────┐
│   workflows     │
├─────────────────┤
│ id (PK)         │
│ user_identifier │
│ name            │
│ action_type     │
│ action_config   │
│ reaction_type   │
│ reaction_config │
│ is_active       │
└─────────────────┘
```

---

## 🔧 Environment Variables (.env)

```env
# App
APP_NAME="AREA"
APP_ENV=local
APP_KEY=base64:...
APP_URL=http://localhost:8000

# Database
DB_CONNECTION=sqlite
DB_DATABASE=/var/www/database/database.sqlite

# GitHub OAuth
GITHUB_CLIENT_ID=Ov23liSuMAy9WnEyiV4z
GITHUB_CLIENT_SECRET=...
GITHUB_REDIRECT_URI=http://localhost:8000/api/auth/github/callback

# Discord OAuth + Bot
DISCORD_CLIENT_ID=1422963310881968248
DISCORD_CLIENT_SECRET=...
DISCORD_BOT_TOKEN=...
DISCORD_REDIRECT_URI=http://localhost:8000/api/auth/discord/callback

# Email (Gmail)
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=mauricio.delanuez@gmail.com
MAIL_PASSWORD=xdlzmvadfyahnsvd
MAIL_ENCRYPTION=tls

# Spotify, Twitch, Google...
```

---

## 📞 Contact & Support

- **Repository:** G-DEV-500-BAR-5-1-area-3
- **Owner:** EpitechPGE3-2025
- **Branch:** master

---

**Good luck with your defense! 🚀**
