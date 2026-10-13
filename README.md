# Drizzle ORM: Complete Guide

A practical guide to Drizzle ORM for TypeScript: setup, schema, migrations, queries, relations, transactions, and tooling. Examples use **PostgreSQL** unless noted. MySQL, SQLite, and other dialects are very similar, and the differences are listed in each section.

> **Note:** Drizzle moves fast. Some APIs (for example, the relational query "v2" in the 1.0 line) may differ by version. Always check the official docs at <https://orm.drizzle.team> for your installed version.

---

## Table of Contents

1. [What is Drizzle ORM?](#1-what-is-drizzle-orm)
2. [Installation](#2-installation)
3. [Project Structure](#3-project-structure)
4. [Connecting to a Database](#4-connecting-to-a-database)
5. [Configuration File (drizzle.config.ts)](#5-configuration-file-drizzleconfigts)
6. [Schema Declaration](#6-schema-declaration)
7. [Relations](#7-relations)
8. [Migrations and Drizzle Kit Commands](#8-migrations-and-drizzle-kit-commands)
9. [CRUD Queries](#9-crud-queries)
10. [Filtering Operators](#10-filtering-operators)
11. [Ordering, Pagination, Distinct](#11-ordering-pagination-distinct)
12. [Aggregations and Group By](#12-aggregations-and-group-by)
13. [Joins](#13-joins)
14. [Subqueries, CTEs, Set Operations](#14-subqueries-ctes-set-operations)
15. [Relational Queries API](#15-relational-queries-api)
16. [Raw SQL and the sql Operator](#16-raw-sql-and-the-sql-operator)
17. [Prepared Statements](#17-prepared-statements)
18. [Dynamic Query Building](#18-dynamic-query-building)
19. [Transactions](#19-transactions)
20. [Batch API](#20-batch-api)
21. [Views](#21-views)
22. [Type Inference](#22-type-inference)
23. [Validation with drizzle-zod](#23-validation-with-drizzle-zod)
24. [Seeding with drizzle-seed](#24-seeding-with-drizzle-seed)
25. [Row-Level Security (PostgreSQL)](#25-row-level-security-postgresql)
26. [Logging and Debugging](#26-logging-and-debugging)
27. [Testing](#27-testing)
28. [Best Practices](#28-best-practices)
29. [Command Cheat Sheet](#29-command-cheat-sheet)
30. [Troubleshooting](#30-troubleshooting)

---

## 1. What is Drizzle ORM?

Drizzle is a lightweight, TypeScript-first ORM with two query styles:

- **SQL-like API**: `db.select().from(users).where(...)`. It looks like SQL and gives full control.
- **Relational API**: `db.query.users.findMany({ with: { posts: true } })`. It is good for nested data.

Key points:

- Fully typed schema, written in TypeScript (no separate schema language).
- No code generation step for types. Types come from your schema file.
- Zero runtime dependencies. Works in Node.js, Bun, Deno, Cloudflare Workers, Vercel Edge, and serverless.
- **Drizzle Kit** is the companion CLI for migrations and Drizzle Studio.

Supported dialects: PostgreSQL, MySQL, SQLite, SingleStore, MSSQL (check the docs for the latest status), plus providers such as Neon, Supabase, Vercel Postgres, PlanetScale, Turso (libSQL), Cloudflare D1, and others.

---

## 2. Installation

```bash
# Core
npm i drizzle-orm
npm i -D drizzle-kit

# Helper tools
npm i -D tsx            # run .ts files (seed scripts, migrations)
npm i dotenv            # load .env file
```

Install **one** database driver:

| Database / Provider | Driver package |
| --- | --- |
| PostgreSQL (node-postgres) | `pg` and `@types/pg` |
| PostgreSQL (postgres.js) | `postgres` |
| Neon (serverless) | `@neondatabase/serverless` |
| Vercel Postgres | `@vercel/postgres` |
| PGlite (in-process Postgres) | `@electric-sql/pglite` |
| MySQL | `mysql2` |
| PlanetScale | `@planetscale/database` |
| SQLite (Node) | `better-sqlite3` and `@types/better-sqlite3` |
| libSQL / Turso | `@libsql/client` |
| Cloudflare D1 | built in (use the D1 binding) |
| Bun SQLite | built in (`bun:sqlite`) |

Example (PostgreSQL with node-postgres):

```bash
npm i drizzle-orm pg dotenv
npm i -D drizzle-kit tsx @types/pg typescript
```

---

## 3. Project Structure

A common layout:

```
.
├── drizzle/                  # generated migration files (SQL + meta)
├── src/
│   ├── db/
│   │   ├── index.ts          # database connection
│   │   ├── schema.ts         # tables, enums, relations
│   │   └── seed.ts           # optional seed script
│   └── ...
├── drizzle.config.ts         # Drizzle Kit config
├── .env                      # DATABASE_URL
└── package.json
```

For larger projects, split the schema into many files (for example `src/db/schema/users.ts`, `posts.ts`) and point `schema` in the config file to the folder or a glob.

---

## 4. Connecting to a Database

### PostgreSQL (node-postgres)

```ts
// src/db/index.ts
import 'dotenv/config';
import { drizzle } from 'drizzle-orm/node-postgres';
import * as schema from './schema';

export const db = drizzle(process.env.DATABASE_URL!, { schema });
```

Using an existing pool or client:

```ts
import { Pool } from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle({ client: pool, schema });
```

### PostgreSQL (postgres.js)

```ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';

const client = postgres(process.env.DATABASE_URL!);
export const db = drizzle({ client, schema });
```

### Neon (HTTP, serverless)

```ts
import { neon } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-http';

const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle({ client: sql, schema });
```

### MySQL

```ts
import { drizzle } from 'drizzle-orm/mysql2';
import mysql from 'mysql2/promise';

const connection = await mysql.createConnection(process.env.DATABASE_URL!);
export const db = drizzle({ client: connection, schema, mode: 'default' });
```

### SQLite (better-sqlite3)

```ts
import { drizzle } from 'drizzle-orm/better-sqlite3';
import Database from 'better-sqlite3';

const sqlite = new Database('sqlite.db');
export const db = drizzle({ client: sqlite, schema });
```

### libSQL / Turso

```ts
import { drizzle } from 'drizzle-orm/libsql';

export const db = drizzle({
  connection: { url: process.env.TURSO_URL!, authToken: process.env.TURSO_TOKEN },
  schema,
});
```

### Cloudflare D1

```ts
import { drizzle } from 'drizzle-orm/d1';

export default {
  async fetch(request: Request, env: { DB: D1Database }) {
    const db = drizzle(env.DB, { schema });
    return Response.json(await db.select().from(users));
  },
};
```

### Useful `drizzle()` options

```ts
drizzle(url, {
  schema,                // enable relational queries (db.query.*)
  logger: true,          // log SQL to console
  casing: 'snake_case',  // map camelCase keys to snake_case columns automatically
});
```

---

## 5. Configuration File (drizzle.config.ts)

```ts
// drizzle.config.ts
import 'dotenv/config';
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql',            // 'postgresql' | 'mysql' | 'sqlite' | 'turso' | 'singlestore'
  schema: './src/db/schema.ts',     // file, folder, glob, or array of paths
  out: './drizzle',                 // migrations output folder
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },

  // Optional settings
  casing: 'snake_case',             // keep the same value as in drizzle()
  verbose: true,                    // print all SQL during push/migrate
  strict: true,                     // ask confirmation before running push
  migrations: {
    table: '__drizzle_migrations',  // migrations log table
    schema: 'public',               // (PostgreSQL) schema of the log table
  },
  tablesFilter: ['app_*'],          // only manage matching tables
  schemaFilter: ['public'],         // (PostgreSQL) only manage these schemas
});
```

Multiple schema files:

```ts
schema: ['./src/db/schema/users.ts', './src/db/schema/posts.ts']
// or
schema: './src/db/schema/*'
```

Custom config file name: use `--config` flag (example: `drizzle-kit generate --config=drizzle-prod.config.ts`).

---

## 6. Schema Declaration

All examples use PostgreSQL imports from `drizzle-orm/pg-core`. For other dialects, use `mysql-core` (`mysqlTable`) or `sqlite-core` (`sqliteTable`).

### 6.1 Basic table

```ts
// src/db/schema.ts
import {
  pgTable, serial, integer, text, varchar, boolean, timestamp, uuid,
} from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 255 }).notNull(),
  email: text('email').notNull().unique(),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at').notNull().defaultNow(),
});
```

### 6.2 Common column types (PostgreSQL)

```ts
import {
  integer, smallint, bigint, serial, bigserial, real, doublePrecision, numeric, decimal,
  text, varchar, char, boolean, json, jsonb, date, time, timestamp, interval,
  uuid, inet, cidr, macaddr, point, line, vector, // vector: pgvector extension
} from 'drizzle-orm/pg-core';

export const examples = pgTable('examples', {
  a: integer('a'),
  b: bigint('b', { mode: 'number' }),         // or 'bigint'
  c: numeric('c', { precision: 10, scale: 2 }),
  d: text('d'),
  e: varchar('e', { length: 100 }),
  f: json('f').$type<{ foo: string }>(),      // typed JSON
  g: jsonb('g').$type<string[]>(),
  h: timestamp('h', { withTimezone: true, mode: 'date' }),
  i: timestamp('i', { mode: 'string' }),
  j: uuid('j').defaultRandom(),
  k: text('k').array(),                       // text[]
  l: vector('l', { dimensions: 1536 }),
});
```

MySQL notes: `int`, `tinyint`, `mediumint`, `bigint`, `serial`, `varchar` (length required), `datetime`, `year`, `mysqlEnum`, and `autoincrement()` for IDs.

SQLite notes: `integer`, `real`, `text`, `blob`, `numeric`. Use `integer({ mode: 'timestamp' })`, `integer({ mode: 'boolean' })`, and `text({ mode: 'json' })` for richer types.

### 6.3 Column modifiers

```ts
text('name')
  .notNull()                         // NOT NULL
  .default('guest')                  // static default
  .unique()                          // UNIQUE
  .primaryKey()                      // PRIMARY KEY
  .$default(() => crypto.randomUUID())  // runtime default (done in JS, not in database)
  .$onUpdate(() => new Date().toISOString()) // runtime value on update
  .references(() => users.id)        // foreign key
  .$type<'a' | 'b'>();               // narrow the TypeScript type
```

SQL-level defaults with `sql`:

```ts
import { sql } from 'drizzle-orm';

createdAt: timestamp('created_at').default(sql`now()`),
```

### 6.4 Identity, serial, and UUID primary keys

```ts
// Recommended in modern PostgreSQL
id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
// or
id: integer('id').primaryKey().generatedByDefaultAsIdentity({ startWith: 1000 }),

// UUID
id: uuid('id').primaryKey().defaultRandom(),

// Legacy
id: serial('id').primaryKey(),
```

### 6.5 Foreign keys and referential actions

```ts
export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  authorId: integer('author_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade', onUpdate: 'no action' }),
});
```

Actions: `'cascade' | 'restrict' | 'no action' | 'set null' | 'set default'`.

Self-reference needs an explicit type annotation:

```ts
import { AnyPgColumn } from 'drizzle-orm/pg-core';

export const categories = pgTable('categories', {
  id: serial('id').primaryKey(),
  parentId: integer('parent_id').references((): AnyPgColumn => categories.id),
});
```

### 6.6 Indexes, composite keys, composite foreign keys, constraints

The third argument of the table function returns an array:

```ts
import {
  pgTable, serial, integer, text, index, uniqueIndex, primaryKey,
  foreignKey, unique, check,
} from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const memberships = pgTable(
  'memberships',
  {
    userId: integer('user_id').notNull(),
    groupId: integer('group_id').notNull(),
    role: text('role').notNull(),
    age: integer('age'),
  },
  (t) => [
    primaryKey({ columns: [t.userId, t.groupId] }),               // composite PK
    index('role_idx').on(t.role),                                 // index
    uniqueIndex('user_group_uq').on(t.userId, t.groupId),         // unique index
    unique('role_age_uq').on(t.role, t.age),                      // unique constraint
    foreignKey({
      columns: [t.userId],
      foreignColumns: [users.id],
      name: 'memberships_user_fk',
    }),
    check('age_check', sql`${t.age} >= 18`),                      // check constraint
    index('partial_idx').on(t.role).where(sql`${t.age} > 30`),    // partial index (PostgreSQL)
  ],
);
```

> Older versions used an object return `(t) => ({ ... })`. The array form is the current style.

### 6.7 Enums

PostgreSQL:

```ts
import { pgEnum, pgTable, serial } from 'drizzle-orm/pg-core';

export const roleEnum = pgEnum('role', ['admin', 'editor', 'viewer']);

export const accounts = pgTable('accounts', {
  id: serial('id').primaryKey(),
  role: roleEnum('role').notNull().default('viewer'),
});
```

MySQL: `mysqlEnum('role', ['admin', 'editor'])`. SQLite: `text('role', { enum: ['admin', 'editor'] })`.

### 6.8 Generated columns

```ts
import { sql } from 'drizzle-orm';

export const people = pgTable('people', {
  first: text('first'),
  last: text('last'),
  full: text('full').generatedAlwaysAs((): SQL => sql`${people.first} || ' ' || ${people.last}`),
});
```

### 6.9 Database schemas (PostgreSQL namespaces)

```ts
import { pgSchema } from 'drizzle-orm/pg-core';

export const authSchema = pgSchema('auth');

export const authUsers = authSchema.table('users', {
  id: uuid('id').primaryKey(),
});
```

### 6.10 Sequences and custom types

```ts
import { pgSequence, customType } from 'drizzle-orm/pg-core';

export const invoiceSeq = pgSequence('invoice_seq', { startWith: 1000, increment: 1 });

const bytea = customType<{ data: Buffer; driverData: Buffer }>({
  dataType() { return 'bytea'; },
});
```

### 6.11 Reusable column groups

```ts
const timestamps = {
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow().$onUpdate(() => new Date()),
};

export const products = pgTable('products', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  ...timestamps,
});
```

### 6.12 Automatic column naming (casing)

```ts
// drizzle.config.ts  and  drizzle() must use the same value
casing: 'snake_case'

// schema: no need to write column names
export const users = pgTable('users', {
  id: serial().primaryKey(),
  firstName: text().notNull(),   // column name becomes "first_name"
});
```

---

## 7. Relations

Foreign keys define the **database** relation. The `relations()` helper defines the **query-level** relation used by the relational API (`db.query.*`). It does not create anything in the database.

### 7.1 One-to-many

```ts
import { relations } from 'drizzle-orm';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
});

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  authorId: integer('author_id').notNull().references(() => users.id),
});

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
}));
```

### 7.2 One-to-one

```ts
export const profiles = pgTable('profiles', {
  id: serial('id').primaryKey(),
  bio: text('bio'),
  userId: integer('user_id').notNull().unique().references(() => users.id),
});

export const usersRelations = relations(users, ({ one }) => ({
  profile: one(profiles),
}));

export const profilesRelations = relations(profiles, ({ one }) => ({
  user: one(users, { fields: [profiles.userId], references: [users.id] }),
}));
```

### 7.3 Many-to-many (junction table)

```ts
export const groups = pgTable('groups', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
});

export const usersToGroups = pgTable(
  'users_to_groups',
  {
    userId: integer('user_id').notNull().references(() => users.id),
    groupId: integer('group_id').notNull().references(() => groups.id),
  },
  (t) => [primaryKey({ columns: [t.userId, t.groupId] })],
);

export const usersRelations = relations(users, ({ many }) => ({
  usersToGroups: many(usersToGroups),
}));

export const groupsRelations = relations(groups, ({ many }) => ({
  usersToGroups: many(usersToGroups),
}));

export const usersToGroupsRelations = relations(usersToGroups, ({ one }) => ({
  group: one(groups, { fields: [usersToGroups.groupId], references: [groups.id] }),
  user: one(users, { fields: [usersToGroups.userId], references: [users.id] }),
}));
```

### 7.4 Multiple relations to the same table (use `relationName`)

```ts
export const usersRelations = relations(users, ({ many }) => ({
  authoredPosts: many(posts, { relationName: 'author' }),
  reviewedPosts: many(posts, { relationName: 'reviewer' }),
}));

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id], relationName: 'author' }),
  reviewer: one(users, { fields: [posts.reviewerId], references: [users.id], relationName: 'reviewer' }),
}));
```

### 7.5 Relational API v2 (1.0 line, check your version)

Newer versions introduce `defineRelations` with object-style filters:

```ts
import { defineRelations } from 'drizzle-orm';
import * as schema from './schema';

export const relations = defineRelations(schema, (r) => ({
  users: {
    posts: r.many.posts(),
  },
  posts: {
    author: r.one.users({ from: r.posts.authorId, to: r.users.id }),
  },
}));

// const db = drizzle(url, { relations });
// await db.query.users.findMany({ where: { id: 1 }, with: { posts: true } });
```

---

## 8. Migrations and Drizzle Kit Commands

Drizzle Kit commands are run with `npx drizzle-kit <command>`.

| Command | Purpose |
| --- | --- |
| `generate` | Create SQL migration files from schema changes |
| `migrate` | Apply generated migration files to the database |
| `push` | Push the schema directly to the database (no migration files) |
| `pull` | Introspect an existing database and generate a schema file |
| `check` | Check migrations for conflicts (useful in teams and CI) |
| `up` | Upgrade the snapshot format of old migrations |
| `studio` | Open Drizzle Studio, a web GUI for browsing data |
| `export` | Print the SQL for the current schema (no migration files) |

### 8.1 Generate + migrate workflow (recommended for production)

```bash
npx drizzle-kit generate                       # creates ./drizzle/0000_xxx.sql
npx drizzle-kit generate --name=add_posts      # custom migration name
npx drizzle-kit migrate                        # apply to the database
```

Generated output (example SQL file in the `drizzle` folder):

```sql
CREATE TABLE "users" (
  "id" serial PRIMARY KEY NOT NULL,
  "name" varchar(255) NOT NULL
);
```

Always **review** generated SQL files and commit them to Git.

### 8.2 Push workflow (fast, for prototyping and local development)

```bash
npx drizzle-kit push
npx drizzle-kit push --force      # auto-approve data-loss statements (be careful)
npx drizzle-kit push --verbose    # print SQL
npx drizzle-kit push --strict     # ask confirmation before running SQL
```

### 8.3 Pull (introspection)

```bash
npx drizzle-kit pull
```

This creates `schema.ts` and `relations.ts` files in the `out` folder from an existing database.

### 8.4 Check, up, export, studio

```bash
npx drizzle-kit check                 # detect conflicting migrations
npx drizzle-kit up                    # upgrade old snapshot format
npx drizzle-kit export                # print SQL of the schema
npx drizzle-kit studio                # https://local.drizzle.studio
npx drizzle-kit studio --port=4000 --host=0.0.0.0
```

### 8.5 Custom (empty) migrations

Use these for data migrations, extensions, triggers, or functions:

```bash
npx drizzle-kit generate --custom --name=seed-admin-user
```

Then edit the new SQL file:

```sql
INSERT INTO "users" ("name", "email") VALUES ('Admin', 'admin@example.com');
```

### 8.6 Run migrations from code (at app startup or in CI)

```ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { migrate } from 'drizzle-orm/node-postgres/migrator';

const db = drizzle(process.env.DATABASE_URL!);

await migrate(db, { migrationsFolder: './drizzle' });
```

SQLite example: `import { migrate } from 'drizzle-orm/better-sqlite3/migrator'`.

### 8.7 Package scripts

```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:push": "drizzle-kit push",
    "db:pull": "drizzle-kit pull",
    "db:check": "drizzle-kit check",
    "db:studio": "drizzle-kit studio",
    "db:seed": "tsx src/db/seed.ts"
  }
}
```

### 8.8 Common team workflow

1. Change `schema.ts` file.
2. Run `npm run db:generate`.
3. Review the new SQL file and commit it.
4. Run `npm run db:migrate` (local, then staging, then production).
5. In CI, run `drizzle-kit check` to detect conflicts between branches.

---

## 9. CRUD Queries

Imports used below:

```ts
import { db } from './db';
import { users, posts } from './db/schema';
import { eq, and, or, sql } from 'drizzle-orm';
```

### 9.1 Insert

```ts
// single row
await db.insert(users).values({ name: 'Ana', email: 'ana@example.com' });

// multiple rows
await db.insert(users).values([
  { name: 'Ben', email: 'ben@example.com' },
  { name: 'Cara', email: 'cara@example.com' },
]);

// returning (PostgreSQL, SQLite, MariaDB)
const [created] = await db
  .insert(users)
  .values({ name: 'Dan', email: 'dan@example.com' })
  .returning();

// return only some columns
const ids = await db.insert(users).values({ name: 'Eli', email: 'eli@example.com' })
  .returning({ id: users.id });

// use DEFAULT for a column
await db.insert(users).values({ name: 'Fay', email: 'fay@example.com', createdAt: sql`default` });

// insert from select
await db.insert(archivedUsers).select(
  db.select().from(users).where(eq(users.isActive, false)),
);
```

MySQL returns `[ResultSetHeader]` with `insertId`. It has no `returning()`.

### 9.2 Upsert (on conflict)

```ts
// PostgreSQL and SQLite
await db.insert(users)
  .values({ id: 1, name: 'Ana', email: 'ana@example.com' })
  .onConflictDoNothing();                       // optional: { target: users.email }

await db.insert(users)
  .values({ id: 1, name: 'Ana Updated', email: 'ana@example.com' })
  .onConflictDoUpdate({
    target: users.email,                        // column(s) of the unique constraint
    set: { name: 'Ana Updated' },
    // setWhere: sql`${users.isActive} = true`, // optional condition
  });

// Use the incoming value
await db.insert(users)
  .values({ name: 'Ana', email: 'ana@example.com' })
  .onConflictDoUpdate({ target: users.email, set: { name: sql`excluded.name` } });

// MySQL
await db.insert(users)
  .values({ id: 1, name: 'Ana' })
  .onDuplicateKeyUpdate({ set: { name: 'Ana' } });
```

### 9.3 Select

```ts
// all columns
const all = await db.select().from(users);

// partial select
const names = await db.select({ id: users.id, name: users.name }).from(users);

// with where
const one = await db.select().from(users).where(eq(users.id, 1));

// get one row or undefined
const [first] = await db.select().from(users).where(eq(users.email, 'ana@example.com')).limit(1);

// computed field
const rows = await db.select({
  id: users.id,
  upperName: sql<string>`upper(${users.name})`,
}).from(users);

// count helper
const total = await db.$count(users);
const activeTotal = await db.$count(users, eq(users.isActive, true));
```

### 9.4 Update

```ts
await db.update(users).set({ name: 'New Name' }).where(eq(users.id, 1));

// returning
const [updated] = await db.update(users)
  .set({ isActive: false })
  .where(eq(users.id, 1))
  .returning();

// SQL expression in set
await db.update(products).set({ stock: sql`${products.stock} - 1` }).where(eq(products.id, 5));

// update with undefined values are ignored
await db.update(users).set({ name: undefined, email: 'new@example.com' }).where(eq(users.id, 1));

// update with FROM (PostgreSQL, SQLite)
await db.update(users)
  .set({ name: posts.title })
  .from(posts)
  .where(eq(posts.authorId, users.id));
```

> Always add `.where(...)`. Without it, **all rows** are updated.

### 9.5 Delete

```ts
await db.delete(users).where(eq(users.id, 1));

const deleted = await db.delete(users).where(eq(users.isActive, false)).returning();

await db.delete(users);   // deletes ALL rows. Be careful.
```

For a full table wipe, use raw SQL: `await db.execute(sql\`TRUNCATE TABLE users RESTART IDENTITY CASCADE\`)` (PostgreSQL).

---

## 10. Filtering Operators

```ts
import {
  eq, ne, gt, gte, lt, lte,
  isNull, isNotNull,
  inArray, notInArray,
  between, notBetween,
  like, ilike, notLike, notIlike,
  exists, notExists,
  and, or, not,
  arrayContains, arrayContained, arrayOverlaps,
  sql,
} from 'drizzle-orm';
```

Examples:

```ts
await db.select().from(users).where(eq(users.id, 1));
await db.select().from(users).where(ne(users.name, 'Ana'));
await db.select().from(products).where(gt(products.price, 100));
await db.select().from(products).where(between(products.price, 10, 50));
await db.select().from(users).where(isNull(users.deletedAt));
await db.select().from(users).where(inArray(users.id, [1, 2, 3]));
await db.select().from(users).where(ilike(users.name, '%ana%'));     // PostgreSQL case-insensitive
await db.select().from(users).where(like(users.name, 'A%'));

// combine
await db.select().from(users).where(
  and(
    eq(users.isActive, true),
    or(like(users.email, '%@example.com'), gt(users.id, 100)),
  ),
);

// not
await db.select().from(users).where(not(eq(users.id, 1)));

// exists
await db.select().from(users).where(
  exists(db.select({ one: sql`1` }).from(posts).where(eq(posts.authorId, users.id))),
);

// array operators (PostgreSQL arrays)
await db.select().from(tags).where(arrayContains(tags.names, ['a', 'b']));

// conditional filters: undefined values are skipped inside and() / or()
const filters = [
  name ? ilike(users.name, `%${name}%`) : undefined,
  onlyActive ? eq(users.isActive, true) : undefined,
];
await db.select().from(users).where(and(...filters));
```

---

## 11. Ordering, Pagination, Distinct

```ts
import { asc, desc } from 'drizzle-orm';

// order
await db.select().from(users).orderBy(asc(users.name), desc(users.id));

// limit and offset
await db.select().from(users).orderBy(users.id).limit(10).offset(20);

// distinct
await db.selectDistinct({ name: users.name }).from(users);

// distinct on (PostgreSQL)
await db.selectDistinctOn([users.name], { name: users.name, id: users.id })
  .from(users)
  .orderBy(users.name);

// keyset (cursor) pagination: faster than offset on large tables
import { gt } from 'drizzle-orm';
const page = await db.select().from(users)
  .where(gt(users.id, lastSeenId))
  .orderBy(asc(users.id))
  .limit(20);
```

---

## 12. Aggregations and Group By

```ts
import { count, countDistinct, sum, sumDistinct, avg, avgDistinct, max, min, sql } from 'drizzle-orm';

const stats = await db.select({
  total: count(),                       // COUNT(*)
  withEmail: count(users.email),        // COUNT(email)
  uniqueNames: countDistinct(users.name),
  avgAge: avg(users.age),
  oldest: max(users.age),
  youngest: min(users.age),
  totalSalary: sum(users.salary),
}).from(users);

// group by + having
const perAuthor = await db
  .select({ authorId: posts.authorId, postCount: count() })
  .from(posts)
  .groupBy(posts.authorId)
  .having(({ postCount }) => gt(postCount, 5));

// custom aggregate
const rows = await db.select({
  day: sql<string>`date_trunc('day', ${posts.createdAt})`.as('day'),
  total: count(),
}).from(posts).groupBy(sql`day`);
```

Note: `avg`, `sum` return strings in some drivers. Use `.mapWith(Number)`: `avg(users.age).mapWith(Number)`.

---

## 13. Joins

```ts
// inner join
const rows = await db
  .select({ postTitle: posts.title, authorName: users.name })
  .from(posts)
  .innerJoin(users, eq(posts.authorId, users.id));

// left join (joined table fields become nullable)
const withOptional = await db
  .select()
  .from(users)
  .leftJoin(posts, eq(users.id, posts.authorId));
// result shape: { users: {...}, posts: {...} | null }[]

// right, full, cross
await db.select().from(users).rightJoin(posts, eq(users.id, posts.authorId));
await db.select().from(users).fullJoin(posts, eq(users.id, posts.authorId));
await db.select().from(users).crossJoin(posts);

// multiple joins
await db.select().from(posts)
  .innerJoin(users, eq(posts.authorId, users.id))
  .leftJoin(comments, eq(comments.postId, posts.id));

// self join with alias
import { alias } from 'drizzle-orm/pg-core';
const parent = alias(users, 'parent');
await db.select({ user: users.name, parent: parent.name })
  .from(users)
  .leftJoin(parent, eq(users.parentId, parent.id));

// lateral join (PostgreSQL, MySQL)
const sq = db.select().from(posts).where(eq(posts.authorId, users.id)).limit(3).as('sq');
await db.select().from(users).leftJoinLateral(sq, sql`true`);
```

---

## 14. Subqueries, CTEs, Set Operations

### Subquery in `from`

```ts
const sq = db.select({ authorId: posts.authorId, total: count().as('total') })
  .from(posts)
  .groupBy(posts.authorId)
  .as('sq');

const result = await db.select({ name: users.name, total: sq.total })
  .from(users)
  .innerJoin(sq, eq(users.id, sq.authorId));
```

### Subquery in `where` and `select`

```ts
const avgPrice = db.select({ v: avg(products.price) }).from(products);
await db.select().from(products).where(gt(products.price, avgPrice));

await db.select({
  name: users.name,
  postCount: db.$count(posts, eq(posts.authorId, users.id)),
}).from(users);
```

### CTE (WITH)

```ts
const activeUsers = db.$with('active_users').as(
  db.select().from(users).where(eq(users.isActive, true)),
);

const rows = await db.with(activeUsers).select().from(activeUsers);

// insert/update/delete with CTE
await db.with(activeUsers).delete(users).where(inArray(users.id, db.select({ id: activeUsers.id }).from(activeUsers)));
```

### Set operations

```ts
import { union, unionAll, intersect, intersectAll, except, exceptAll } from 'drizzle-orm/pg-core';

await union(
  db.select({ name: users.name }).from(users),
  db.select({ name: customers.name }).from(customers),
);

// builder style
await db.select({ name: users.name }).from(users)
  .union(db.select({ name: customers.name }).from(customers));
```

---

## 15. Relational Queries API

Requires passing `schema` (and the relations) to `drizzle()`.

```ts
// findMany
const all = await db.query.users.findMany();

// with relations
const usersWithPosts = await db.query.users.findMany({
  with: { posts: true },
});

// nested relations
const deep = await db.query.users.findMany({
  with: {
    posts: {
      with: { comments: true },
    },
  },
});

// select only some columns
await db.query.users.findMany({
  columns: { id: true, name: true },           // include
  // columns: { email: false },                // or exclude
});

// filter relations
await db.query.users.findMany({
  with: {
    posts: {
      where: (posts, { eq }) => eq(posts.published, true),
      orderBy: (posts, { desc }) => [desc(posts.createdAt)],
      limit: 5,
    },
  },
});

// filter and order main table
await db.query.users.findMany({
  where: (users, { eq, and, like }) => and(eq(users.isActive, true), like(users.name, 'A%')),
  orderBy: (users, { asc }) => [asc(users.name)],
  limit: 10,
  offset: 0,
});

// findFirst (adds LIMIT 1, returns object or undefined)
const user = await db.query.users.findFirst({
  where: (users, { eq }) => eq(users.id, 1),
  with: { profile: true },
});

// extra computed fields
import { sql } from 'drizzle-orm';
await db.query.users.findMany({
  extras: {
    loweredName: sql<string>`lower(${users.name})`.as('lowered_name'),
  },
});

// many-to-many through the junction table
const usersWithGroups = await db.query.users.findMany({
  with: { usersToGroups: { with: { group: true } } },
});

// prepared
const prepared = db.query.users.findMany({
  where: (users, { eq }) => eq(users.id, sql.placeholder('id')),
}).prepare('get_user_by_id');
await prepared.execute({ id: 1 });
```

Use the relational API when you need nested data in one query. Use the SQL-like API when you need joins, aggregates, or full SQL control.

---

## 16. Raw SQL and the sql Operator

```ts
import { sql } from 'drizzle-orm';

// inside queries (values are parameterized, safe from SQL injection)
await db.select().from(users).where(sql`${users.name} = ${userInput}`);

// typed result
const rows = await db.select({
  lower: sql<string>`lower(${users.name})`,
  count: sql<number>`count(*)`.mapWith(Number),
}).from(users);

// alias
sql`count(*)`.as('total');

// run a full raw statement
const result = await db.execute(sql`select * from users where id = ${1}`);
// node-postgres: result.rows

// raw (unescaped) strings: dangerous with user input
sql.raw('ORDER BY id DESC');

// join fragments
const conditions = [sql`a = 1`, sql`b = 2`];
sql.join(conditions, sql` and `);

// identifiers
sql`select * from ${users}`;               // table reference
sql`${sql.identifier('my_table')}`;        // identifier
```

> Never build SQL by joining raw user input strings. Use the `sql` template tag with `${value}` so values are passed as parameters.

---

## 17. Prepared Statements

Prepared statements are parsed once and executed many times. They are useful for hot paths.

```ts
import { sql } from 'drizzle-orm';

const getUser = db
  .select()
  .from(users)
  .where(eq(users.id, sql.placeholder('id')))
  .prepare('get_user');          // name is required for PostgreSQL

const u1 = await getUser.execute({ id: 1 });
const u2 = await getUser.execute({ id: 2 });

// with insert
const insertUser = db.insert(users)
  .values({ name: sql.placeholder('name'), email: sql.placeholder('email') })
  .prepare('insert_user');
await insertUser.execute({ name: 'Ana', email: 'ana@example.com' });

// limit and offset placeholders
const paged = db.select().from(users)
  .limit(sql.placeholder('limit'))
  .offset(sql.placeholder('offset'))
  .prepare('paged_users');
await paged.execute({ limit: 10, offset: 0 });
```

---

## 18. Dynamic Query Building

Calling `.where()` twice normally throws an error. Use `$dynamic()` to build queries in steps:

```ts
import type { PgSelect } from 'drizzle-orm/pg-core';

function withPagination<T extends PgSelect>(qb: T, page = 1, size = 10) {
  return qb.limit(size).offset((page - 1) * size);
}

let query = db.select().from(users).$dynamic();

if (onlyActive) query = query.where(eq(users.isActive, true));
query = withPagination(query, 2, 20);

const rows = await query;
```

---

## 19. Transactions

```ts
await db.transaction(async (tx) => {
  const [account] = await tx.select().from(accounts).where(eq(accounts.id, 1));

  if (account.balance < 100) {
    tx.rollback();                 // throws and rolls back the transaction
  }

  await tx.update(accounts).set({ balance: sql`${accounts.balance} - 100` }).where(eq(accounts.id, 1));
  await tx.update(accounts).set({ balance: sql`${accounts.balance} + 100` }).where(eq(accounts.id, 2));
});

// return a value
const newUser = await db.transaction(async (tx) => {
  const [u] = await tx.insert(users).values({ name: 'Ana', email: 'a@x.com' }).returning();
  await tx.insert(profiles).values({ userId: u.id });
  return u;
});

// nested transaction (savepoint)
await db.transaction(async (tx) => {
  await tx.insert(users).values({ name: 'Outer', email: 'o@x.com' });
  try {
    await tx.transaction(async (tx2) => {
      await tx2.insert(users).values({ name: 'Inner', email: 'i@x.com' });
      tx2.rollback();              // rolls back only the inner savepoint
    });
  } catch {}
});

// transaction options (PostgreSQL)
await db.transaction(async (tx) => { /* ... */ }, {
  isolationLevel: 'serializable',  // 'read uncommitted' | 'read committed' | 'repeatable read' | 'serializable'
  accessMode: 'read write',        // or 'read only'
  deferrable: true,
});

// SQLite specific
await db.transaction(async (tx) => { /* ... */ }, { behavior: 'immediate' }); // 'deferred' | 'immediate' | 'exclusive'
```

Any thrown error inside the callback rolls back the transaction. Use `tx`, not `db`, for every query inside.

> `neon-http` and `d1` drivers do not support interactive transactions. Use the [Batch API](#20-batch-api) instead.

---

## 20. Batch API

Send multiple statements in one round trip as an implicit transaction. Supported by D1, libSQL (Turso), and Neon HTTP.

```ts
const results = await db.batch([
  db.insert(users).values({ name: 'Ana', email: 'a@x.com' }).returning({ id: users.id }),
  db.update(users).set({ name: 'Ben' }).where(eq(users.id, 2)),
  db.query.users.findMany({}),
  db.select().from(users).where(eq(users.id, 1)),
]);
// results is a typed tuple: [insertResult, updateResult, findManyResult, selectResult]
```

---

## 21. Views

```ts
import { pgView, pgMaterializedView } from 'drizzle-orm/pg-core';

// query-defined view
export const activeUsersView = pgView('active_users_view').as((qb) =>
  qb.select().from(users).where(eq(users.isActive, true)),
);

// raw SQL view with explicit columns
export const rawView = pgView('raw_view', {
  id: integer('id'),
  name: text('name'),
}).as(sql`select id, name from users`);

// existing view in the database (not managed by migrations)
export const legacyView = pgView('legacy_view', { id: integer('id') }).existing();

// materialized view
export const stats = pgMaterializedView('user_stats').as((qb) =>
  qb.select({ total: count() }).from(users),
);

// query a view like a table
const rows = await db.select().from(activeUsersView);

// refresh a materialized view
await db.refreshMaterializedView(stats);
await db.refreshMaterializedView(stats).concurrently();
```

---

## 22. Type Inference

```ts
import { InferSelectModel, InferInsertModel } from 'drizzle-orm';

// two equivalent styles
type User = typeof users.$inferSelect;       // row returned by select
type NewUser = typeof users.$inferInsert;    // data accepted by insert

type User2 = InferSelectModel<typeof users>;
type NewUser2 = InferInsertModel<typeof users>;

// use in functions
async function createUser(data: NewUser): Promise<User> {
  const [row] = await db.insert(users).values(data).returning();
  return row;
}

// infer result type of any query
const query = db.select({ id: users.id }).from(users);
type QueryResult = Awaited<ReturnType<typeof query.execute>>;
```

---

## 23. Validation with drizzle-zod

```bash
npm i drizzle-zod zod
```

```ts
import { createInsertSchema, createSelectSchema, createUpdateSchema } from 'drizzle-zod';
import { z } from 'zod';

export const insertUserSchema = createInsertSchema(users);
export const selectUserSchema = createSelectSchema(users);
export const updateUserSchema = createUpdateSchema(users);

// refine fields
const insertUser = createInsertSchema(users, {
  email: (schema) => schema.email(),
  name: (schema) => schema.min(2).max(50),
});

// use it
const parsed = insertUser.parse({ name: 'Ana', email: 'ana@example.com' });
await db.insert(users).values(parsed);

type InsertUser = z.infer<typeof insertUser>;
```

Similar packages exist for Valibot, TypeBox, and ArkType (check the docs for current package names).

---

## 24. Seeding with drizzle-seed

```bash
npm i -D drizzle-seed
```

```ts
// src/db/seed.ts
import 'dotenv/config';
import { drizzle } from 'drizzle-orm/node-postgres';
import { seed, reset } from 'drizzle-seed';
import * as schema from './schema';

const db = drizzle(process.env.DATABASE_URL!);

async function main() {
  await reset(db, schema);                       // optional: clear tables first

  await seed(db, schema, { count: 100, seed: 42 })   // fixed seed = repeatable data
    .refine((f) => ({
      users: {
        count: 50,
        columns: {
          name: f.fullName(),
          email: f.email(),
          age: f.int({ minValue: 18, maxValue: 65 }),
        },
        with: {
          posts: 3,                              // 3 posts per user
        },
      },
    }));
}

main();
```

Run with `npx tsx src/db/seed.ts`.

---

## 25. Row-Level Security (PostgreSQL)

```ts
import { pgTable, pgPolicy, pgRole, integer, text } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const appUser = pgRole('app_user').existing();

export const notes = pgTable(
  'notes',
  {
    id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
    ownerId: text('owner_id').notNull(),
    body: text('body'),
  },
  (t) => [
    pgPolicy('owner_can_read', {
      for: 'select',
      to: appUser,
      using: sql`${t.ownerId} = current_setting('app.user_id')`,
    }),
  ],
).enableRLS();
```

To manage roles through Drizzle Kit, set `entities: { roles: true }` in the config file.

---

## 26. Logging and Debugging

```ts
// simple
const db = drizzle(url, { logger: true });

// custom logger
import { DefaultLogger, LogWriter } from 'drizzle-orm/logger';

class MyWriter implements LogWriter {
  write(message: string) { console.log('[SQL]', message); }
}
const db2 = drizzle(url, { logger: new DefaultLogger({ writer: new MyWriter() }) });

// print SQL without running it
const q = db.select().from(users).where(eq(users.id, 1));
console.log(q.toSQL());   // { sql: 'select ... where "users"."id" = $1', params: [1] }
```

---

## 27. Testing

Options:

- **PGlite**: in-memory PostgreSQL, fast, no Docker needed.
- **SQLite in-memory** (`:memory:`): for SQLite projects.
- **Testcontainers**: real PostgreSQL or MySQL in Docker.

PGlite example:

```ts
import { PGlite } from '@electric-sql/pglite';
import { drizzle } from 'drizzle-orm/pglite';
import { migrate } from 'drizzle-orm/pglite/migrator';
import * as schema from './schema';

const client = new PGlite();
export const testDb = drizzle({ client, schema });

beforeAll(async () => {
  await migrate(testDb, { migrationsFolder: './drizzle' });
});

afterEach(async () => {
  await testDb.delete(schema.users);
});
```

You can also push the schema in tests using `drizzle-kit/api` (`pushSchema`) instead of running migration files.

---

## 28. Best Practices

- Use **generate + migrate** for staging and production. Use **push** only for prototypes and local work.
- Review every generated SQL file before applying it, and commit the `drizzle` folder to Git.
- Always add `.where()` to `update` and `delete`.
- Add indexes for foreign keys and for columns you filter or sort by.
- Prefer `generatedAlwaysAsIdentity()` over `serial` on new PostgreSQL tables.
- Use `timestamp(..., { withTimezone: true })` for time data.
- Use transactions for multi-step writes.
- Use `$inferSelect` and `$inferInsert` types instead of writing your own interfaces.
- Validate external input (API bodies) with drizzle-zod or another validator before inserting.
- Use prepared statements for hot paths.
- Use keyset pagination for large tables.
- Keep `casing` the same in `drizzle()` and in the `drizzle.config.ts` file.
- Reuse one `db` instance (singleton). In serverless, use a driver or pooler made for it (Neon, Supabase pooler, Hyperdrive).
- Keep secrets in `.env` file and never commit it.

---

## 29. Command Cheat Sheet

| Task | Command |
| --- | --- |
| Install | `npm i drizzle-orm && npm i -D drizzle-kit` |
| Generate migration | `npx drizzle-kit generate` |
| Generate named migration | `npx drizzle-kit generate --name=my_change` |
| Generate empty custom migration | `npx drizzle-kit generate --custom --name=my_data_fix` |
| Apply migrations | `npx drizzle-kit migrate` |
| Push schema (no files) | `npx drizzle-kit push` |
| Push with confirmation / verbose | `npx drizzle-kit push --strict --verbose` |
| Push, auto-approve | `npx drizzle-kit push --force` |
| Introspect database | `npx drizzle-kit pull` |
| Check migration conflicts | `npx drizzle-kit check` |
| Upgrade snapshots | `npx drizzle-kit up` |
| Export SQL | `npx drizzle-kit export` |
| Open Studio | `npx drizzle-kit studio` |
| Studio on custom port | `npx drizzle-kit studio --port=4000` |
| Use another config file | `npx drizzle-kit <command> --config=other.config.ts` |
| Run seed script | `npx tsx src/db/seed.ts` |

Query cheat sheet:

| Task | Code |
| --- | --- |
| Select all | `db.select().from(t)` |
| Insert | `db.insert(t).values({...})` |
| Insert + return | `.returning()` |
| Upsert | `.onConflictDoUpdate({ target, set })` |
| Update | `db.update(t).set({...}).where(...)` |
| Delete | `db.delete(t).where(...)` |
| Count | `db.$count(t, where?)` |
| Join | `.innerJoin(t2, eq(a, b))` |
| Nested data | `db.query.t.findMany({ with: {...} })` |
| Transaction | `db.transaction(async (tx) => {...})` |
| Raw SQL | `db.execute(sql\`...\`)` |
| Show SQL | `query.toSQL()` |

---

## 30. Troubleshooting

| Problem | Likely cause and fix |
| --- | --- |
| `db.query.users is undefined` | Pass `schema` (and relations) to `drizzle()`. |
| Relations not working | Check the `relations()` definitions on **both** sides and export them from the schema file. |
| `drizzle-kit` cannot find the schema | Check `schema` path in the `drizzle.config.ts` file. Paths are relative to the project root. |
| `DATABASE_URL` is undefined in config | Add `import 'dotenv/config'` at the top of the `drizzle.config.ts` file. |
| Column names do not match | `casing` setting differs between `drizzle()` and the config file. |
| `numeric` / `avg` returns a string | This is normal for exact numbers. Use `.mapWith(Number)` or cast in SQL. |
| `bigint` precision problems | Use `bigint({ mode: 'bigint' })` or `mode: 'number'` only for safe ranges. |
| Cannot use `tx` in Neon HTTP or D1 | These drivers have no interactive transactions. Use `db.batch([...])`. |
| Self-reference type error | Add `(): AnyPgColumn =>` return type in `.references()`. |
| Migration conflict after merging branches | Run `drizzle-kit check`, then regenerate or fix the migration order. |
| Push wants to drop a column or table | Read the warning. Rename operations may be seen as drop + create. |
| ESM / import errors in scripts | Run scripts with `tsx`, and check `"type": "module"` and `tsconfig` settings. |
| Update changed every row | The `.where()` was missing. Always include it. |

---

## Useful Links

- Official docs: <https://orm.drizzle.team>
- Drizzle Kit docs: <https://orm.drizzle.team/docs/kit-overview>
- GitHub: <https://github.com/drizzle-team/drizzle-orm>
- Drizzle Studio: <https://orm.drizzle.team/drizzle-studio/overview>
