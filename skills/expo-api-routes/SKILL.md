---
name: expo-api-routes
description: Use when building server-side API endpoints in Expo apps, handling backend logic, protecting credentials, connecting to databases, or deploying API routes with EAS Hosting
---

# Expo API Routes

## Overview

Expo API Routes let you write server-side endpoints alongside your React Native app code. Routes use the `+api.ts` file suffix and run on the server, never shipped to the client bundle. This keeps credentials secure and enables direct database access.

**Core principle:** API routes are server-only code collocated with your app. They deploy to EAS Hosting (Cloudflare Workers) and handle standard HTTP request/response patterns.

## When to Use

- Protecting API keys, database credentials, or third-party secrets from the client
- Building webhooks (Stripe, payment processors, push notification services)
- Performing server-side data validation or transformation
- Connecting directly to databases (Neon, Supabase, PlanetScale, Turso)
- Proxying requests to external APIs with rate limiting
- Server-side rendering or data aggregation

## File Convention

Any file with the `+api.ts` suffix in the `app/` directory becomes a server endpoint:

```
app/
  api/
    hello+api.ts           # GET /api/hello
    users+api.ts           # /api/users (multiple methods)
    users/[id]+api.ts      # /api/users/123 (dynamic route)
    webhooks/stripe+api.ts # POST /api/webhooks/stripe
```

## Request Handling

### Basic Endpoint

```tsx
// app/api/hello+api.ts
export function GET(request: Request) {
  return Response.json({ message: 'Hello from the server!' });
}
```

### Multiple HTTP Methods

```tsx
// app/api/users+api.ts
export async function GET(request: Request) {
  const users = await db.query('SELECT * FROM users');
  return Response.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  const { name, email } = body;

  if (!name || !email) {
    return Response.json(
      { error: 'Name and email required' },
      { status: 400 }
    );
  }

  const user = await db.query(
    'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
    [name, email]
  );

  return Response.json(user, { status: 201 });
}

export async function DELETE(request: Request) {
  // Handle deletion
  return new Response(null, { status: 204 });
}
```

### Dynamic Routes

```tsx
// app/api/users/[id]+api.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const user = await db.query('SELECT * FROM users WHERE id = $1', [params.id]);

  if (!user) {
    return Response.json({ error: 'User not found' }, { status: 404 });
  }

  return Response.json(user);
}
```

### Query Parameters

```tsx
// app/api/search+api.ts
export async function GET(request: Request) {
  const url = new URL(request.url);
  const query = url.searchParams.get('q') ?? '';
  const page = parseInt(url.searchParams.get('page') ?? '1', 10);
  const limit = parseInt(url.searchParams.get('limit') ?? '20', 10);

  const results = await db.query(
    'SELECT * FROM items WHERE name ILIKE $1 LIMIT $2 OFFSET $3',
    [`%${query}%`, limit, (page - 1) * limit]
  );

  return Response.json({ results, page, limit });
}
```

## Server-Side Credential Protection

Environment variables in `+api.ts` files stay on the server:

```tsx
// app/api/payment+api.ts
export async function POST(request: Request) {
  // These never reach the client bundle
  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

  const body = await request.json();
  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    line_items: body.items,
    mode: 'payment',
    success_url: `${process.env.APP_URL}/success`,
    cancel_url: `${process.env.APP_URL}/cancel`,
  });

  return Response.json({ sessionId: session.id });
}
```

Set environment variables in EAS:

```bash
eas env:create --name STRIPE_SECRET_KEY --value sk_live_... --environment production
eas env:create --name DATABASE_URL --value postgresql://... --environment production
```

## Database Connections

### Neon (Serverless Postgres)

```tsx
// lib/db.ts
import { neon } from '@neondatabase/serverless';

export const sql = neon(process.env.DATABASE_URL!);

// app/api/items+api.ts
import { sql } from '../../lib/db';

export async function GET() {
  const items = await sql`SELECT * FROM items ORDER BY created_at DESC`;
  return Response.json(items);
}
```

### Supabase

```tsx
// lib/supabase-server.ts
import { createClient } from '@supabase/supabase-js';

export const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY! // Service role for server-side
);

// app/api/posts+api.ts
import { supabase } from '../../lib/supabase-server';

export async function GET() {
  const { data, error } = await supabase
    .from('posts')
    .select('*')
    .order('created_at', { ascending: false });

  if (error) {
    return Response.json({ error: error.message }, { status: 500 });
  }

  return Response.json(data);
}
```

### Turso (SQLite Edge)

```tsx
// lib/turso.ts
import { createClient } from '@libsql/client';

export const turso = createClient({
  url: process.env.TURSO_DATABASE_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN!,
});

// app/api/notes+api.ts
import { turso } from '../../lib/turso';

export async function GET() {
  const result = await turso.execute('SELECT * FROM notes');
  return Response.json(result.rows);
}
```

## Webhooks

```tsx
// app/api/webhooks/stripe+api.ts
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(request: Request) {
  const body = await request.text();
  const signature = request.headers.get('stripe-signature')!;

  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    return Response.json({ error: 'Invalid signature' }, { status: 400 });
  }

  switch (event.type) {
    case 'checkout.session.completed':
      const session = event.data.object;
      await fulfillOrder(session);
      break;
    case 'payment_intent.payment_failed':
      await handleFailedPayment(event.data.object);
      break;
  }

  return Response.json({ received: true });
}
```

## Rate Limiting

Simple in-memory rate limiter (for single-instance deployments):

```tsx
// lib/rate-limit.ts
const requests = new Map<string, { count: number; resetTime: number }>();

export function rateLimit(
  ip: string,
  limit: number = 60,
  windowMs: number = 60_000
): boolean {
  const now = Date.now();
  const record = requests.get(ip);

  if (!record || now > record.resetTime) {
    requests.set(ip, { count: 1, resetTime: now + windowMs });
    return true;
  }

  if (record.count >= limit) {
    return false;
  }

  record.count++;
  return true;
}

// Usage in API route
export async function GET(request: Request) {
  const ip = request.headers.get('x-forwarded-for') ?? 'unknown';

  if (!rateLimit(ip)) {
    return Response.json(
      { error: 'Too many requests' },
      { status: 429, headers: { 'Retry-After': '60' } }
    );
  }

  // Handle request...
}
```

## EAS Hosting Deployment

API routes deploy to EAS Hosting, powered by Cloudflare Workers:

```bash
# Install EAS CLI
npm install -g eas-cli

# Login
eas login

# Deploy
npx expo export --platform web
eas hosting:deploy ./dist

# Or link to GitHub for automatic deploys
eas hosting:link
```

### Limitations on Cloudflare Workers

- No filesystem access (`fs` module)
- No long-running processes (30s timeout on free tier)
- No Node.js native modules (use Web APIs)
- Limited memory (128MB)
- Use `fetch` instead of `http`/`https` modules

## Calling API Routes from the Client

```tsx
// From your React Native app
async function fetchUsers() {
  // In development, use the local dev server
  const response = await fetch('/api/users');
  const data = await response.json();
  return data;
}

// With full URL for production
const API_URL = process.env.EXPO_PUBLIC_API_URL ?? '';

async function createUser(name: string, email: string) {
  const response = await fetch(`${API_URL}/api/users`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name, email }),
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }

  return response.json();
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Importing server code in client components | API route files (`+api.ts`) are server-only; never import from them in app components |
| Using `EXPO_PUBLIC_` prefix for secrets | Only `process.env.SECRET` (no prefix) stays server-side |
| Using Node.js `fs` module | EAS Hosting runs on Cloudflare Workers; use KV or D1 for storage |
| Not validating request body | Always validate and sanitize input before database queries |
| Returning raw database errors | Catch errors and return sanitized messages to clients |
| Missing CORS headers | Add appropriate headers for cross-origin requests if needed |

## Quick Reference

| Task | Pattern |
|------|---------|
| Create endpoint | `app/api/name+api.ts` |
| Export handler | `export function GET(request: Request)` |
| JSON response | `Response.json(data)` |
| Error response | `Response.json({ error }, { status: 400 })` |
| Read body | `await request.json()` |
| Read query params | `new URL(request.url).searchParams` |
| Dynamic route param | Second arg: `{ params: { id: string } }` |
| Set env var | `eas env:create --name KEY --value VAL` |
| Deploy | `npx expo export --platform web && eas hosting:deploy ./dist` |
