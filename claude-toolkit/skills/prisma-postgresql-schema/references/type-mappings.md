# Type Mappings — PostgreSQL ↔ Prisma ↔ TypeScript ↔ Zod

## PostgreSQL → Prisma

| PostgreSQL Type | Prisma Type | Notes |
|---|---|---|
| `UUID` | `String @db.Uuid` | Always use `@db.Uuid` for clarity |
| `TEXT` | `String` | Default string mapping |
| `VARCHAR(n)` | `String @db.VarChar(n)` | Use when length matters |
| `CHAR(n)` | `String @db.Char(n)` | Fixed-length strings |
| `INT` / `INTEGER` | `Int` | |
| `BIGINT` | `BigInt` | |
| `SMALLINT` | `Int @db.SmallInt` | |
| `SERIAL` | `Int @default(autoincrement())` | |
| `BIGSERIAL` | `BigInt @default(autoincrement())` | |
| `BOOLEAN` | `Boolean` | |
| `FLOAT` / `REAL` | `Float` | |
| `DOUBLE PRECISION` | `Float @db.DoublePrecision` | |
| `DECIMAL(p,s)` | `Decimal @db.Decimal(p,s)` | Use for money/precision |
| `NUMERIC` | `Decimal` | |
| `TIMESTAMPTZ` | `DateTime @db.Timestamptz` | Preferred for timestamps |
| `TIMESTAMP` | `DateTime` | No timezone |
| `DATE` | `DateTime @db.Date` | Date only |
| `TIME` | `DateTime @db.Time` | Time only |
| `JSON` | `Json` | |
| `JSONB` | `Json @db.JsonB` | Preferred over JSON in PG |
| `BYTEA` | `Bytes` | |
| `ENUM` | `enum` (Prisma enum) | Define as top-level enum |
| `ARRAY` | Not natively supported | Use Json or relation |

---

## Prisma → TypeScript

| Prisma Type | TypeScript Type |
|---|---|
| `String` | `string` |
| `Int` | `number` |
| `BigInt` | `bigint` |
| `Float` | `number` |
| `Decimal` | `Prisma.Decimal` |
| `Boolean` | `boolean` |
| `DateTime` | `Date` |
| `Json` | `Prisma.JsonValue` |
| `Bytes` | `Buffer` |
| `Enum` | TypeScript `enum` or union type |
| Optional (`?`) | `T \| null` |

---

## Prisma → Zod

| Prisma Type | Zod Schema |
|---|---|
| `String` | `z.string()` |
| `String @db.Uuid` | `z.string().uuid()` |
| `String` (email) | `z.string().email()` |
| `Int` | `z.number().int()` |
| `BigInt` | `z.bigint()` |
| `Float` | `z.number()` |
| `Decimal` | `z.number()` or `z.string()` (if precision needed) |
| `Boolean` | `z.boolean()` |
| `DateTime` | `z.date()` or `z.string().datetime()` |
| `Json` | `z.record(z.unknown())` or `z.any()` |
| `Bytes` | `z.instanceof(Buffer)` |
| Enum (e.g. `UserRole`) | `z.enum(['ADMIN', 'USER'])` |
| Optional (`?`) | `.nullable()` or `.optional()` |
| Array (`[]`) | `z.array(...)` |

---

## Notes

- For monetary values, always use `Decimal @db.Decimal(18, 8)` or `Decimal @db.Decimal(10, 2)` — never `Float`
- For wallet addresses: `String @db.VarChar(66)` (covers EVM 0x + 64 chars and XRPL addresses)
- For hashes (tx hashes, etc.): `String @db.VarChar(66)` for EVM, `String @db.VarChar(100)` for XRPL
- Always prefer `@db.Uuid` over plain `String` for UUID fields for DB-level type enforcement