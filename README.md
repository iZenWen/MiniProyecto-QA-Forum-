# MiniProyecto-QA-Forum-
Full-stack Q&amp;A forum built with React, Node.js, TypeScript and PostgreSQL

# QA Forum
 
Foro de preguntas y respuestas estilo Stack Overflow, construido como proyecto de portfolio para prácticas de empresa. Permite publicar preguntas, responder, votar contenido y acumular reputación.
 
> **Contexto:** Proyecto desarrollado por un estudiante de 3º de Ingeniería Informática (especialización Software) con experiencia previa en React. Objetivo: completar el proyecto fullstack en 10 días (~70h).
 
---
 
## Índice
 
- [Demo](#demo)
- [Stack tecnológico](#stack-tecnológico)
- [Arquitectura](#arquitectura)
- [Modelo de datos](#modelo-de-datos)
- [Endpoints de la API](#endpoints-de-la-api)
- [Autenticación JWT](#autenticación-jwt)
- [Plan de implementación](#plan-de-implementación)
- [Estado actual](#estado-actual)
- [Cómo levantar el proyecto](#cómo-levantar-el-proyecto)
- [Variables de entorno](#variables-de-entorno)
- [Tests](#tests)
- [Convenciones](#convenciones)
 
---
 
## Demo
 
> _URL disponible cuando se complete el deploy en Railway/Render_
 
---
 
## Stack tecnológico
 
### Frontend
 
| Tecnología | Versión | Uso |
|---|---|---|
| React | 19.x | Framework de UI |
| Vite | 6.x | Bundler y servidor de desarrollo |
| TypeScript | 5.x | Tipado estático |
| Tailwind CSS | 4.2.2 | Estilos utilitarios (plugin `@tailwindcss/vite`) |
| React Router | 7.x | Navegación entre páginas |
| TanStack Query | 5.x | Gestión de estado del servidor y caché |
| Axios | 1.x | Cliente HTTP con interceptores |
| React Hook Form | 7.x | Gestión de formularios y validación |
 
### Backend
 
| Tecnología | Versión | Uso |
|---|---|---|
| Node.js | 20.x | Entorno de ejecución JavaScript |
| Express | 4.x | Framework HTTP para la API REST |
| TypeScript | 5.x | Tipado estático |
| Prisma ORM | 6.x | Acceso tipado a la base de datos |
| PostgreSQL | 16 | Base de datos relacional |
| JWT (jsonwebtoken) | 9.x | Autenticación sin sesión |
| bcrypt | 5.x | Hash seguro de contraseñas |
| Zod | 3.x | Validación de schemas en runtime |
| helmet | 8.x | Cabeceras HTTP de seguridad |
| cors | 2.x | Control de acceso entre orígenes |
| express-rate-limit | 7.x | Limitación de peticiones |
 
### Infraestructura y herramientas
 
| Herramienta | Uso |
|---|---|
| Docker + Docker Compose | Contenedores para app y base de datos |
| Jest + Supertest | Tests de integración de la API |
| ESLint + Prettier | Calidad y formato de código |
| GitHub Actions | Integración continua (CI) |
| swagger-ui-express | Documentación interactiva de la API en `/api-docs` |
 
---
 
## Arquitectura
 
### Estructura de carpetas
 
```
qa-forum/
├── backend/
│   ├── src/
│   │   ├── app.ts              # Express app (sin listen, para tests)
│   │   ├── server.ts           # Punto de entrada (listen)
│   │   ├── routes/             # Define las URLs disponibles
│   │   ├── controllers/        # Recibe peticiones, orquesta respuesta
│   │   ├── services/           # Lógica de negocio
│   │   ├── repositories/       # Acceso a BD a través de Prisma
│   │   ├── middlewares/        # Auth, roles, errores, validación
│   │   ├── types/              # Interfaces y tipos TypeScript
│   │   └── config/             # Configuración general
│   ├── prisma/
│   │   ├── schema.prisma       # Modelos y relaciones de la BD
│   │   └── seed.ts             # Script de datos de prueba
│   ├── .env                    # Variables de entorno (NO en git)
│   ├── .env.example            # Plantilla de variables (SÍ en git)
│   ├── Dockerfile
│   ├── package.json
│   └── tsconfig.json
├── frontend/
│   ├── src/
│   │   ├── pages/              # Componentes de página completa
│   │   ├── components/         # Componentes reutilizables
│   │   ├── hooks/              # Custom hooks de React
│   │   ├── lib/                # Cliente Axios configurado
│   │   └── types/              # Tipos TypeScript compartidos
│   ├── vite.config.ts
│   └── package.json
├── docker-compose.yml
└── .github/
    └── workflows/
        └── ci.yml
```
 
### Arquitectura en capas del backend
 
Cada petición HTTP recorre las siguientes capas en orden:
 
```
Petición HTTP
      │
      ▼
   Router          → identifica qué controller gestiona la URL
      │
      ▼
  Middleware        → verifica JWT (auth) y permisos (roles)
      │
      ▼
  Controller        → recibe datos validados, llama al service
      │
      ▼
   Service          → lógica de negocio (votos, reputación, permisos)
      │
      ▼
  Repository        → queries a PostgreSQL a través de Prisma
      │
      ▼
  Respuesta HTTP
```
 
Esta separación garantiza que cada capa tenga una única responsabilidad. Si se cambia la base de datos, solo se tocan los repositories. Si cambia la lógica de negocio, solo los services.
 
### Comunicación frontend-backend
 
En desarrollo, Vite proxea automáticamente las peticiones del frontend al backend:
 
- Frontend: `http://localhost:5173`
- Backend: `http://localhost:3000`
- Las llamadas a `/api/*` desde el frontend se redirigen al backend vía proxy de Vite
 
El frontend usa **Axios con un interceptor** que añade el Bearer token JWT a cada petición automáticamente. **TanStack Query** gestiona el caché, los estados de carga y los errores de todas las llamadas a la API.
 
---
 
## Modelo de datos
 
### Diagrama de entidades
 
```
User
 ├── id, email, password, username, reputation, createdAt
 ├── → Question[] (author)
 ├── → Answer[] (author)
 ├── → Vote[]
 ├── → Comment[]
 └── → RefreshToken[]
 
Question
 ├── id, title, body, views, createdAt, authorId
 ├── → User (author)
 ├── → Answer[]
 ├── → Vote[]
 ├── → Comment[]
 └── → Tag[] (via QuestionTag)
 
Answer
 ├── id, body, isAccepted, createdAt, authorId, questionId
 ├── → User (author)
 ├── → Question
 ├── → Vote[]
 └── → Comment[]
 
Vote
 ├── id, value (+1 / -1), userId
 ├── → User
 ├── → Question? (opcional)
 └── → Answer? (opcional)
 
Tag
 ├── id, name, description
 └── → Question[] (via QuestionTag)
 
Comment
 ├── id, body, createdAt, authorId
 ├── → User
 ├── → Question? (opcional)
 └── → Answer? (opcional)
 
RefreshToken
 ├── id, token, expiresAt, userId
 └── → User
```
 
### Reglas de negocio de la base de datos
 
- Un usuario **no puede votar su propio contenido**
- Si un usuario vota algo que ya había votado, el segundo voto **cancela el primero** (toggle)
- Al votar, la reputación del autor se actualiza **atómicamente** en la misma transacción Prisma
- Solo puede haber **una respuesta aceptada** por pregunta (`isAccepted = true`)
- Solo el **autor de la pregunta** puede aceptar una respuesta
- La búsqueda full-text usa **`to_tsvector` de PostgreSQL** vía `Prisma.$queryRaw`
 
---
 
## Endpoints de la API
 
La documentación interactiva completa está disponible en `/api-docs` (Swagger UI).
 
### Autenticación — `/api/auth`
 
| Método | Ruta | Auth | Descripción |
|---|---|---|---|
| POST | `/api/auth/register` | No | Registrar nuevo usuario |
| POST | `/api/auth/login` | No | Login, devuelve access + refresh token |
| POST | `/api/auth/refresh` | No | Renovar access token con refresh token |
| POST | `/api/auth/logout` | Sí | Invalidar refresh token |
 
### Preguntas — `/api/questions`
 
| Método | Ruta | Auth | Descripción |
|---|---|---|---|
| GET | `/api/questions` | No | Listar preguntas (paginado, filtros por tag/texto/orden) |
| POST | `/api/questions` | Sí | Crear pregunta con título, cuerpo y tags |
| GET | `/api/questions/:id` | No | Detalle con respuestas, tags y votos |
| PATCH | `/api/questions/:id` | Sí (autor) | Editar pregunta |
| DELETE | `/api/questions/:id` | Sí (autor) | Eliminar pregunta |
| POST | `/api/questions/:id/vote` | Sí | Votar pregunta (`{ "value": 1 }` o `{ "value": -1 }`) |
| POST | `/api/questions/:id/comments` | Sí | Añadir comentario a la pregunta |
 
### Respuestas — `/api/answers`
 
| Método | Ruta | Auth | Descripción |
|---|---|---|---|
| POST | `/api/questions/:id/answers` | Sí | Responder a una pregunta |
| PATCH | `/api/answers/:id` | Sí (autor) | Editar respuesta |
| DELETE | `/api/answers/:id` | Sí (autor) | Eliminar respuesta |
| PATCH | `/api/answers/:id/accept` | Sí (autor pregunta) | Marcar como respuesta correcta |
| POST | `/api/answers/:id/vote` | Sí | Votar respuesta |
| POST | `/api/answers/:id/comments` | Sí | Añadir comentario a la respuesta |
 
### Tags y Usuarios
 
| Método | Ruta | Auth | Descripción |
|---|---|---|---|
| GET | `/api/tags` | No | Listar todos los tags con conteo de preguntas |
| GET | `/api/users/:id` | No | Perfil público con reputación y actividad |
| GET | `/api/users/me` | Sí | Perfil propio del usuario autenticado |
 
### Formato de respuestas
 
```jsonc
// Éxito
{ "data": { ... }, "message": "string" }
 
// Error
{ "error": "string", "details": { ... } }
```
 
### Códigos HTTP utilizados
 
| Código | Significado |
|---|---|
| 200 | OK — petición exitosa |
| 201 | Created — recurso creado |
| 400 | Bad Request — datos inválidos |
| 401 | Unauthorized — no autenticado |
| 403 | Forbidden — sin permisos |
| 404 | Not Found — recurso no existe |
| 429 | Too Many Requests — rate limit |
| 500 | Internal Server Error |
 
---
 
## Autenticación JWT
 
### Flujo completo
 
```
1. POST /api/auth/login  →  { email, password }
2. Backend verifica contraseña con bcrypt.compare()
3. Genera dos tokens:
   - Access token  → expira en 15 minutos
   - Refresh token → expira en 7 días, se guarda en BD
4. Frontend guarda tokens
5. Axios interceptor añade "Authorization: Bearer <token>" en cada petición
6. Cuando el access token expira, el interceptor llama a /api/auth/refresh
7. En logout, el refresh token se elimina de la BD
```
 
### Middleware de autenticación
 
El `authMiddleware` extrae el Bearer token del header `Authorization`, verifica la firma JWT con la clave secreta del `.env`, y si es válido añade el usuario decodificado a `req.user`. Si el token es inválido o ha expirado, responde con `401 Unauthorized`.
 
---
 
## Plan de implementación
 
| Día | Horas | Contenido | Capa |
|---|---|---|---|
| 1 | 5h | Setup fullstack: monorepo, Express+TS, React+Vite, Docker Compose, GitHub Actions | Infra |
| 2 | 7h | Schema Prisma completo, migraciones, seed, endpoints de autenticación JWT | Backend |
| 3 | 7h | CRUD de preguntas, tags, búsqueda full-text con `to_tsvector`, paginación | Backend |
| 4 | 7h | Respuestas, sistema de votos con transacciones atómicas, lógica de reputación | Backend |
| 5 | 6h | Comentarios, perfil de usuario, seguridad (helmet/cors/rate-limit), Swagger UI | Backend |
| 6 | 6h | Frontend: React Router, Axios con interceptor, TanStack Query, login/registro | Frontend |
| 7 | 7h | Frontend: home con listado, búsqueda debounced, filtro por tags, detalle de pregunta | Frontend |
| 8 | 7h | Frontend: publicar pregunta, responder, comentar, aceptar respuesta, votar | Frontend |
| 9 | 7h | Frontend: perfil de usuario, tests de integración Jest+Supertest | Frontend + Tests |
| 10 | 5h | README, diagrama de arquitectura, screenshots, deploy opcional, release v1.0 | Docs |
 
---
 
## Estado actual
 
### ✅ Día 1 — Completado
 
- [x] Repositorio GitHub público creado y configurado
- [x] Estructura monorepo con `/backend` y `/frontend`
- [x] Backend: Express + TypeScript con arquitectura en capas
- [x] Backend: ESLint, Prettier y tsconfig.json configurados
- [x] Backend: endpoint `GET /health` respondiendo JSON
- [x] Backend: variables de entorno con `.env` y `.env.example`
- [x] Frontend: React + Vite + TypeScript inicializado
- [x] Frontend: Tailwind CSS v4.2.2 con plugin `@tailwindcss/vite`
- [x] Frontend: React Router, TanStack Query, Axios y React Hook Form instalados
- [x] Frontend: proxy de Vite apuntando al backend en puerto 3000
- [x] Docker Compose: levanta backend + PostgreSQL con `docker compose up`
- [x] GitHub Actions: workflow de CI ejecutando lint en cada push a `main`
 
### 🔄 Día 2 — En progreso
 
- [ ] Schema Prisma completo con todos los modelos
- [ ] Primera migración y script seed
- [ ] Endpoints de autenticación (register, login, refresh, logout)
 
---
 
## Cómo levantar el proyecto
 
### Requisitos previos
 
- Node.js v20 o superior
- Docker Desktop instalado y en ejecución
- Git
 
### Pasos
 
```bash
# 1. Clonar el repositorio
git clone https://github.com/TU_USUARIO/qa-forum.git
cd qa-forum
 
# 2. Configurar variables de entorno del backend
cd backend
cp .env.example .env
cd ..
 
# 3. Levantar backend + base de datos con Docker
docker compose up
 
# 4. En otra terminal, levantar el frontend
cd frontend
npm install
npm run dev
```
 
| Servicio | URL |
|---|---|
| Backend (API) | http://localhost:3000 |
| Frontend | http://localhost:5173 |
| Swagger UI | http://localhost:3000/api-docs _(disponible desde el día 5)_ |
| Health check | http://localhost:3000/health |
 
---
 
## Variables de entorno
 
Copia `backend/.env.example` a `backend/.env` y rellena los valores:
 
```env
PORT=3000
NODE_ENV=development
DATABASE_URL=postgresql://qauser:qapassword@localhost:5432/qaforum
JWT_ACCESS_SECRET=tu_clave_secreta_access
JWT_REFRESH_SECRET=tu_clave_secreta_refresh
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
```
 
---
 
## Tests
 
Los tests de integración usan Jest + Supertest y se ejecutan contra una base de datos de test separada.
 
```bash
cd backend
npm test
```
 
Los tests cubren:
 
- Auth: registro, login, token inválido (401)
- Preguntas: crear, listar, buscar, eliminar sin auth (403)
- Votos: upvote, downvote, toggle, no votar tu propio contenido (400)
- Respuestas: crear, aceptar, solo el autor de la pregunta puede aceptar (403)
 
---
 
## Convenciones
 
### Commits
 
Se usan commits convencionales en español:
 
```
feat: nueva funcionalidad
fix: corrección de error
refactor: reestructuración sin cambio de comportamiento
test: tests nuevos o modificados
docs: cambios en documentación
chore: configuración, dependencias
```
 
### Estilo de código
 
- Indentación: 2 espacios
- Comillas: simples
- Punto y coma: sí
- Trailing comma: es5
- ESLint + Prettier se ejecutan automáticamente en el CI
 
---
 
_Proyecto desarrollado como miniproyecto de portfolio · Ingeniería Informática · 2026_