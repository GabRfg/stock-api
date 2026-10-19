# Stock Control API

**English** · [Português](readme-PT-BR.md)

Multi-tenant REST API for inventory management. Each company (organization) gets its own users, products and warehouses, and stock levels are derived from a ledger of entry/exit movements rather than stored as a mutable counter.

**Stack:** Node.js · Express · Sequelize · PostgreSQL (MySQL / SQL Server drivers included) · JWT · bcrypt

---

## What it does

- **Self-service onboarding** — creating an organization automatically provisions its first `admin` user with a randomly generated password, returned once in the response.
- **Role-based access** — two roles, `admin` and `employee`. Admins manage users and the organization; both roles operate products, inventories and movements.
- **Tenant scoping** — the `organization_id` comes from the JWT, not from the request body, so product, inventory and user queries are filtered by the caller's company.
- **Movement ledger** — every stock change is recorded as an `in` or `out` movement tied to a product, an inventory and the user who made it. Querying an inventory returns its current balance per product, computed from that history.

## Design decisions

| Decision | Why | Trade-off |
| --- | --- | --- |
| Stock balance is computed from movements (`src/fns/get-product-movements.js`) | Full audit trail: you always know who moved what, and the balance can't drift from the history | Balance is recalculated on every read — fine at small scale, would need a snapshot/materialized total for large ledgers |
| Tenant ID taken from the token | A client cannot ask for another company's data just by changing a field in the body | Every service method must remember to filter by it (see *Known limitations*) |
| Layered structure: `route → api → service → model` | Routes only wire middleware, controllers handle HTTP, services hold business rules | More files per entity than a flat Express app |
| Random admin password on organization creation | No default credentials ever exist | Password is shown only once; there is no reset flow yet |
| Passwords hashed with bcrypt (cost 12); JWT expires in 1 hour | Standard, slow-hash storage and short-lived tokens | No refresh token — the client must log in again |

## Data model

```
organizations ─┬─< users
               ├─< products
               └─< inventories ─< inventoryMovements >─ products
                                         │
                                         └─ users (who made the movement)
```

## Running locally

**Requirements:** Node.js 18+ and a PostgreSQL database.

```bash
git clone https://github.com/GabRfg/StockControl.git
cd StockControl
npm install
```

Create a `.env` file in the project root:

```env
DB_DIALECT=postgres
DB_HOST=localhost
DB_PORT=5432
DB_NAME=stock_control
DB_USER=postgres
DB_PASSWORD=your_password
```

Start the API (tables are created automatically on first run via `sequelize.sync`):

```bash
npm run dev
```

The server listens on `http://localhost:3000`.

## API

All routes are prefixed with `/api/v1`. Protected routes expect the JWT in the `Authorization` header.

| Method | Route | Access | Description |
| --- | --- | --- | --- |
| POST | `/organization` | public | Create an organization and its admin user |
| POST | `/login` | public | Returns a JWT |
| GET / PUT / DELETE | `/organization/:id` | admin | Manage the organization |
| GET | `/user/info` | any | Current user's profile |
| PUT / DELETE | `/user` | any | Update or delete own account |
| GET / POST | `/user` | admin | List / create users in the organization |
| GET / PUT / DELETE | `/user/:id` | admin | Manage a specific user |
| GET / POST | `/products` | any | List / create products |
| GET / PUT / DELETE | `/products/:id` | any | Manage a product |
| GET / POST | `/inventory` | any | List / create inventories |
| GET | `/inventory/:id` | any | Inventory details **with current balance per product** |
| PUT / DELETE | `/inventory/:id` | any | Manage an inventory |
| GET / POST | `/inventoryMovement/:inventoryId` | any | List / register movements (`type`: `in` or `out`) |
| GET / PUT / DELETE | `/inventoryMovement/:inventoryId/:id` | any | Manage a movement |

**Example — register a stock entry**

```http
POST /api/v1/inventoryMovement/1
Authorization: <token>
Content-Type: application/json

{ "type": "in", "amount": 50, "productId": 3 }
```

## Project structure

```
src/
├── route/        # Express routers and role checks per endpoint
├── api/          # Controllers: read the request, call the service, shape the response
├── service/      # Business rules and validation
├── model/        # Sequelize models and associations
├── middleware/   # JWT authentication and role authorization
├── fns/          # Helpers (balance calculation, password generator)
└── database.js   # Sequelize connection from environment variables
```

## Known limitations / roadmap

- [ ] Movement routes and `/organization/:id` do not yet verify that the resource belongs to the caller's organization — tenant check to be enforced in the service layer
- [ ] JWT secret is hard-coded; move to an environment variable
- [ ] `amount` accepts any value and `out` movements can make balance negative — add validation
- [ ] Errors always return HTTP 500; map validation/not-found errors to 400/404
- [ ] Automated tests (Jest is installed, no tests yet)
- [ ] Replace `sync()` with migrations

## License

MIT
