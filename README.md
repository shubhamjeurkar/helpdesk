# Helpdesk API

A learning-oriented Helpdesk API with clean layering, type-safe data access, and production-style workflows (migrations, seeding, pagination, RBAC-ready).

## Tech Stack

- **Runtime**: Node.js 20+
- **Framework**: NestJS
- **ORM**: Prisma
- **Database**: PostgreSQL 16 (Docker)
- **Language**: TypeScript
- **Tooling**: Docker Compose, class-validator, JWT-ready

## Features

### Current
- Domain models: Organization, User, Ticket, Comment
- Cursor pagination for ticket lists
- Prisma migrations and Studio
- Ready for RBAC (roles: customer, agent, admin)
- Optimistic locking field on tickets (version)

### Planned
- JWT auth & guards (org scoping)
- Request validation & DTOs everywhere
- Better indices & SQL optimizations

## Quick Start

### Prerequisites
- Node 20+, npm 10+
- Docker Desktop (or Docker Engine) running

### 1. Clone & Install
```bash
git clone https://github.com/shubhamjeurkar/helpdesk.git helpdesk
cd helpdesk
npm install
```

### 2. Start PostgreSQL (Docker)
The project includes a `docker-compose.yml` file:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: helpdesk
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d helpdesk"]
      interval: 5s
      timeout: 3s
      retries: 10
volumes:
  pg_data:
```

Start the database:
```bash
docker compose up -d
```

> **Note**: If you already have a local Postgres on port 5432, either stop it or map to `5433:5432` and update `.env` accordingly.

### 3. Environment Variables
Create a `.env` file in the project root:

```ini
# Use 127.0.0.1 to force TCP (avoids socket/peer auth quirks)
DATABASE_URL="postgresql://app:app@127.0.0.1:5432/helpdesk?schema=public"

# Optional: make Prisma migrations more robust locally
# SHADOW_DATABASE_URL="postgresql://app:app@127.0.0.1:5432/postgres?schema=public"
```

### 4. Prisma: Migrate & Generate
```bash
npx prisma migrate dev --name init
npx prisma generate

# Optional GUI:
npx prisma studio
```

### 5. Seed Minimal Data (Optional)
```bash
npm run seed
```

### 6. Run the API
```bash
npm run dev
# Server should be available at http://localhost:3000
```

## NPM Scripts

```json
{
  "scripts": {
    "dev": "nest start --watch",
    "prisma:migrate": "prisma migrate dev",
    "prisma:studio": "prisma studio",
    "prisma:format": "prisma format",
    "seed": "ts-node prisma/seed.ts"
  }
}
```

## Project Structure

```
src/
  app.module.ts
  main.ts
  prisma/
    prisma.module.ts
    prisma.service.ts
  tickets/
    tickets.module.ts
    tickets.controller.ts
    tickets.service.ts
    dto/
      create-ticket.dto.ts
prisma/
  schema.prisma
  migrations/
  seed.ts
docker-compose.yml
.env
```

## Database Model

### Tables
- **Organization**: `id`, `name`, `slug`, `createdAt`
- **User**: `id`, `email` (unique), `password`, `role` (customer|agent|admin), `orgId`
- **Ticket**: `id`, `title`, `content?`, `status`, `priority`, `version`, `assigneeId?`, `reporterId`, `orgId`, timestamps
- **Comment**: `id`, `ticketId`, `authorId`, `body`, `createdAt`

### Indexes
Optimized for common queries:
- **Ticket**: `@@index([orgId, status, priority, createdAt])`

## API Examples

### Create a Ticket
```bash
curl -X POST http://localhost:3000/tickets \
  -H "Content-Type: application/json" \
  -d '{"title":"Login not working","content":"User gets 500 on login","priority":"high"}'
```

### List Tickets (Cursor Pagination)
```bash
curl "http://localhost:3000/tickets?limit=10"
```

Response format:
```json
{
  "data": [ ...tickets ],
  "meta": { 
    "hasMore": true, 
    "nextCursor": "<id>", 
    "limit": 10 
  }
}
```

> **Note**: In early development, `orgId`/`reporterId` may be hardcoded until JWT auth is implemented.

## Development Notes

### Auth & Multitenancy (Upcoming)
- JWT payload: `{ sub: userId, orgId, role }`
- Guards ensure org scoping (`req.user.orgId` vs entity `orgId`)
- Policy helpers for RBAC (e.g., who can assign/close tickets)

### Optimistic Locking
- `Ticket.version` increments on update
- Use `where: { id, version }` to detect concurrent edits

### Performance
- Use `select`/`include` carefully to avoid N+1 queries
- Add indexes for frequently filtered fields (status/assignee/createdAt)
- Use `$queryRaw` for complex SQL (CTEs/window functions) when needed


## Testing Strategy (Suggested)

- **Unit**: Services & policies as pure functions
- **Integration**: Spin up DB container; run migrations; test repositories
- **e2e**: Nest TestingModule + supertest (auth → create ticket → comment → assign → close)
