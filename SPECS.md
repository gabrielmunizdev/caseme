# SPECS - Case-me

## STACK TECNOLÓGICA

### Frontend
- **Framework:** Next.js 14+ (App Router)
- **Linguagem:** TypeScript 5+
- **UI Library:** shadcn/ui + Radix UI
- **Styling:** Tailwind CSS 3.4+
- **State Management:** Zustand (client state), TanStack Query v5 (server state)
- **Forms:** React Hook Form + Zod
- **Animações:** Framer Motion
- **Ícones:** Lucide React

### Backend & Database
- **Database:** Supabase (PostgreSQL)
- **ORM:** Drizzle ORM
- **API:** Next.js Server Actions
- **Realtime:** Supabase Realtime (V2 - chat)
- **Storage:** Supabase Storage (portfólios, fotos casamento)

### Autenticação
- **Provider:** Clerk
- **Features:** Organizations (multi-tenant), OAuth (Google), Session Management
- **Sync:** Webhooks Clerk → Supabase (users + organizations)

### Email
- **Provider:** Resend
- **Templates:** React Email
- **Tipos:** Boas-vindas (noivo/fornecedor), novo lead, nova avaliação, comissão gerada, confirmação de pagamento

### Infraestrutura
- **Hosting:** Vercel (Edge Functions)
- **Repository:** GitHub
- **CI/CD:** GitHub Actions + Vercel
- **Monitoring:** Vercel Analytics + Sentry (opcional)

---

## ARQUITETURA DE DADOS

### 🏢 MULTI-TENANT: Row-Level Security via Supabase

**Tenant Context Flow:**
```
Request → Clerk Auth → Extract Org ID → Supabase Query com RLS
```

- Isolamento garantido no nível do banco
- RLS policies nativas do Supabase
- Escala bem até 10k+ organizações

---

## SCHEMA DO BANCO DE DADOS (SUPABASE)

### Convenções
- Todas as tabelas empresariais têm `organization_id`
- RLS policies filtram por `organization_id`
- Soft deletes: `deleted_at TIMESTAMP`
- Audit trail: `created_at`, `updated_at`
- UUIDs para IDs (`gen_random_uuid()`)

### Tabela: organizations
```sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clerk_org_id TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE organizations ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can view their organizations"
  ON organizations FOR SELECT
  USING (clerk_org_id = current_setting('app.current_org_id', true));
```

### Tabela: users
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clerk_user_id TEXT UNIQUE NOT NULL,
  email TEXT NOT NULL,
  name TEXT,
  role TEXT NOT NULL DEFAULT 'noivo', -- 'noivo', 'fornecedor', 'assessor', 'admin'
  avatar_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can view own profile"
  ON users FOR SELECT USING (clerk_user_id = current_setting('app.current_user_id', true));
CREATE POLICY "Users can update own profile"
  ON users FOR UPDATE USING (clerk_user_id = current_setting('app.current_user_id', true));
```

### Tabela: vendor_profiles
```sql
CREATE TABLE vendor_profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  user_id UUID NOT NULL REFERENCES users(id),
  category TEXT NOT NULL, -- 'fotografo', 'buffet', 'decoracao', 'dj', etc.
  description TEXT,
  city TEXT NOT NULL,
  state TEXT NOT NULL,
  price_range TEXT, -- 'economico', 'moderado', 'premium'
  avg_rating NUMERIC(3,2) DEFAULT 0,
  total_reviews INTEGER DEFAULT 0,
  is_verified BOOLEAN DEFAULT FALSE,
  plan TEXT NOT NULL DEFAULT 'free', -- 'free', 'pro', 'premium'
  portfolio_limit INTEGER DEFAULT 5,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_vendor_profiles_org ON vendor_profiles(organization_id);
CREATE INDEX idx_vendor_profiles_category ON vendor_profiles(category);
CREATE INDEX idx_vendor_profiles_city ON vendor_profiles(city);
CREATE INDEX idx_vendor_profiles_rating ON vendor_profiles(avg_rating DESC);

ALTER TABLE vendor_profiles ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Public read" ON vendor_profiles FOR SELECT USING (deleted_at IS NULL);
CREATE POLICY "Org members can update" ON vendor_profiles FOR UPDATE
  USING (organization_id IN (
    SELECT id FROM organizations WHERE clerk_org_id = current_setting('app.current_org_id', true)
  ));
```

### Tabela: couples
```sql
CREATE TABLE couples (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  partner1_name TEXT NOT NULL,
  partner2_name TEXT NOT NULL,
  wedding_date DATE,
  city TEXT,
  state TEXT,
  slug TEXT UNIQUE, -- para página pública
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE couples ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Owner access" ON couples FOR ALL
  USING (user_id IN (SELECT id FROM users WHERE clerk_user_id = current_setting('app.current_user_id', true)));
CREATE POLICY "Public read by slug" ON couples FOR SELECT USING (slug IS NOT NULL);
```

### Tabela: wedding_pages
```sql
CREATE TABLE wedding_pages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  couple_id UUID NOT NULL REFERENCES couples(id) ON DELETE CASCADE,
  title TEXT,
  story TEXT,
  cover_image_url TEXT,
  theme TEXT DEFAULT 'classic', -- 'classic', 'modern', 'romantic', 'garden'
  is_published BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE wedding_pages ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Owner access" ON wedding_pages FOR ALL
  USING (couple_id IN (SELECT id FROM couples WHERE user_id IN (
    SELECT id FROM users WHERE clerk_user_id = current_setting('app.current_user_id', true)
  )));
CREATE POLICY "Public read published" ON wedding_pages FOR SELECT USING (is_published = TRUE);
```

### Tabela: gift_lists
```sql
CREATE TABLE gift_lists (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  couple_id UUID NOT NULL REFERENCES couples(id) ON DELETE CASCADE,
  name TEXT NOT NULL DEFAULT 'Lista de Presentes',
  is_monetary BOOLEAN DEFAULT FALSE, -- lista convertível em dinheiro
  pix_key TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE gift_lists ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Owner manage" ON gift_lists FOR ALL
  USING (couple_id IN (SELECT id FROM couples WHERE user_id IN (
    SELECT id FROM users WHERE clerk_user_id = current_setting('app.current_user_id', true)
  )));
CREATE POLICY "Public read" ON gift_lists FOR SELECT USING (TRUE);
```

### Tabela: gift_items
```sql
CREATE TABLE gift_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  gift_list_id UUID NOT NULL REFERENCES gift_lists(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT,
  image_url TEXT,
  price NUMERIC(10,2),
  is_purchased BOOLEAN DEFAULT FALSE,
  purchased_by TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE gift_items ENABLE ROW LEVEL SECURITY;
CREATE POLICY "List owner manage" ON gift_items FOR ALL
  USING (gift_list_id IN (SELECT id FROM gift_lists WHERE couple_id IN (
    SELECT id FROM couples WHERE user_id IN (
      SELECT id FROM users WHERE clerk_user_id = current_setting('app.current_user_id', true)
  ))));
CREATE POLICY "Public read" ON gift_items FOR SELECT USING (TRUE);
CREATE POLICY "Public update purchased" ON gift_items FOR UPDATE USING (TRUE);
```

### Tabela: wishlists (favoritos)
```sql
CREATE TABLE wishlists (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  vendor_profile_id UUID NOT NULL REFERENCES vendor_profiles(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, vendor_profile_id)
);

ALTER TABLE wishlists ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Owner access" ON wishlists FOR ALL
  USING (user_id IN (SELECT id FROM users WHERE clerk_user_id = current_setting('app.current_user_id', true)));
```

### Tabela: guests (convidados + RSVP)
```sql
CREATE TABLE guests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  couple_id UUID NOT NULL REFERENCES couples(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  email TEXT,
  phone TEXT,
  status TEXT DEFAULT 'pending', -- 'pending', 'confirmed', 'declined'
  plus_one BOOLEAN DEFAULT FALSE,
  dietary_notes TEXT,
  confirmed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE guests ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Owner manage" ON guests FOR ALL
  USING (couple_id IN (SELECT id FROM couples WHERE user_id IN (
    SELECT id FROM users WHERE clerk_user_id = current_setting('app.current_user_id', true)
  )));
```

### Tabela: reviews
```sql
CREATE TABLE reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  vendor_profile_id UUID NOT NULL REFERENCES vendor_profiles(id),
  user_id UUID NOT NULL REFERENCES users(id),
  lead_id UUID REFERENCES leads(id), -- validação antifraude
  rating INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
  comment TEXT,
  is_moderated BOOLEAN DEFAULT FALSE,
  is_visible BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, vendor_profile_id)
);

ALTER TABLE reviews ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Public read visible" ON reviews FOR SELECT USING (is_visible = TRUE);
CREATE POLICY "Owner create" ON reviews FOR INSERT
  WITH CHECK (user_id IN (SELECT id FROM users WHERE clerk_user_id = current_setting('app.current_user_id', true)));
```

### Tabela: leads
```sql
CREATE TABLE leads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  vendor_profile_id UUID NOT NULL REFERENCES vendor_profiles(id),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  user_id UUID NOT NULL REFERENCES users(id), -- noivo que enviou
  message TEXT,
  status TEXT DEFAULT 'new', -- 'new', 'viewed', 'contacted', 'closed'
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE leads ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Vendor org read" ON leads FOR SELECT
  USING (organization_id IN (SELECT id FROM organizations WHERE clerk_org_id = current_setting('app.current_org_id', true)));
CREATE POLICY "Noivo create" ON leads FOR INSERT
  WITH CHECK (user_id IN (SELECT id FROM users WHERE clerk_user_id = current_setting('app.current_user_id', true)));
```

### Tabela: affiliate_links
```sql
CREATE TABLE affiliate_links (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  code TEXT UNIQUE NOT NULL, -- código de tracking
  total_referrals INTEGER DEFAULT 0,
  active_referrals INTEGER DEFAULT 0,
  badge TEXT DEFAULT 'bronze', -- 'bronze', 'prata', 'ouro', 'elite'
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE affiliate_links ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Owner access" ON affiliate_links FOR ALL
  USING (user_id IN (SELECT id FROM users WHERE clerk_user_id = current_setting('app.current_user_id', true)));
```

### Tabela: commissions
```sql
CREATE TABLE commissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  affiliate_link_id UUID NOT NULL REFERENCES affiliate_links(id),
  subscription_id UUID NOT NULL REFERENCES subscriptions(id),
  amount NUMERIC(10,2) NOT NULL,
  percentage NUMERIC(5,2) NOT NULL,
  status TEXT DEFAULT 'pending', -- 'pending', 'approved', 'paid'
  paid_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE commissions ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Affiliate read own" ON commissions FOR SELECT
  USING (affiliate_link_id IN (SELECT id FROM affiliate_links WHERE user_id IN (
    SELECT id FROM users WHERE clerk_user_id = current_setting('app.current_user_id', true)
  )));
```

### Tabela: plans
```sql
CREATE TABLE plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL, -- 'Free', 'Pro', 'Premium'
  slug TEXT UNIQUE NOT NULL,
  price NUMERIC(10,2) NOT NULL DEFAULT 0,
  features JSONB DEFAULT '[]',
  portfolio_limit INTEGER DEFAULT 5,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE plans ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Public read" ON plans FOR SELECT USING (is_active = TRUE);
```

### Tabela: subscriptions
```sql
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  plan_id UUID NOT NULL REFERENCES plans(id),
  affiliate_link_id UUID REFERENCES affiliate_links(id), -- tracking de afiliado
  status TEXT DEFAULT 'active', -- 'active', 'cancelled', 'expired'
  started_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ,
  cancelled_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Org access" ON subscriptions FOR SELECT
  USING (organization_id IN (SELECT id FROM organizations WHERE clerk_org_id = current_setting('app.current_org_id', true)));
```

### Tabela: rankings
```sql
CREATE TABLE rankings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  vendor_profile_id UUID UNIQUE NOT NULL REFERENCES vendor_profiles(id),
  category TEXT NOT NULL,
  city TEXT NOT NULL,
  position INTEGER NOT NULL,
  score NUMERIC(5,2) NOT NULL, -- média ponderada
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_rankings_category_city ON rankings(category, city, position);

ALTER TABLE rankings ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Public read" ON rankings FOR SELECT USING (TRUE);
```

### Tabela: badges
```sql
CREATE TABLE badges (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL, -- 'Bronze', 'Prata', 'Ouro', 'Elite'
  slug TEXT UNIQUE NOT NULL,
  min_referrals INTEGER NOT NULL,
  icon TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE badges ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Public read" ON badges FOR SELECT USING (TRUE);
```

---

## DRIZZLE ORM SCHEMA

```typescript
// lib/db/schema.ts
import { pgTable, text, timestamp, uuid, numeric, integer, boolean, jsonb, date, unique } from 'drizzle-orm/pg-core';

// --- Core ---
export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  clerkOrgId: text('clerk_org_id').unique().notNull(),
  name: text('name').notNull(),
  slug: text('slug').unique().notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  clerkUserId: text('clerk_user_id').unique().notNull(),
  email: text('email').notNull(),
  name: text('name'),
  role: text('role').notNull().default('noivo'),
  avatarUrl: text('avatar_url'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

// --- Fornecedores ---
export const vendorProfiles = pgTable('vendor_profiles', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  userId: uuid('user_id').notNull().references(() => users.id),
  category: text('category').notNull(),
  description: text('description'),
  city: text('city').notNull(),
  state: text('state').notNull(),
  priceRange: text('price_range'),
  avgRating: numeric('avg_rating', { precision: 3, scale: 2 }).default('0'),
  totalReviews: integer('total_reviews').default(0),
  isVerified: boolean('is_verified').default(false),
  plan: text('plan').notNull().default('free'),
  portfolioLimit: integer('portfolio_limit').default(5),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
  deletedAt: timestamp('deleted_at', { withTimezone: true }),
});

// --- Noivos ---
export const couples = pgTable('couples', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id),
  partner1Name: text('partner1_name').notNull(),
  partner2Name: text('partner2_name').notNull(),
  weddingDate: date('wedding_date'),
  city: text('city'),
  state: text('state'),
  slug: text('slug').unique(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

export const weddingPages = pgTable('wedding_pages', {
  id: uuid('id').primaryKey().defaultRandom(),
  coupleId: uuid('couple_id').notNull().references(() => couples.id, { onDelete: 'cascade' }),
  title: text('title'),
  story: text('story'),
  coverImageUrl: text('cover_image_url'),
  theme: text('theme').default('classic'),
  isPublished: boolean('is_published').default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

export const giftLists = pgTable('gift_lists', {
  id: uuid('id').primaryKey().defaultRandom(),
  coupleId: uuid('couple_id').notNull().references(() => couples.id, { onDelete: 'cascade' }),
  name: text('name').notNull().default('Lista de Presentes'),
  isMonetary: boolean('is_monetary').default(false),
  pixKey: text('pix_key'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

export const giftItems = pgTable('gift_items', {
  id: uuid('id').primaryKey().defaultRandom(),
  giftListId: uuid('gift_list_id').notNull().references(() => giftLists.id, { onDelete: 'cascade' }),
  name: text('name').notNull(),
  description: text('description'),
  imageUrl: text('image_url'),
  price: numeric('price', { precision: 10, scale: 2 }),
  isPurchased: boolean('is_purchased').default(false),
  purchasedBy: text('purchased_by'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
});

export const wishlists = pgTable('wishlists', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id),
  vendorProfileId: uuid('vendor_profile_id').notNull().references(() => vendorProfiles.id),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
}, (t) => ({ unq: unique().on(t.userId, t.vendorProfileId) }));

export const guests = pgTable('guests', {
  id: uuid('id').primaryKey().defaultRandom(),
  coupleId: uuid('couple_id').notNull().references(() => couples.id, { onDelete: 'cascade' }),
  name: text('name').notNull(),
  email: text('email'),
  phone: text('phone'),
  status: text('status').default('pending'),
  plusOne: boolean('plus_one').default(false),
  dietaryNotes: text('dietary_notes'),
  confirmedAt: timestamp('confirmed_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

// --- Reviews & Leads ---
export const leads = pgTable('leads', {
  id: uuid('id').primaryKey().defaultRandom(),
  vendorProfileId: uuid('vendor_profile_id').notNull().references(() => vendorProfiles.id),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  userId: uuid('user_id').notNull().references(() => users.id),
  message: text('message'),
  status: text('status').default('new'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

export const reviews = pgTable('reviews', {
  id: uuid('id').primaryKey().defaultRandom(),
  vendorProfileId: uuid('vendor_profile_id').notNull().references(() => vendorProfiles.id),
  userId: uuid('user_id').notNull().references(() => users.id),
  leadId: uuid('lead_id').references(() => leads.id),
  rating: integer('rating').notNull(),
  comment: text('comment'),
  isModerated: boolean('is_moderated').default(false),
  isVisible: boolean('is_visible').default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
}, (t) => ({ unq: unique().on(t.userId, t.vendorProfileId) }));

// --- Afiliados ---
export const affiliateLinks = pgTable('affiliate_links', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id),
  code: text('code').unique().notNull(),
  totalReferrals: integer('total_referrals').default(0),
  activeReferrals: integer('active_referrals').default(0),
  badge: text('badge').default('bronze'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

// --- Planos & Assinaturas ---
export const plans = pgTable('plans', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').unique().notNull(),
  price: numeric('price', { precision: 10, scale: 2 }).notNull().default('0'),
  features: jsonb('features').default([]),
  portfolioLimit: integer('portfolio_limit').default(5),
  isActive: boolean('is_active').default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

export const subscriptions = pgTable('subscriptions', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  planId: uuid('plan_id').notNull().references(() => plans.id),
  affiliateLinkId: uuid('affiliate_link_id').references(() => affiliateLinks.id),
  status: text('status').default('active'),
  startedAt: timestamp('started_at', { withTimezone: true }).defaultNow(),
  expiresAt: timestamp('expires_at', { withTimezone: true }),
  cancelledAt: timestamp('cancelled_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

export const commissions = pgTable('commissions', {
  id: uuid('id').primaryKey().defaultRandom(),
  affiliateLinkId: uuid('affiliate_link_id').notNull().references(() => affiliateLinks.id),
  subscriptionId: uuid('subscription_id').notNull().references(() => subscriptions.id),
  amount: numeric('amount', { precision: 10, scale: 2 }).notNull(),
  percentage: numeric('percentage', { precision: 5, scale: 2 }).notNull(),
  status: text('status').default('pending'),
  paidAt: timestamp('paid_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
});

// --- Rankings & Badges ---
export const rankings = pgTable('rankings', {
  id: uuid('id').primaryKey().defaultRandom(),
  vendorProfileId: uuid('vendor_profile_id').unique().notNull().references(() => vendorProfiles.id),
  category: text('category').notNull(),
  city: text('city').notNull(),
  position: integer('position').notNull(),
  score: numeric('score', { precision: 5, scale: 2 }).notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

export const badges = pgTable('badges', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').unique().notNull(),
  minReferrals: integer('min_referrals').notNull(),
  icon: text('icon'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
});
```

---

## CLERK INTEGRATION

### Middleware
```typescript
// middleware.ts
import { authMiddleware } from '@clerk/nextjs';

export default authMiddleware({
  publicRoutes: ['/', '/marketplace', '/marketplace/(.*)', '/casamento/(.*)'],
  afterAuth(auth, req) {
    if (!auth.userId && !auth.isPublicRoute) {
      return Response.redirect(new URL('/sign-in', req.url));
    }
  }
});

export const config = {
  matcher: ['/((?!.+\\.[\\w]+$|_next).*)', '/', '/(api|trpc)(.*)'],
};
```

### Webhooks
```typescript
// app/api/webhooks/clerk/route.ts
import { WebhookEvent } from '@clerk/nextjs/server';
import { headers } from 'next/headers';
import { Webhook } from 'svix';
import { db } from '@/lib/db';
import { users, organizations } from '@/lib/db/schema';

export async function POST(req: Request) {
  const WEBHOOK_SECRET = process.env.CLERK_WEBHOOK_SECRET;
  const headerPayload = headers();
  const svix_id = headerPayload.get('svix-id');
  const svix_timestamp = headerPayload.get('svix-timestamp');
  const svix_signature = headerPayload.get('svix-signature');

  const body = await req.json();
  const wh = new Webhook(WEBHOOK_SECRET!);
  const evt = wh.verify(JSON.stringify(body), {
    'svix-id': svix_id!,
    'svix-timestamp': svix_timestamp!,
    'svix-signature': svix_signature!,
  }) as WebhookEvent;

  switch (evt.type) {
    case 'user.created':
      await db.insert(users).values({
        clerkUserId: evt.data.id,
        email: evt.data.email_addresses[0]?.email_address ?? '',
        name: `${evt.data.first_name ?? ''} ${evt.data.last_name ?? ''}`.trim(),
        avatarUrl: evt.data.image_url,
      });
      break;
    case 'organization.created':
      await db.insert(organizations).values({
        clerkOrgId: evt.data.id,
        name: evt.data.name,
        slug: evt.data.slug ?? evt.data.id,
      });
      break;
  }

  return new Response('OK', { status: 200 });
}
```

---

## SUPABASE INTEGRATION

### Client Setup
```typescript
// lib/supabase/client.ts
import { createClient } from '@supabase/supabase-js';

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);
```

### RLS Helper
```typescript
// lib/supabase/rls.ts
import { auth } from '@clerk/nextjs';
import { db } from '@/lib/db';
import { sql } from 'drizzle-orm';

export async function withOrgContext<T>(callback: () => Promise<T>): Promise<T> {
  const { orgId, userId } = auth();
  if (!orgId || !userId) throw new Error('Não autorizado');

  await db.execute(sql`SELECT set_config('app.current_org_id', ${orgId}, true)`);
  await db.execute(sql`SELECT set_config('app.current_user_id', ${userId}, true)`);
  return callback();
}
```

---

## RESEND INTEGRATION

### Email Templates
```typescript
// emails/welcome-noivo.tsx
import { Html, Head, Body, Container, Text, Button, Heading, Hr } from '@react-email/components';

interface Props { partnerNames: string; weddingDate?: string; }

export function WelcomeNoivoEmail({ partnerNames, weddingDate }: Props) {
  return (
    <Html>
      <Head />
      <Body style={{ backgroundColor: '#FFF8F0', fontFamily: 'sans-serif' }}>
        <Container style={{ maxWidth: '600px', margin: '0 auto', padding: '40px 20px' }}>
          <Heading style={{ color: '#8B6F47', textAlign: 'center' }}>
            💍 Bem-vindos ao Case-me!
          </Heading>
          <Text>Olá, {partnerNames}!</Text>
          <Text>Estamos muito felizes em fazer parte dessa jornada especial.</Text>
          {weddingDate && <Text>Faltam poucos dias para o grande dia: <strong>{weddingDate}</strong></Text>}
          <Hr />
          <Button href="https://caseme.com.br/dashboard" style={{
            backgroundColor: '#C4956A', color: '#fff', padding: '12px 24px',
            borderRadius: '8px', textDecoration: 'none'
          }}>
            Acessar meu painel
          </Button>
        </Container>
      </Body>
    </Html>
  );
}
```

### Send Email Action
```typescript
// lib/actions/email.ts
'use server';
import { Resend } from 'resend';
import { WelcomeNoivoEmail } from '@/emails/welcome-noivo';

const resend = new Resend(process.env.RESEND_API_KEY);

export async function sendWelcomeNoivoEmail(email: string, partnerNames: string) {
  await resend.emails.send({
    from: 'Case-me <contato@caseme.com.br>',
    to: email,
    subject: '💍 Bem-vindos ao Case-me!',
    react: WelcomeNoivoEmail({ partnerNames }),
  });
}
```

---

## ESTRUTURA DE PASTAS

```
/app
  /(auth)
    /sign-in/[[...sign-in]]
    /sign-up/[[...sign-up]]
  /(onboarding)
    /onboarding
      /noivo
      /fornecedor
      /assessor
  /(app)
    /dashboard
      /casamento        ← Dashboard Noivos
      /fornecedor       ← Dashboard Fornecedor
      /afiliado         ← Dashboard Assessor
    /marketplace        ← Busca pública de fornecedores
    /casamento/[slug]   ← Página pública do casamento
  /(admin)
    /admin
  /api
    /webhooks
      /clerk
/components
  /landing             ← Hero, Features, Testimonials, Pricing
  /dashboard           ← Stats, Charts, Cards
  /marketplace         ← VendorCard, Filters, Search
  /wedding             ← WeddingPage, GiftList, RSVP, GuestList
  /affiliate           ← CommissionTable, BadgeDisplay
  /ui                  ← shadcn components
/lib
  /db                  ← Drizzle schema + migrations
  /supabase            ← Client + RLS helpers
  /hooks               ← Custom hooks
  /actions             ← Server Actions por feature
  /validations         ← Zod schemas
/emails                ← React Email templates
/public
  /images
```

---

## SERVER ACTIONS — Exemplos Principais

```typescript
// lib/actions/vendor.ts
'use server';
import { auth } from '@clerk/nextjs';
import { db } from '@/lib/db';
import { vendorProfiles } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';
import { revalidatePath } from 'next/cache';
import { z } from 'zod';

const updateProfileSchema = z.object({
  description: z.string().min(10).max(2000),
  category: z.string().min(1),
  city: z.string().min(1),
  state: z.string().length(2),
  priceRange: z.enum(['economico', 'moderado', 'premium']),
});

export async function updateVendorProfile(data: z.infer<typeof updateProfileSchema>) {
  const { userId, orgId } = auth();
  if (!userId || !orgId) throw new Error('Não autorizado');
  const validated = updateProfileSchema.parse(data);

  await db.update(vendorProfiles)
    .set({ ...validated, updatedAt: new Date() })
    .where(eq(vendorProfiles.organizationId, orgId));

  revalidatePath('/dashboard/fornecedor');
  return { success: true };
}
```

```typescript
// lib/actions/lead.ts
'use server';
import { auth } from '@clerk/nextjs';
import { db } from '@/lib/db';
import { leads } from '@/lib/db/schema';
import { revalidatePath } from 'next/cache';
import { sendNewLeadEmail } from './email';

export async function sendLead(vendorProfileId: string, orgId: string, message: string) {
  const { userId } = auth();
  if (!userId) throw new Error('Não autorizado');

  const [lead] = await db.insert(leads).values({
    vendorProfileId, organizationId: orgId, userId, message,
  }).returning();

  // Notificar fornecedor por email
  await sendNewLeadEmail(vendorProfileId, message);
  revalidatePath('/dashboard/fornecedor');
  return { success: true, leadId: lead.id };
}
```

---

## DESIGN SYSTEM

### Cores (Tailwind Config)
```javascript
// tailwind.config.ts
colors: {
  primary: {
    50: '#FFF8F0',
    100: '#FCECD8',
    200: '#F5D5B0',
    300: '#E8B87A',
    400: '#C4956A',
    500: '#A67B5B',  // cor principal — champagne/dourado
    600: '#8B6F47',
    700: '#6B5335',
    800: '#4A3A25',
    900: '#2D2318',
  },
  accent: {
    rose: '#E8C4C4',    // rosé suave
    gold: '#D4AF37',    // dourado
    cream: '#FFF8F0',   // creme
    blush: '#F5E6E0',   // blush
  },
  neutral: {
    white: '#FFFFFF',
    offwhite: '#FAFAF8',
    gray: '#6B7280',
    dark: '#1F2937',
  },
}
```

### Typography
- **Headings:** Playfair Display (serif, elegância/casamento)
- **Body:** Inter (sans-serif, legibilidade)
- **Small/Labels:** Inter (sans-serif, 12-14px)

### Componentes Base (shadcn/ui)
Button, Card, Dialog, Sheet, Tabs, Table, Badge, Avatar, DropdownMenu, Input, Textarea, Select, Command (busca), Separator, Skeleton, Toast, Tooltip

---

## SEGURANÇA

### Checklist
- ✅ RLS habilitado em todas as tabelas
- ✅ Server Actions validam `auth()` (`userId` e `orgId`)
- ✅ Zod validation em todos os formulários
- ✅ Variáveis de ambiente seguras (nunca expostas no client)
- ✅ CORS configurado
- ✅ Rate limiting (Vercel)
- ✅ Todas as queries filtram por `organization_id`
- ✅ Webhooks Clerk validados com Svix
- ✅ Proteção antifraude em avaliações (vinculação a leads)
- ✅ Soft deletes (dados nunca removidos permanentemente)

---

## PERFORMANCE

### Otimizações
- React Server Components (RSC) por padrão
- Streaming SSR para dashboards
- `next/image` para todas as imagens (portfólios e fotos)
- Edge Functions para APIs de busca
- Paginação cursor-based no marketplace
- Cache com `revalidatePath` e `revalidateTag`
- Lazy loading de componentes pesados (galeria, gráficos)

### Metas
- First Contentful Paint: < 1.5s
- Time to Interactive: < 3s
- Lighthouse Score: > 90

---

## DEPLOY CHECKLIST

**Variáveis de Ambiente (Vercel):**
```bash
# Clerk
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
CLERK_WEBHOOK_SECRET=

# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Resend
RESEND_API_KEY=

# App
NEXT_PUBLIC_URL=https://caseme.com.br
```

**Passos:**
1. Criar projeto no Vercel (conectar GitHub)
2. Criar projeto no Supabase
3. Configurar Clerk (adicionar domínio de produção, habilitar Organizations)
4. Configurar Resend (verificar domínio caseme.com.br)
5. Executar migrations no Supabase (Drizzle push)
6. Configurar webhooks (Clerk → /api/webhooks/clerk)
7. Seed de planos (Free, Pro, Premium) e badges
8. Deploy automático via GitHub

---

## ROADMAP TÉCNICO

**Fase 1 — MVP (Semana 1-3):**
- Setup completo da infraestrutura (Next.js, Clerk, Supabase, Drizzle)
- Landing page premium
- Autenticação e onboarding (noivo + fornecedor)
- Dashboard do Fornecedor (perfil, portfólio, leads)
- Dashboard dos Noivos (contador, wishlist, página pública)
- Marketplace básico (busca, filtros, perfil público)

**Fase 2 — Core Features (Semana 4-6):**
- Sistema de avaliações (com antifraude)
- Rankings públicos
- Lista de presentes + lista monetária
- RSVP digital + gestão de convidados
- Sistema de afiliados (link, tracking, comissões)
- Emails transacionais (todos os templates)

**Fase 3 — Monetização + Polish (Semana 7-8):**
- Planos e assinaturas (integração pagamento)
- Painel administrativo completo
- Otimizações de performance
- Testes E2E
- Monitoramento e analytics
- Documentação final

**Fase 4 — Escala (Semana 9+):**
- Chat em tempo real (Supabase Realtime)
- App mobile (React Native)
- Internacionalização (i18n)
- SEO avançado e marketing automation
