# Store Ratings Platform

A full-stack web application where users submit 1–5 star ratings for registered
stores. A single login serves three roles — **System Administrator**, **Normal
User**, and **Store Owner** — each with role-specific functionality.

> **Stack:** React (Vite) · Express.js · MySQL 8 · JWT auth · bcrypt

---

## Table of contents
- [Architecture](#architecture)
- [Project structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Database](#database)
- [Default accounts](#default-accounts)
- [API reference](#api-reference)
- [Validation rules](#validation-rules)
- [Feature checklist](#feature-checklist)

---

## Architecture

```
React SPA (Vite, :5173)  ──HTTP/JSON──>  Express API (:4000)  ──>  MySQL (:3306)
        │                                      │
   AuthContext + JWT in              routes → controllers → services → SQL
   localStorage, axios               (express-validator, JWT + role guards)
```

- **Backend** follows a layered structure: `routes → controllers → services`,
  with cross-cutting concerns (auth, validation, errors) in `middleware`.
- **Auth** is stateless JWT. Passwords are hashed with bcrypt. A single
  `/auth/login` endpoint serves every role; the token carries the role, and
  route guards (`authorize(...roles)`) enforce access.
- **SQL** uses a connection pool and **parameterized queries** everywhere
  (no string concatenation), and sort columns are whitelisted — both guard
  against SQL injection.

## Project structure

```
prasad project/
├─ backend/
│  ├─ src/
│  │  ├─ config/        env loading + MySQL pool
│  │  ├─ db/            schema.sql, init.js, seed.js
│  │  ├─ middleware/    auth, validation, error handling
│  │  ├─ validators/    express-validator chains (challenge rules)
│  │  ├─ services/      data access (users, stores, ratings)
│  │  ├─ controllers/   request/response handlers
│  │  ├─ routes/        route definitions + role guards
│  │  ├─ utils/         password, token, ApiError, rules
│  │  ├─ app.js         express app (helmet, cors, rate-limit)
│  │  └─ server.js      entry point
│  └─ .env              local configuration
└─ frontend/
   └─ src/
      ├─ api/           axios client (+ JWT interceptor)
      ├─ context/       AuthContext
      ├─ components/    Navbar, ProtectedRoute, DataTable, StarRating, Field
      ├─ pages/         login, signup, admin/*, user/*, owner/*
      └─ styles.css
```

## Prerequisites

- **Node.js 18+** (developed on Node 24)
- **MySQL 8** — a portable instance is set up under `D:\mysql-portable`
  (see [Database](#database)). Any MySQL 8 server works; just update
  `backend/.env`.

## Quick start

```bash
# 1) Backend
cd backend
npm install
# edit .env if your MySQL host/user/password differ
npm run db:init     # create database + tables
npm run db:seed     # load demo users, stores, ratings
npm run dev         # API on http://localhost:4000

# 2) Frontend (new terminal)
cd frontend
npm install
npm run dev         # app on http://localhost:5173
```

Open http://localhost:5173 and log in with one of the [default accounts](#default-accounts).

## Database

The connection is configured in `backend/.env`:

```
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=
DB_NAME=store_ratings
```

### Portable MySQL (this machine)

MySQL 8.0 was installed as a portable (no-admin) instance at
`D:\mysql-portable`. To start / stop it:

```powershell
# start (run in a normal PowerShell window; keep it open)
D:\mysql-portable\start-mysql.ps1

# stop
D:\mysql-portable\stop-mysql.ps1
```

### Schema (3 tables)

- **users** — `id, name, email (unique), password_hash, address, role(admin|user|owner)`
- **stores** — `id, name, email (unique), address, owner_id → users.id`
- **ratings** — `id, user_id, store_id, rating(1–5)`, unique on `(user_id, store_id)`

A user rates each store at most once (`UNIQUE(user_id, store_id)`); submitting
again updates the existing rating. Overall ratings are computed with `AVG()`.

## Default accounts

After `npm run db:seed`:

| Role          | Email                          | Password    |
|---------------|--------------------------------|-------------|
| Administrator | `admin@storeratings.com`       | `Admin@1234` |
| Normal User   | `olivia.thompson@example.com`  | `User@1234`  |
| Store Owner   | `daniel.reeves@example.com`    | `Owner@1234` |

(Other seeded users share the `User@1234` / `Owner@1234` passwords.)

## API reference

All protected routes require `Authorization: Bearer <token>`.

| Method | Path                        | Role        | Purpose                              |
|--------|-----------------------------|-------------|--------------------------------------|
| POST   | `/api/auth/register`        | public      | Normal-user sign up                  |
| POST   | `/api/auth/login`           | public      | Login (all roles)                    |
| GET    | `/api/auth/me`              | any         | Current user                         |
| PUT    | `/api/auth/password`        | any         | Change own password                  |
| GET    | `/api/admin/stats`          | admin       | Dashboard totals                     |
| GET    | `/api/users`                | admin       | List users (filter + sort)           |
| POST   | `/api/users`                | admin       | Create user (any role)               |
| GET    | `/api/users/:id`            | admin       | User detail (owner rating included)  |
| GET    | `/api/stores`               | admin, user | List stores (search + sort)          |
| POST   | `/api/stores`               | admin       | Create store                         |
| POST   | `/api/stores/:id/ratings`   | user        | Submit / modify a rating             |
| GET    | `/api/owner/dashboard`      | owner       | Owner's stores, raters, avg rating   |

List endpoints accept `?name=&email=&address=&role=&sortBy=&order=asc|desc`.

## Validation rules

Enforced on **both** client and server:

| Field     | Rule                                                              |
|-----------|-------------------------------------------------------------------|
| Name      | 20–60 characters                                                  |
| Address   | ≤ 400 characters                                                  |
| Password  | 8–16 chars, ≥ 1 uppercase letter and ≥ 1 special character        |
| Email     | Standard email format                                             |
| Rating    | Integer 1–5                                                       |

## Feature checklist

**System Administrator** — add stores/users/admins · dashboard totals (users,
stores, ratings) · list & filter users and stores · user detail with owner
rating · sortable tables · logout.

**Normal User** — sign up · login · change password · browse all stores ·
search by name/address · see overall + own rating · submit/modify rating ·
logout.

**Store Owner** — login · change password · dashboard with average rating and
the list of users who rated their store(s) · logout.
