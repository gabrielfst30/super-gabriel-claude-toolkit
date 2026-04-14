# Common Prisma Patterns

## Soft Delete

```prisma
model Post {
  id        String    @id @default(uuid()) @db.Uuid
  deletedAt DateTime?
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt

  @@index([deletedAt]) // speeds up WHERE deletedAt IS NULL
}
```

Query pattern:
```ts
// Find active records
prisma.post.findMany({ where: { deletedAt: null } })

// Soft delete
prisma.post.update({ where: { id }, data: { deletedAt: new Date() } })
```

---

## Multi-tenancy (organization scoping)

```prisma
model Organization {
  id    String  @id @default(uuid()) @db.Uuid
  name  String
  users User[]
  posts Post[]
}

model User {
  id             String       @id @default(uuid()) @db.Uuid
  organizationId String       @db.Uuid
  organization   Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)

  @@index([organizationId])
}

model Post {
  id             String       @id @default(uuid()) @db.Uuid
  organizationId String       @db.Uuid
  organization   Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)

  @@index([organizationId])
}
```

---

## Audit Log

```prisma
model AuditLog {
  id         String   @id @default(uuid()) @db.Uuid
  action     String
  entityType String
  entityId   String   @db.Uuid
  actorId    String?  @db.Uuid
  before     Json?
  after      Json?
  createdAt  DateTime @default(now())

  @@index([entityType, entityId])
  @@index([actorId])
  @@index([createdAt])
}
```

---

## Pagination (cursor-based)

```ts
// First page
const page1 = await prisma.post.findMany({
  take: 20,
  orderBy: { createdAt: 'desc' },
})

// Next page — use last item's id as cursor
const page2 = await prisma.post.findMany({
  take: 20,
  skip: 1, // skip the cursor itself
  cursor: { id: lastItemId },
  orderBy: { createdAt: 'desc' },
})
```

Schema requirement: cursor field must be `@unique` (id works fine).

---

## Role-based access (RBAC)

```prisma
enum UserRole {
  SUPER_ADMIN
  ADMIN
  MODERATOR
  USER
}

model User {
  id   String   @id @default(uuid()) @db.Uuid
  role UserRole @default(USER)
}
```

For per-organization roles, use an explicit join table:

```prisma
model OrganizationMember {
  userId         String       @db.Uuid
  organizationId String       @db.Uuid
  role           UserRole     @default(USER)
  user           User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  organization   Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)

  @@id([userId, organizationId])
  @@index([organizationId])
}
```

---

## Wallet / Web3 patterns

```prisma
model Wallet {
  id        String   @id @default(uuid()) @db.Uuid
  address   String   @unique @db.VarChar(66) // EVM: 42 chars, XRPL: up to 46 chars
  network   Network
  userId    String   @db.Uuid
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())

  @@index([userId])
  @@index([network])
}

enum Network {
  EVM
  XRPL
}
```

---

## Timestamps-only base (abstract pattern)

Prisma doesn't support abstract models, but use a comment convention:

```prisma
// Base fields — include in every model:
// id        String   @id @default(uuid()) @db.Uuid
// createdAt DateTime @default(now())
// updatedAt DateTime @updatedAt
```
