# Prisma Relations — Full Patterns

## 1-to-1

```prisma
model User {
  id      String   @id @default(uuid()) @db.Uuid
  profile Profile?
}

model Profile {
  id     String @id @default(uuid()) @db.Uuid
  userId String @unique @db.Uuid
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

Notes:
- The FK lives on the "child" side (Profile)
- `@unique` on the FK enforces the 1-1 constraint
- `Profile?` on User means profile is optional

---

## 1-to-many

```prisma
model User {
  id    String @id @default(uuid()) @db.Uuid
  posts Post[]
}

model Post {
  id     String @id @default(uuid()) @db.Uuid
  userId String @db.Uuid
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}
```

Notes:
- FK on the "many" side (Post)
- Always `@@index` the FK field
- Use `onDelete: Cascade` if child records should be removed with parent

---

## Many-to-many (implicit — Prisma managed)

```prisma
model Post {
  id   String @id @default(uuid()) @db.Uuid
  tags Tag[]
}

model Tag {
  id    String @id @default(uuid()) @db.Uuid
  posts Post[]
}
```

Notes:
- Prisma creates a hidden join table `_PostToTag`
- No control over extra fields on the join table
- Good for simple cases with no extra metadata

---

## Many-to-many (explicit join table — preferred)

```prisma
model Post {
  id   String      @id @default(uuid()) @db.Uuid
  tags PostToTag[]
}

model Tag {
  id    String      @id @default(uuid()) @db.Uuid
  posts PostToTag[]
}

model PostToTag {
  postId    String   @db.Uuid
  tagId     String   @db.Uuid
  assignedAt DateTime @default(now())
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag       Tag      @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([postId, tagId])
  @@index([tagId])
}
```

Notes:
- Use when the join itself has metadata (assignedAt, role, weight, etc.)
- Composite `@@id` avoids duplicate pairs
- Always index the second FK in the composite key

---

## Self-relation (tree / parent-child)

```prisma
model Category {
  id       String     @id @default(uuid()) @db.Uuid
  name     String
  parentId String?    @db.Uuid
  parent   Category?  @relation("CategoryTree", fields: [parentId], references: [id])
  children Category[] @relation("CategoryTree")

  @@index([parentId])
}
```

Notes:
- Named relation (`"CategoryTree"`) is required for self-relations
- `parentId` is optional (nullable) to allow root nodes

---

## Self-relation (many-to-many — followers)

```prisma
model User {
  id        String @id @default(uuid()) @db.Uuid
  following User[] @relation("UserFollows")
  followers User[] @relation("UserFollows")
}
```

---

## onDelete / onUpdate policies

| Policy | Behavior |
|---|---|
| `Cascade` | Delete/update child when parent is deleted/updated |
| `Restrict` | Prevent parent delete/update if children exist (error) |
| `SetNull` | Set FK to null on parent delete (field must be `?`) |
| `SetDefault` | Set FK to default value on parent delete |
| `NoAction` | Defer to DB default (similar to Restrict in PG) |

**Rules of thumb:**
- User → Profile: `Cascade` (profile meaningless without user)
- Order → OrderItem: `Cascade`
- Post → Category: `SetNull` or `Restrict` (don't delete posts when category is removed)
- User → AuditLog: `Restrict` or `SetNull` (preserve logs)

---

## Relation naming conventions

| Pattern | Example |
|---|---|
| FK field | `userId`, `postId`, `organizationId` |
| Relation field (single) | `user`, `post`, `organization` |
| Relation field (list) | `users`, `posts`, `organizations` |
| Named relation | `"CategoryTree"`, `"UserFollows"` |
| Join table | `UserToRole`, `PostToTag` |