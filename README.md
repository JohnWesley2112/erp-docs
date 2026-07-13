# Enterprise ERP — Technical Case Study

## 1. Project Overview & Problem Statement

This project is a modular Enterprise Resource Planning (ERP) application under active development. It uses a React 19 / Vite single-page application (SPA) for the operational interface and a TypeScript Express API backed by PostgreSQL through Prisma ORM. The foundation currently concentrates on Identity and Access Management (IAM): user creation, role assignment, a permissions matrix, login, and protected client routes.

An ERP becomes difficult to evolve when each business domain—such as finance, procurement, inventory, HR, or education administration—owns isolated data and authorization rules. The design addresses that problem by establishing a shared identity layer and a consistent API/data boundary before domain modules are added. This supports:

- **Consistent authorization:** permissions are modeled as data rather than scattered as hard-coded UI conditions.
- **Composable modules:** future ERP areas can be represented as permission records and implemented as independent API modules without replacing the IAM core.
- **Relational integrity:** PostgreSQL relationships prevent orphaned role-permission records and Prisma provides a typed access layer over them.
- **Operational scale:** a separate SPA and API allow each tier to be deployed and scaled independently behind the supplied Nginx configuration.

The present schema is single-organization: there is no tenant or organization key yet. Multi-tenancy is therefore a planned extension, not a current capability. A robust next design would introduce `Organization` and scoped membership/role assignments, then require tenant scope on every domain query and unique constraint.

## 2. System Architecture & Folder Pattern

The backend follows a **modular layered architecture**. Each business capability is located under `src/modules/<module>` and exposes routes, controllers, services, types/DTOs, and mappers where appropriate. Requests move from the HTTP boundary through application logic to Prisma; controllers do not embed SQL or database queries. Prisma is currently the persistence boundary rather than a separately named repository layer.

```text
E:\erp
├── web.erp/                              # React/Vite client
│   ├── src/
│   │   ├── api/                           # Axios instance and feature API helpers
│   │   ├── components/gaurd/              # Client-side route authorization
│   │   ├── layout/                        # Authenticated/blank shells, navigation
│   │   ├── pages/ and views/              # Route-level and feature UI
│   │   ├── router/                        # Lazy route configuration and role IDs
│   │   ├── store/                         # Redux Toolkit slices and typed hooks
│   │   ├── theme/                         # MUI theme composition
│   │   ├── App.tsx                        # App initialization and router provider
│   │   └── main.tsx                       # React root, Redux provider, suspense
│   └── vite.config.ts                     # /erp base path and build configuration
├── api.erp/                              # Express/TypeScript API
│   ├── prisma/
│   │   ├── schemas/                       # Prisma datasource and IAM model files
│   │   └── migrations/                    # Versioned PostgreSQL DDL
│   └── src/
│       ├── configs/                       # Prisma client and Swagger configuration
│       ├── errors/                        # AppError and centralized error handler
│       ├── helpers/                       # JWT and password helpers
│       ├── middlewares/                   # Authentication middleware
│       ├── modules/
│       │   ├── auth/                      # Login controller/service/mapper/routes
│       │   ├── iam/                       # Roles and permission-matrix reads
│       │   └── user/                      # User CRUD orchestration
│       ├── utils/                         # Prisma error mapping
│       ├── app.ts                         # Express cross-cutting middleware
│       └── server.ts                      # Versioned route mounting and startup
├── api.documentation/                     # API/Postman documentation assets
└── nginx/                                 # Local reverse-proxy configuration
```

```text
React view → API helper → Axios interceptor → /api/v1 route
                                                  │
                                           controller (HTTP/validation)
                                                  │
                                           service (use-case logic)
                                                  │
                                           Prisma Client → PostgreSQL
```

This pattern keeps change local. Adding, for example, Inventory should add `modules/inventory` and a client feature/API helper rather than enlarge a global controller or database utility. Controllers remain testable HTTP adapters, services retain reusable business policies, and Prisma calls are kept near the business operation that owns them. It also makes a future repository abstraction straightforward if query reuse, multiple stores, or more complex transaction boundaries justify it.

## 3. Database Design & Schema Modeling

Prisma is configured with PostgreSQL through `DATABASE_URL`, with schemas split into `base.prisma` (generator/datasource) and `iam.prisma` (domain model). `prisma generate` runs before development and builds, producing a typed Prisma Client. Service methods consequently use typed operations such as `prisma.user.create`, `findUnique`, `findMany`, relation `include`, and relation `select`; `Prisma.UserCreateInput` is used when creating a user.

| Model | Purpose and relationships |
| --- | --- |
| `User` | Login identity with unique, indexed `userEmail`, bcrypt password hash, and audit timestamps. It has a many-to-many `assignedRoles` relation with `Role`. |
| `Role` | A named authority grouping. It connects many users and many permission records through `RolePermission`. |
| `Permission` | Master catalog of ERP modules/features, identified by a unique `permissionName` (for example, `students` or `fees`). |
| `RolePermission` | Explicit role/permission join matrix. Its compound primary key `(roleId, permissionId)` permits each role/module pair once and stores CRUD plus restriction/exception flags. |

The User–Role relationship is Prisma’s implicit many-to-many association. Role–Permission is intentionally explicit because the relationship carries business attributes: `canRead`, `canAdd`, `canEdit`, `canDelete`, `isRestricted`, and `isExceptional`. This is the correct schema shape for a dynamic CRUD matrix; a simple join table could not hold action-level access decisions.

The model maps TypeScript-oriented fields to stable snake_case PostgreSQL table/column names with `@map`/`@@map`. Primary keys use database-generated integers; `createdAt` and `updatedAt` use PostgreSQL `TIMESTAMPTZ`, with `@updatedAt` managed by Prisma. Cascading deletes from roles or permissions to the matrix prevent dangling permission assignments.

Structural evolution is versioned under `prisma/migrations`. The latest migration replaces a coarse permission-level flag design with `role_permissions`, moving CRUD flags to the role-permission pair and adding database foreign keys. This preserves the key architectural decision: authorization changes are schema-managed and deployable, rather than ad hoc production edits. Prisma’s generated types, relational API, constraints, and centralized Prisma-error mapping reduce drift between application assumptions and PostgreSQL reality.

## 4. Security & Authentication Architecture

### Implemented request and authentication flow

1. `POST /api/v1/auth/login` validates the email/password presence in the controller.
2. `AuthService` retrieves the user with assigned roles, compares the password with bcrypt, and returns a signed one-day JWT plus role IDs through `AuthMapper`.
3. The React login form stores the access token and role-ID array. Axios injects `Authorization: Bearer <token>` on subsequent requests.
4. `authenticate` validates the bearer format and JWT signature, then attaches identity claims to the request for downstream handlers.
5. The global error handler maps known Prisma and operational errors to controlled JSON responses and sends errors to Loggerverse.

JWT claims include the user ID, name, email, and role values. The project also contains a seven-day refresh-token generator, although refresh-token issuance, storage, rotation, and a refresh endpoint are not yet wired into the request flow.

### RBAC design

The server-side IAM query `getAllUserPermissions(userId)` resolves every role assigned to the user, reads matching `RolePermission` rows, and aggregates each action with an **allow-if-any-role-allows** rule. This produces a per-permission effective policy suitable for module/action checks. On the client, Redux loads roles and permission mappings concurrently; `selectUserPermission` implements the same matrix lookup for UI affordances. `ProtectedRoute` also guards route groups using role IDs—currently, the user-creation path is limited to `SUPER_ADMIN` and `ADMIN` IDs—and redirects unauthenticated/unauthorized users to login or the 401 page.

The client guard improves navigation and prevents accidental exposure, but it is deliberately not the security boundary: local storage and browser code are controlled by the client. The API must independently authenticate every protected route and authorize the requested action.

### Current security hardening required

The authentication middleware exists but is commented out in `server.ts`, and IAM routes are mounted before that commented protection point. As checked in, those API routes are not protected by JWT middleware and no reusable backend `authorize(permission, action)` middleware is mounted. This is the principal gap before production use.

Recommended next implementation:

- Mount `authenticate` ahead of all protected API routers; explicitly preserve only public endpoints such as login.
- Add an authorization middleware that resolves the authenticated user’s effective permission for a route/action and returns `403` on denial.
- Keep the permission selector for UX only; hide or disable controls and navigation from the same server-issued/effective policy.
- Move tokens to secure, `HttpOnly`, `SameSite` cookies (or carefully mitigate the XSS risk of local storage), configure CORS with explicit origins/credentials, add refresh rotation/revocation, rate-limit login, and validate request DTOs.
- Remove development `console.log` output that can disclose claims or role information and standardize the JWT payload field names consumed by middleware.

## 5. Frontend Performance Engineering

The client is a TypeScript React 19 SPA built with Vite and Material UI. State is deliberately separated by concern: `CustomizerSlice` stores presentation/layout state, while `PermissionSlice` stores authorization bootstrap data, current user data, loading state, and errors. Redux Toolkit’s `createAsyncThunk` coordinates role and permission retrieval with `Promise.all`, avoiding serial startup network calls. Typed store hooks provide a consistent integration point for feature components.

Route-level code splitting is implemented through `React.lazy` and the `Loadable` wrapper. Layouts, login/signup, home, permissions, error screens, and user creation are loaded only when their route is visited; both route wrappers and the root use `Suspense` with a spinner fallback. This keeps the initial ERP shell smaller as modules multiply. The Vite configuration also sets a deployable `/erp/` base path and uses optimized dependency handling.

The current performance strategy is appropriate for the active IAM scope. As data-intensive ERP modules arrive, extend it with paginated/filterable server endpoints, request cancellation and cache invalidation, virtualization for large tables, route/module prefetching for common workflows, and measured bundle chunk boundaries. Avoid putting every form and table record in global Redux: retain short-lived component state locally and reserve global state for cross-route, shared concerns such as session, permissions, and UI preferences.

## 6. Current Status & Next Steps

### Current phase

The project is in the **IAM and application-shell foundation phase**. Working foundations include the React/Vite/MUI shell, Redux store, lazy route definitions, login UI and JWT issuance, bcrypt password hashing, Prisma/PostgreSQL schema and migrations, user creation with role assignment, IAM read endpoints, structured API error handling, Swagger setup, and slow-query/error logging hooks.

The active product surface is intentionally concentrated on authorization administration: roles, permissions, permission-to-role matrix reads, protected client routing, and super-admin user creation. ERP domain modules (such as finance, purchasing, inventory, HR, or domain-specific operations) are not represented in the Prisma schema or route modules yet.

### Next engineering milestones

1. Make server-side JWT authentication and action-level RBAC mandatory on every protected route; add authorization tests for both allow and deny cases.
2. Complete IAM write operations and audited administration flows for role-permission assignments; provide an effective-permissions endpoint for the signed-in user.
3. Define the first bounded ERP domain module with its own Prisma models, migration, API module, route-level UI, ownership rules, and integration tests.
4. Establish production controls: DTO validation, secure session/token lifecycle, explicit CORS policy, secrets management, CI migrations, health checks, metrics, structured logs, and API contract tests.
5. If the product serves multiple organizations, introduce tenant ownership and enforce tenant scoping in database constraints, Prisma queries, authorization policies, and observability metadata from the first domain migration.

The architecture is well positioned for this expansion because IAM is already a first-class module and the codebase separates UI, HTTP, business, and persistence responsibilities. The most important transition from foundation to production is enforcing the same RBAC policy at the API boundary that the frontend currently uses for navigation and presentation.
