# Basic Kanban

Kanban board (estilo do mockup de referência) construído em **monorepo** com **Node.js** no backend e **React** no frontend, organizado por **módulos** seguindo o artigo [Module-Based Code Organization](https://medium.com/@alyssonbarrera.s/module-based-code-organization-48091ee917b0).

O escopo do produto é apenas o board (colunas + cards arrastáveis), sem sidebar/topbar do mockup.

---

## Stack

### Geral
- **pnpm workspaces** + **Turborepo** — orquestração do monorepo
- **TypeScript** em todos os pacotes
- **Biome** (lint + format)

### Backend (`apps/api`)
- **Fastify** — servidor HTTP
- **Zod** + `fastify-type-provider-zod` — validação e tipagem de rotas
- **Drizzle ORM** + **PostgreSQL** — persistência e migrations
- **@fastify/cors**, **@fastify/jwt** — middlewares

### Frontend (`apps/web`)
- **React 19** + **Vite**
- **TanStack Router** — roteamento file-based e type-safe
- **TanStack Query** — cache de dados do servidor
- **Atlaskit** — UI primitives do Atlassian Design System
  - `@atlaskit/pragmatic-drag-and-drop` + `@atlaskit/pragmatic-drag-and-drop-hitbox` — drag-and-drop entre colunas
  - `@atlaskit/avatar`, `@atlaskit/tag`, `@atlaskit/lozenge`, `@atlaskit/dropdown-menu`, `@atlaskit/button`, `@atlaskit/textfield` — componentes do card e da coluna

---

## Estrutura do monorepo

```
basic_kanban/
├── apps/
│   ├── api/                       # Fastify + Zod + Drizzle
│   └── web/                       # React + TanStack + Atlaskit
├── packages/
│   ├── contracts/                 # Zod schemas/DTOs compartilhados (api ↔ web)
│   ├── tsconfig/                  # tsconfig.base.json e presets
│   └── biome-config/              # config do Biome compartilhada
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
└── README.md
```

`packages/contracts` exporta os schemas Zod das entidades (Board, Column, Card) — o backend valida o request com eles e o frontend tipa o response sem duplicar definição.

---

## Backend — `apps/api`

Organização por módulos (um módulo por entidade do domínio do kanban).

```
apps/api/
├── src/
│   ├── modules/
│   │   ├── boards/
│   │   │   ├── boards.routes.ts        # registra rotas no Fastify
│   │   │   ├── boards.controller.ts    # handlers (req → service → reply)
│   │   │   ├── boards.service.ts       # regras de negócio
│   │   │   ├── boards.repository.ts    # acesso ao banco via Drizzle
│   │   │   ├── boards.schema.ts        # schemas Zod (input/output)
│   │   │   └── boards.dto.ts           # tipos inferidos dos schemas
│   │   ├── columns/
│   │   │   └── ... (mesma estrutura)
│   │   └── cards/
│   │       └── ... (mesma estrutura)
│   ├── core/
│   │   ├── errors/                     # AppError, NotFoundError, etc.
│   │   ├── plugins/                    # plugins Fastify (zod, cors, jwt)
│   │   ├── utils/                      # helpers genéricos
│   │   └── env.ts                      # parse de env com Zod
│   ├── infra/
│   │   ├── db/
│   │   │   ├── client.ts               # instância do Drizzle
│   │   │   ├── schema/                 # tabelas: boards, columns, cards
│   │   │   └── migrations/             # output do drizzle-kit
│   │   └── http/
│   │       └── server.ts               # bootstrap do Fastify
│   └── app.ts                          # registra plugins + módulos
├── drizzle.config.ts
├── package.json
└── tsconfig.json
```

### Modelo de dados (Drizzle)

```
boards   (id, title, created_at)
columns  (id, board_id, title, position)
cards    (id, column_id, position, title, description, assignee_name,
          assignee_avatar_url, due_date, priority, tags[])
```

`position` é numérico (float ou inteiro com gaps) para reordenação por drag-and-drop sem reescrever a coluna inteira.

### Rotas

```
GET    /boards/:id           # board com columns e cards aninhados
POST   /columns              # cria coluna
PATCH  /columns/:id          # renomeia / reordena
DELETE /columns/:id
POST   /cards                # cria card
PATCH  /cards/:id            # edita / move (column_id + position)
DELETE /cards/:id
```

Toda rota declara `schema: { body, params, response }` em Zod — validação, serialização e tipos do handler vêm daí.

---

## Frontend — `apps/web`

Organização por módulos seguindo o artigo: tudo que é específico de uma feature mora dentro do módulo; só sobe para `core/` quando é reusado por múltiplos módulos.

```
apps/web/
├── src/
│   ├── modules/
│   │   └── board/
│   │       ├── components/
│   │       │   ├── board.tsx               # container do kanban
│   │       │   ├── column.tsx              # coluna (header + lista de cards + "+ New task")
│   │       │   ├── column-header.tsx       # status, contador, menu
│   │       │   ├── card.tsx                # card draggable
│   │       │   ├── card-meta.tsx           # avatar, data, prioridade, tags
│   │       │   └── new-task-button.tsx
│   │       ├── dtos/
│   │       │   ├── board.dto.ts
│   │       │   ├── column.dto.ts
│   │       │   └── card.dto.ts
│   │       ├── forms/
│   │       │   ├── card-form.tsx           # form de criar/editar card
│   │       │   ├── card-form.hook.ts       # react-hook-form + zodResolver
│   │       │   └── card-form.action.ts     # submit → mutation
│   │       ├── http/
│   │       │   ├── get-board.ts            # useQuery
│   │       │   ├── create-card.ts          # useMutation
│   │       │   ├── update-card.ts          # move/edita (otimista)
│   │       │   └── delete-card.ts
│   │       ├── screens/
│   │       │   └── board.screen.tsx        # composição da página
│   │       ├── validations/
│   │       │   └── card.schema.ts          # Zod schemas (reexporta de @contracts)
│   │       └── utils/
│   │           └── reorder.ts              # cálculo de position no drop
│   ├── core/
│   │   ├── components/                     # botões, modais, inputs genéricos
│   │   ├── hooks/                          # useDebounce, useMediaQuery
│   │   ├── lib/
│   │   │   ├── query-client.ts             # TanStack Query
│   │   │   └── atlaskit.tsx                # AtlaskitThemeProvider
│   │   └── utils/                          # formatDate, cn, etc.
│   ├── infra/
│   │   ├── auth/                           # contexto/JWT (placeholder)
│   │   └── http/
│   │       └── api-client.ts               # fetch wrapper + base URL
│   ├── routes/                             # TanStack Router (file-based)
│   │   ├── __root.tsx
│   │   └── boards.$boardId.tsx             # renderiza BoardScreen
│   ├── main.tsx
│   └── routeTree.gen.ts
├── index.html
├── vite.config.ts
├── package.json
└── tsconfig.json
```

### Drag-and-drop (Pragmatic DnD)

- `card.tsx` registra `draggable({ element, getInitialData: { cardId, columnId } })`.
- `column.tsx` registra `dropTargetForElements` na lista e em cada card (para inserir entre cards).
- No `onDrop`, calcula a nova `position` (média entre vizinhos) e dispara `updateCard` com **update otimista** no cache do TanStack Query.

### Mapeamento UI ↔ Atlaskit (com base no mockup)

| Elemento do mockup           | Componente                                    |
|------------------------------|-----------------------------------------------|
| Header da coluna (status)    | `@atlaskit/lozenge` + contador                |
| Menu `...` da coluna/card    | `@atlaskit/dropdown-menu`                     |
| Avatar + nome no card        | `@atlaskit/avatar`                            |
| Tag "Medium" / "High Priority" | `@atlaskit/tag` ou `@atlaskit/lozenge`      |
| Data / location / attachments| ícones `@atlaskit/icon` + texto               |
| "+ New task"                 | `@atlaskit/button` (subtle)                   |
| Drag-and-drop                | `@atlaskit/pragmatic-drag-and-drop`           |

---

## Scripts

Na raiz (via Turborepo):

```bash
pnpm install
pnpm dev          # roda api + web em paralelo
pnpm build
pnpm lint
pnpm typecheck
```

No `apps/api`:

```bash
pnpm dev                    # tsx watch src/infra/http/server.ts
pnpm db:generate            # drizzle-kit generate
pnpm db:migrate             # aplica migrations
pnpm db:studio              # drizzle studio
```

No `apps/web`:

```bash
pnpm dev                    # vite
pnpm build
```

---

## Variáveis de ambiente

`apps/api/.env`
```
DATABASE_URL=postgres://user:pass@localhost:5432/kanban
PORT=3333
JWT_SECRET=change-me
```

`apps/web/.env`
```
VITE_API_URL=http://localhost:3333
```

Ambos os arquivos são parseados por um schema Zod em `core/env.ts` — a app falha no boot se faltar variável.

---

## Como começar

```bash
git clone <repo>
cd basic_kanban
pnpm install
docker compose up -d postgres        # ou Postgres local
pnpm --filter api db:migrate
pnpm dev
```

Acesse `http://localhost:5173/boards/<id>`.
