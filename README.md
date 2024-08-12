# Supabase + Prisma: useful notes for questions and blockers I had during the setup and how to solve them.

## Setup Supabase and Prisma

> complete guide for Supabase + Prisma here:
> [supabase.com/partners/integrations/prisma](https://supabase.com/partners/integrations/prisma)

Following the guide, the first thing I noticed was that supabase requires two ✌️ URLs: `DATABASE_URL` and `DIRECT_URL`.

**The steps are explained in the guide but in simple terms, how can we recognize these urls?**

**DATABASE_URL**

It's the `Transaction` string with port **6543** and we need to append `?pgbouncer=true&connection_limit=1`. for example:

_postgres://[db-user].[project-ref]:[db-password]@aws-0-[aws-region].pooler.supabase.com:6543/[db-name]?pgbouncer=true&connection_limit=1_

**DIRECT_URL**

It's the `Session` string with port **5432**.

_postgres://[db-user].[project-ref]:[db-password]@aws-0-[aws-region].pooler.supabase.com:5432/[db-name]_

Now we can add them in the `schema.prisma`:

```prisma
generator client {
  provider        = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}
```

## Connect Supabase Auth with Prisma using multi-schema

> complete guide for Supabase + Prisma here:
> [supabase.com/partners/integrations/prisma](https://supabase.com/partners/integrations/prisma)

To add relations between public and auth schemas from supabase we need to include these schemas in prisma.

1. Enable multi-schemas

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["multiSchema"]
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
  schemas   = ["auth", "public"]
}

model Profile {
  id           String    @id @db.Uuid
  email        String    @unique @db.VarChar
  createdAt    DateTime? @default(now())
  updatedAt    DateTime? @updatedAt
  users        users     @relation(fields: [id], references: [id], onDelete: Cascade, onUpdate: NoAction)
  // more data

  @@schema("public")
}
```

Don't forget to include the `@@schema` for each table.

In this example a relation between _public_ `Profile.email` and _auth_ `users.email` is not required because this value is `@unique` and `auth.users` validates the email is not already registered.

2. Pull auth schema tables.

```bash
npx prisma db pull
```

This gets all the tables from the selected schemas `auth` and `public` in supabase, public is empty so there won't be changes and auth tables will be created automatically, allowing us to add relations.

## Setup supabase to add a row in `public.Profile` table at the same time a new user is registered in `auth.users`.

> reference: [@supabase issue #563 comment #772954907](https://github.com/supabase/supabase/issues/563#issuecomment-772954907)

This allows us to have extra user data, for example: `nickname`, `bio`, `age`, `whatsapp`, etc. To do this we will use triggers to create a new row in `public.Profile` with the same `id` and `email` from _auth_ `users.id` and `users.email` when a new user is registered.

Using the supabase `SQL editor`:

1. Create a public.users table:

> In my case using Prisma, I added the table to the schema.

with schema.prisma:

```prisma
model Profile {
  id           String    @id @db.Uuid
  email        String    @unique @db.VarChar
  createdAt    DateTime? @default(now())
  updatedAt    DateTime? @updatedAt
  // extra fields here
  bio          String?
  age          String?
  whatsapp     String?
  subscription String?   @default("free")
  role         String?   @default("user")
  users        users     @relation(fields: [id], references: [id], onDelete: Cascade, onUpdate: NoAction)

  @@index([email])
  @@index([whatsapp])
  @@schema("public")
}
```

with postgresql:

```sql
create table Profile (
  id uuid references auth.users not null primary key,
  email text
  --- extra fields
);
```

2. Create a trigger:

```sql
create or replace function public.handle_new_user()
returns trigger as $$
begin
  insert into public."Profile" (id, email)
  values (new.id, new.email);
  return new;
end;
$$ language plpgsql security definer;
```

3. Trigger the function on invite:

```sql
-- trigger the function every time a user is created
create or replace trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();
```

4. Invite a user via the UI

5. Validate a the new user is added in `auth.users` and `public.Profile`

## Update Signup Email template for NextJS SSR

Update href from `Confirm signup` template

```html
<h2>Confirm your signup</h2>

<p>Follow this link to confirm your user:</p>
<p><a href="{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=signup">Confirm your mail</a></p>
```
