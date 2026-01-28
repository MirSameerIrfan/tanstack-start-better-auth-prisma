# TanStack Start + Better Auth + Prisma

A full-stack authentication setup combining TanStack Start, Better Auth, and Prisma with PostgreSQL.

## Project Setup

This project demonstrates a complete authentication flow using:
- **TanStack Start**: Full-stack framework
- **Better Auth**: Authentication library
- **Prisma**: ORM with PostgreSQL adapter

---

## Fix: Better Auth CLI + Prisma "MODULE_NOT_FOUND" in TanStack Start

### Problem Summary

When setting up Better Auth with Prisma in TanStack Start, you may encounter a module resolution error when running the Better Auth CLI `generate` command:

```
[@better-auth]: Couldn't read your auth config. Error: Cannot find module 'generated/prisma/client'
Require stack:
- /workspaces/codespaces-blank/src/utils/auth.ts
  at Module._resolveFilename (node:internal/modules/cjs/loader:1421:15)
  at require.resolve (node:internal/modules/cjs/loader:163:19)
  at jitiResolve (/home/codespace/.npm/_npx/167ca1f1f6d365e6/node_modules/.jiti/dist/jiti.cjs:1:148703)
  ...
```

![Better Auth CLI error](docs/images/better-auth-prisma-fix/01-error.png)

*Screenshot 1: Error output showing MODULE_NOT_FOUND when running Better Auth CLI generate*

### Root Cause

Better Auth CLI executes your config file (`src/utils/auth.ts`) in a Node.js context at build time. It needs to resolve the `PrismaClient` import at this point. This fails when:

1. **Import path is incorrect**: Using `import { PrismaClient } from "generated/prisma/client"` (bare path) instead of a relative path like `import { PrismaClient } from "../../generated/prisma/client"`
2. **Prisma client not yet generated**: The Prisma schema needs to be compiled first
3. **Output path mismatch**: Your Prisma `generator` block specifies an output path that doesn't match your import

When Prisma generates the client to a custom folder (e.g., `generated/prisma/`), the import in your config must use a relative path from the config file location, not a bare module import.

![Auth config with problematic import](docs/images/better-auth-prisma-fix/02-config-before.png)

*Screenshot 2: Original `auth.ts` with incorrect import path*

### The Working Fix (Step-by-Step)

#### Step 1: Verify Dependencies

Ensure you have the required packages installed:

```bash
npm install @prisma/client prisma better-auth @better-auth/cli @prisma/adapter-pg
# or
pnpm install @prisma/client prisma better-auth @better-auth/cli @prisma/adapter-pg
```

#### Step 2: Configure Prisma Generator Output Path

In `prisma/schema.prisma`, ensure your generator block specifies the custom output path:

```prisma
generator client {
  provider = "prisma-client-js"
  output   = "../generated/prisma"
}
```

This tells Prisma to generate the client to `generated/prisma/` instead of the default `node_modules/.prisma/client`.

![Prisma schema.prisma configuration](docs/images/better-auth-prisma-fix/03-prisma-config.png)

*Screenshot 3: Prisma generator output configuration in schema.prisma*

#### Step 3: Generate the Prisma Client

From the **root of your project**, run:

```bash
npx prisma generate
```

This creates the client at `generated/prisma/client.ts`.

#### Step 4: Update Auth Config with Correct Import Path

Update `src/utils/auth.ts` to use a **relative path** from the config file location:

**INCORRECT:**
```typescript
import { PrismaClient } from "generated/prisma/client"
```

**CORRECT:**
```typescript
import { PrismaClient } from "../../generated/prisma/client"
```

Here's the complete working config:

```typescript
// src/utils/auth.ts

import { betterAuth } from "better-auth"
import { prismaAdapter } from "better-auth/adapters/prisma"
import { PrismaClient } from "../../generated/prisma/client"
import { PrismaPg } from "@prisma/adapter-pg"

const adapter = new PrismaPg({
  connectionString: process.env.DATABASE_URL!,
})

const prisma = new PrismaClient({ adapter })

export const auth = betterAuth({
  database: prismaAdapter(prisma, {
    provider: "postgresql"
  })
})
```

**Key points:**
- Use `../../` relative path from `src/utils/auth.ts` to `generated/prisma/client`
- Initialize `PrismaClient` with the PostgreSQL adapter from `@prisma/adapter-pg`
- Pass the configured Prisma instance to `prismaAdapter()`

![Working auth.ts configuration](docs/images/better-auth-prisma-fix/04-config-fixed.png)

*Screenshot 4: Fixed `auth.ts` with correct relative import and adapter setup*

#### Step 5: Generate Better Auth Schema

Now run the Better Auth CLI generate command:

```bash
npm @better-auth/cli generate
# or
pnpm @better-auth/cli generate
```

This command will now successfully:
1. Load your auth config from `src/utils/auth.ts`
2. Resolve the `PrismaClient` import using the correct relative path
3. Generate Better Auth schema files (typically in `generated/auth/`)

![Successful Better Auth CLI execution](docs/images/better-auth-prisma-fix/05-success.png)

*Screenshot 5: Better Auth CLI successfully generates after fix (command runs without MODULE_NOT_FOUND error)*

### Gotchas & Troubleshooting

- **Prisma client not found**: If you still get an error after Step 3, ensure `generated/prisma/client.ts` actually exists: `ls generated/prisma/`
- **Running from wrong directory**: Always run `npx prisma generate` from the project root (where `prisma/schema.prisma` is located)
- **Stale node_modules**: If the Prisma client was previously generated to `node_modules/.prisma`, you may need to clear it: `rm -rf node_modules/.prisma`
- **Dev server still running**: Restart your dev server after running `prisma generate` to avoid import caching issues
- **Monorepo setups**: If using a monorepo, ensure `@prisma/client` is installed in the root `node_modules` and run Prisma commands from the root

### Verification

Confirm the fix is working:

```bash
# 1. Verify the generated Prisma client exists
ls generated/prisma/client.ts

# 2. Verify Better Auth CLI command succeeds
npm @better-auth/cli generate

# 3. Start your dev server and test
npm run dev
# Try accessing the sign-in route to confirm auth is working
```

---

## Project Structure

```
.
├── src/
│   ├── utils/
│   │   └── auth.ts              # Better Auth + Prisma config
│   ├── routes/
│   │   └── api/auth/$.ts        # Auth API handler
│   └── ...
├── prisma/
│   └── schema.prisma            # Database schema
├── generated/
│   ├── prisma/                  # Generated Prisma client
│   └── auth/                    # Generated Better Auth files
├── docs/
│   └── images/
└── ...
```

## Getting Started

1. Set up environment variables (`.env`):
   ```
   DATABASE_URL="postgresql://user:password@localhost:5432/dbname"
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Generate Prisma client:
   ```bash
   npx prisma generate
   ```

4. Generate Better Auth:
   ```bash
   npm @better-auth/cli generate
   ```

5. Run the dev server:
   ```bash
   npm run dev
   ```

