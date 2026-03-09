# Plan de Implementación: Meta Ads Manager App

**Fecha:** 2026-03-09
**Estado:** Plan de implementación
**Stack:** Next.js 15 + shadcn/ui + TypeScript + Prisma + PostgreSQL

---

## Visión General

App desktop-first, AI-first, tema claro y moderno para gestionar campañas de Meta Ads de forma visual y rápida. Permite subir videos pre-generados, mejorar copy con IA, y administrar el ciclo completo de campañas publicitarias desde una sola interfaz.

---

## 1. Arquitectura General

```
┌─────────────────────────────────────────────────────────────────────┐
│                          FRONTEND (Next.js 15 App Router)          │
│                                                                     │
│  ┌──────────┐ ┌──────────────┐ ┌────────────┐ ┌────────────────┐  │
│  │Dashboard │ │Campaign Mgr  │ │AI Studio   │ │Media Library   │  │
│  │Overview  │ │CRUD completo │ │Copy/Prompt │ │Videos/Thumbs   │  │
│  └──────────┘ └──────────────┘ └────────────┘ └────────────────┘  │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                          API LAYER (Route Handlers + Server Actions)│
│                                                                     │
│  ┌──────────┐ ┌──────────────┐ ┌────────────┐ ┌────────────────┐  │
│  │Auth      │ │Meta Ads API  │ │AI Providers│ │File Upload     │  │
│  │(NextAuth)│ │(SDK wrapper) │ │(OpenRouter) │ │(R2/S3)        │  │
│  └──────────┘ └──────────────┘ └────────────┘ └────────────────┘  │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                          DATA LAYER                                 │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
│  │ PostgreSQL        │  │ Cloudflare R2    │  │ Redis (cache    │  │
│  │ (Prisma ORM)      │  │ (media storage)  │  │ + rate limits)  │  │
│  └──────────────────┘  └──────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Estructura del Proyecto

```
meta-ads-manager/
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── src/
│   ├── app/
│   │   ├── layout.tsx                    # Root layout (tema claro, fonts)
│   │   ├── page.tsx                      # Redirect a /dashboard
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   └── callback/meta/route.ts    # OAuth callback Meta
│   │   ├── (app)/
│   │   │   ├── layout.tsx                # App shell: sidebar + topbar
│   │   │   ├── dashboard/
│   │   │   │   └── page.tsx              # Dashboard overview
│   │   │   ├── campaigns/
│   │   │   │   ├── page.tsx              # Lista de campañas
│   │   │   │   ├── new/page.tsx          # Crear campaña
│   │   │   │   └── [id]/
│   │   │   │       ├── page.tsx          # Detalle campaña
│   │   │   │       ├── adsets/
│   │   │   │       │   ├── page.tsx
│   │   │   │       │   └── [adsetId]/
│   │   │   │       │       ├── page.tsx  # Detalle ad set
│   │   │   │       │       └── ads/
│   │   │   │       │           ├── page.tsx
│   │   │   │       │           └── [adId]/page.tsx
│   │   │   │       └── insights/page.tsx # Métricas de campaña
│   │   │   ├── media/
│   │   │   │   └── page.tsx              # Media library
│   │   │   ├── ai-studio/
│   │   │   │   ├── page.tsx              # Hub principal AI
│   │   │   │   ├── copy-generator/page.tsx
│   │   │   │   ├── prompt-generator/page.tsx
│   │   │   │   └── thumbnail-generator/page.tsx
│   │   │   └── settings/
│   │   │       ├── page.tsx
│   │   │       ├── meta-connection/page.tsx
│   │   │       └── ai-providers/page.tsx
│   │   └── api/
│   │       ├── auth/[...nextauth]/route.ts
│   │       ├── meta/
│   │       │   ├── campaigns/route.ts
│   │       │   ├── adsets/route.ts
│   │       │   ├── ads/route.ts
│   │       │   ├── insights/route.ts
│   │       │   └── upload/route.ts
│   │       ├── ai/
│   │       │   ├── generate-copy/route.ts
│   │       │   ├── improve-copy/route.ts
│   │       │   └── generate-prompt/route.ts
│   │       └── media/
│   │           ├── upload/route.ts
│   │           └── [id]/route.ts
│   ├── components/
│   │   ├── ui/                           # shadcn/ui components
│   │   ├── layout/
│   │   │   ├── sidebar.tsx
│   │   │   ├── topbar.tsx
│   │   │   ├── command-palette.tsx       # ⌘K quick actions
│   │   │   └── breadcrumbs.tsx
│   │   ├── dashboard/
│   │   │   ├── stats-cards.tsx
│   │   │   ├── spend-chart.tsx
│   │   │   ├── performance-table.tsx
│   │   │   └── quick-actions.tsx
│   │   ├── campaigns/
│   │   │   ├── campaign-card.tsx
│   │   │   ├── campaign-form.tsx
│   │   │   ├── budget-slider.tsx
│   │   │   ├── status-toggle.tsx
│   │   │   ├── targeting-builder.tsx
│   │   │   └── objective-selector.tsx
│   │   ├── adsets/
│   │   │   ├── adset-card.tsx
│   │   │   ├── adset-form.tsx
│   │   │   ├── schedule-picker.tsx
│   │   │   └── audience-builder.tsx
│   │   ├── ads/
│   │   │   ├── ad-card.tsx
│   │   │   ├── ad-form.tsx
│   │   │   ├── ad-preview.tsx            # Preview visual del ad
│   │   │   ├── media-selector.tsx
│   │   │   ├── copy-editor.tsx
│   │   │   └── cta-selector.tsx
│   │   ├── ai/
│   │   │   ├── ai-chat-panel.tsx         # Panel lateral de IA
│   │   │   ├── copy-improver.tsx
│   │   │   ├── prompt-builder.tsx
│   │   │   └── suggestion-card.tsx
│   │   └── media/
│   │       ├── media-grid.tsx
│   │       ├── video-player.tsx
│   │       ├── upload-dropzone.tsx
│   │       └── media-details.tsx
│   ├── lib/
│   │   ├── prisma.ts                     # Prisma client singleton
│   │   ├── auth.ts                       # NextAuth config
│   │   ├── meta/
│   │   │   ├── client.ts                 # Meta API client wrapper
│   │   │   ├── campaigns.ts              # CRUD campañas
│   │   │   ├── adsets.ts                 # CRUD ad sets
│   │   │   ├── ads.ts                    # CRUD ads
│   │   │   ├── insights.ts              # Métricas
│   │   │   ├── upload.ts                # Upload de media a Meta
│   │   │   └── types.ts                 # Tipos de Meta API
│   │   ├── ai/
│   │   │   ├── client.ts                # Multi-provider client
│   │   │   ├── prompts.ts               # System prompts templates
│   │   │   └── types.ts
│   │   ├── storage/
│   │   │   └── r2.ts                    # Cloudflare R2 client
│   │   └── utils.ts
│   ├── hooks/
│   │   ├── use-meta-campaigns.ts
│   │   ├── use-meta-insights.ts
│   │   ├── use-ai-generate.ts
│   │   ├── use-media-upload.ts
│   │   └── use-budget-update.ts
│   └── types/
│       ├── meta.ts
│       ├── ai.ts
│       └── media.ts
├── public/
├── .env.local
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

---

## 3. Schema de Base de Datos (Prisma)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─── AUTH ────────────────────────────────────────────

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String?
  image         String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  metaAccounts  MetaAccount[]
  aiProviders   AIProvider[]
  mediaAssets   MediaAsset[]

  accounts      Account[]
  sessions      Session[]
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

// ─── META ADS ────────────────────────────────────────

model MetaAccount {
  id              String   @id @default(cuid())
  userId          String
  metaUserId      String
  accessToken     String   // Encrypted
  tokenExpiresAt  DateTime
  adAccountId     String   // act_XXXXXXX
  adAccountName   String
  pageId          String?
  pageName        String?
  igAccountId     String?
  isActive        Boolean  @default(true)
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  user            User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  campaigns       CampaignSync[]
}

// Tabla local para cache/sync de campañas Meta
model CampaignSync {
  id              String   @id @default(cuid())
  metaAccountId   String
  metaCampaignId  String   @unique // ID en Meta
  name            String
  objective       String
  status          String   // ACTIVE, PAUSED, ARCHIVED
  dailyBudget     Float?
  lifetimeBudget  Float?
  lastSyncAt      DateTime @default(now())
  createdAt       DateTime @default(now())

  metaAccount     MetaAccount @relation(fields: [metaAccountId], references: [id], onDelete: Cascade)
  adSetSyncs      AdSetSync[]
}

model AdSetSync {
  id              String   @id @default(cuid())
  campaignSyncId  String
  metaAdSetId     String   @unique
  name            String
  status          String
  dailyBudget     Float?
  targeting       Json?    // Snapshot del targeting
  lastSyncAt      DateTime @default(now())

  campaignSync    CampaignSync @relation(fields: [campaignSyncId], references: [id], onDelete: Cascade)
  adSyncs         AdSync[]
}

model AdSync {
  id              String   @id @default(cuid())
  adSetSyncId     String
  metaAdId        String   @unique
  name            String
  status          String
  creativeData    Json?    // Snapshot del creative
  lastSyncAt      DateTime @default(now())

  adSetSync       AdSetSync @relation(fields: [adSetSyncId], references: [id], onDelete: Cascade)
}

// ─── MEDIA ───────────────────────────────────────────

model MediaAsset {
  id            String   @id @default(cuid())
  userId        String
  type          String   // video, image, thumbnail
  filename      String
  url           String   // URL en R2/S3
  mimeType      String
  sizeBytes     Int
  duration      Float?   // Para videos, en segundos
  width         Int?
  height        Int?
  metaVideoId   String?  // ID después de subir a Meta
  tags          String[] // Tags para organizar
  createdAt     DateTime @default(now())

  user          User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

// ─── AI ──────────────────────────────────────────────

model AIProvider {
  id            String   @id @default(cuid())
  userId        String
  provider      String   // openrouter, openai, anthropic
  apiKey        String   // Encrypted
  isDefault     Boolean  @default(false)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  user          User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model AIGeneration {
  id            String   @id @default(cuid())
  userId        String
  type          String   // copy, video_prompt, thumbnail_prompt
  provider      String
  model         String
  inputPrompt   String
  systemPrompt  String?
  output        String
  tokensUsed    Int?
  costUsd       Float?
  createdAt     DateTime @default(now())
}
```

---

## 4. Módulos Principales

### 4.1 Autenticación y Conexión Meta OAuth

**Flujo:**

```
Usuario → Login (email/magic link via NextAuth)
       → Settings → "Conectar Meta"
       → Redirect a Facebook OAuth
       → Permisos: ads_management, ads_read, pages_show_list,
                   pages_read_engagement, business_management,
                   instagram_basic, instagram_manage_insights
       → Callback → Guardar access_token (encrypted)
       → Seleccionar Ad Account + Page
```

**Implementación clave (`src/lib/meta/client.ts`):**

```typescript
import { FacebookAdsApi, AdAccount } from 'facebook-nodejs-business-sdk';

export function createMetaClient(accessToken: string) {
  const api = FacebookAdsApi.init(accessToken);
  api.setDebug(process.env.NODE_ENV === 'development');
  return api;
}

export function getAdAccount(accessToken: string, adAccountId: string) {
  createMetaClient(accessToken);
  return new AdAccount(adAccountId);
}
```

**Token refresh:** Meta long-lived tokens duran ~60 días. Implementar refresh automático vía cron o middleware que detecte expiración próxima.

---

### 4.2 Dashboard Overview

**Ruta:** `/dashboard`

**Contenido:**

```
┌─────────────────────────────────────────────────────────────┐
│ [Sidebar]  │  Dashboard                            [⌘K] 🔔 │
│            │                                                │
│ Dashboard  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────┐ │
│ Campaigns  │  │ Spend   │ │ Impress.│ │ Clicks  │ │ CTR │ │
│ Media      │  │ $1,234  │ │ 45.2K   │ │ 1,847   │ │2.1% │ │
│ AI Studio  │  │ ↑ 12%   │ │ ↑ 8%    │ │ ↑ 15%   │ │↑0.3 │ │
│ Settings   │  └─────────┘ └─────────┘ └─────────┘ └─────┘ │
│            │                                                │
│            │  ┌─────────────────────────────────────────┐   │
│            │  │ Spend Over Time (Area Chart - Recharts) │   │
│            │  │ ▁▂▃▅▆▇█▇▆▅▄▃▂▁▂▃▅▆▇                   │   │
│            │  └─────────────────────────────────────────┘   │
│            │                                                │
│            │  ┌─────────────────────────────────────────┐   │
│            │  │ Top Campaigns              Quick Budget │   │
│            │  │ ┌───┬──────┬────┬────┬───────┬───────┐ │   │
│            │  │ │ ● │ Name │Spnd│ CTR│ ROAS  │Budget │ │   │
│            │  │ ├───┼──────┼────┼────┼───────┼───────┤ │   │
│            │  │ │ 🟢│ Camp1│$320│2.4%│ 3.2x  │[$──]  │ │   │
│            │  │ │ 🟢│ Camp2│$180│1.8%│ 2.1x  │[$──]  │ │   │
│            │  │ │ 🟡│ Camp3│$95 │0.9%│ 1.1x  │[$──]  │ │   │
│            │  │ │ 🔴│ Camp4│$50 │0.3%│ 0.4x  │[$──]  │ │   │
│            │  │ └───┴──────┴────┴────┴───────┴───────┘ │   │
│            │  └─────────────────────────────────────────┘   │
│            │                                                │
│            │  ┌──────────────────┐ ┌────────────────────┐   │
│            │  │ Quick Actions    │ │ AI Suggestions     │   │
│            │  │ + New Campaign   │ │ "Camp4 tiene CTR   │   │
│            │  │ ↑ Upload Media   │ │  bajo. ¿Pausar?"   │   │
│            │  │ ✨ Generate Copy │ │ [Pausar] [Ignorar] │   │
│            │  └──────────────────┘ └────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Componentes:**
- `StatsCards` — KPIs principales con comparación vs periodo anterior
- `SpendChart` — Gráfico de gasto diario (Recharts area chart)
- `PerformanceTable` — Tabla interactiva con inline budget sliders
- `QuickActions` — Accesos rápidos a acciones frecuentes
- `AISuggestions` — Panel con sugerencias basadas en métricas

**Data fetching:** Server Components + React Query para polling cada 5min de métricas.

---

### 4.3 Gestión de Campañas

**Ruta:** `/campaigns`

#### Vista de lista

- Tabla con DataTable de shadcn (basado en TanStack Table)
- Columnas: Status toggle, Nombre, Objetivo, Budget (editable inline), Spend, CTR, CPC, ROAS, Acciones
- Filtros: por status, objetivo, rango de fechas
- Búsqueda por nombre
- Bulk actions: pausar/activar/archivar múltiples

#### Crear campaña (`/campaigns/new`)

```
Paso 1: Objetivo
┌──────────────────────────────────────────────┐
│ ¿Cuál es tu objetivo?                        │
│                                              │
│ [🛒 Ventas]  [🔗 Tráfico]  [👁 Awareness]  │
│ [📩 Leads]   [📱 App]      [💬 Engagement]  │
└──────────────────────────────────────────────┘

Paso 2: Configuración
┌──────────────────────────────────────────────┐
│ Nombre: [Auto-generado o manual]             │
│ Budget type: (●) Diario  ( ) Lifetime       │
│ Budget: [$] [───●────] [$50/día]            │
│ Bid strategy: [Lowest Cost ▼]               │
│ Schedule: [Mar 10] → [Mar 30]               │
└──────────────────────────────────────────────┘

Paso 3: Audiencia (Ad Set)
┌──────────────────────────────────────────────┐
│ Ubicación: [Colombia ▼] [+Agregar]          │
│ Edad: [25] ─────●───── [45]                 │
│ Género: [Todos ▼]                           │
│ Intereses: [Running] [Fitness] [+]          │
│ Placements:                                  │
│   ☑ Facebook Feed   ☑ Instagram Feed        │
│   ☑ Facebook Reels  ☑ Instagram Reels       │
│   ☑ Stories         ☐ Audience Network       │
└──────────────────────────────────────────────┘

Paso 4: Anuncio (Ad Creative)
┌──────────────────────────────────────────────┐
│ Formato: (●) Video  ( ) Imagen  ( ) Carousel│
│                                              │
│ ┌────────────┐  Headline: [___________]     │
│ │            │  Primary text:               │
│ │  [Video    │  [________________________]  │
│ │  Preview]  │  [✨ Mejorar con IA]         │
│ │            │                              │
│ │ [Cambiar]  │  CTA: [Shop Now ▼]          │
│ └────────────┘  URL: [https://...]          │
│                                              │
│ [← Atrás]              [Crear en Pausa →]   │
│                         [Crear y Activar →]  │
└──────────────────────────────────────────────┘
```

#### Detalle de campaña (`/campaigns/[id]`)

- Header con nombre, status toggle, badge de objetivo
- Budget control con slider + input numérico (update instantáneo)
- Tabs: Ad Sets | Ads | Insights | Configuración
- Cada tab con su propia tabla/grid interactiva
- Drill-down: Campaign → Ad Sets → Ads (breadcrumb navigation)

#### Budget rápido

Componente `BudgetSlider` reutilizable en todos los niveles:

```typescript
// Edición inline de presupuesto en cualquier nivel
<BudgetSlider
  entityType="campaign" // | "adset" | "ad"
  entityId={metaCampaignId}
  currentBudget={dailyBudget}
  budgetType="daily"
  onChange={async (newBudget) => {
    await updateMetaBudget(entityType, entityId, newBudget);
    toast.success(`Budget actualizado a $${newBudget}`);
  }}
/>
```

---

### 4.4 Gestión de Ads

**Cada ad es editable en todos sus aspectos:**

| Campo | Componente UI | Meta API field |
|---|---|---|
| Status | Toggle switch | `status` |
| Nombre | Input editable | `name` |
| Video/Imagen | Media selector + preview | `object_story_spec.video_data` |
| Headline | Input + AI improve btn | `asset_feed_spec.titles` |
| Primary Text | Textarea + AI improve btn | `asset_feed_spec.bodies` |
| Description | Input | `asset_feed_spec.descriptions` |
| CTA | Dropdown selector | `call_to_action.type` |
| URL destino | Input con validación | `call_to_action.value.link` |
| Formulario | Form builder (para Lead Ads) | `lead_gen_form_id` |
| Tipo de ad | Selector (single/carousel/collection) | `creative.format` |

**Ad Preview:** Componente que simula cómo se verá el ad en Feed, Stories, y Reels usando los datos reales.

---

### 4.5 AI Studio

**Ruta:** `/ai-studio`

#### Módulo de Copy

```
┌──────────────────────────────────────────────────────────┐
│ AI Copy Generator                                        │
│                                                          │
│ Prompt base:                                             │
│ ┌──────────────────────────────────────────────────────┐ │
│ │ Zapatillas de running ultraligeras, suela reactiva,  │ │
│ │ para corredores urbanos de 25-40 años...             │ │
│ └──────────────────────────────────────────────────────┘ │
│                                                          │
│ Plataforma: [Instagram ▼]  Tono: [Energético ▼]        │
│ Objetivo: [Conversiones ▼] Variaciones: [3]             │
│                                                          │
│ [✨ Generar Copy]                                        │
│                                                          │
│ ┌─ Variación A ──────────────────────────────────────┐  │
│ │ Headline: "Corre sin límites"                      │  │
│ │ Primary: "Las nuevas X redefinen tu running.       │  │
│ │          Suela reactiva que devuelve cada paso..."  │  │
│ │ CTA: Shop Now                                      │  │
│ │ [Usar en Ad] [Copiar] [Regenerar] [Editar]         │  │
│ ├─ Variación B ──────────────────────────────────────┤  │
│ │ Headline: "Tu próximo PR empieza aquí"             │  │
│ │ ...                                                │  │
│ └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

#### Módulo de Prompts para Video

Genera prompts optimizados para Higgsfield/otros generadores:

```
Input: Descripción del producto + estilo deseado
Output: Prompt técnico listo para image-to-video

Ejemplo output:
"Slow cinematic zoom on white running shoes on urban concrete,
golden hour lighting, shallow depth of field, subtle dust particles,
smooth camera dolly forward, 5 seconds, 9:16 vertical"
```

#### Módulo de Thumbnails

Genera prompts para crear thumbnails que complementen los videos.

#### Implementación AI multi-provider (`src/lib/ai/client.ts`)

```typescript
type AIProvider = 'openrouter' | 'openai' | 'anthropic';

interface AIConfig {
  provider: AIProvider;
  apiKey: string;
  model: string;
}

export async function generateCompletion(
  config: AIConfig,
  systemPrompt: string,
  userPrompt: string
): Promise<string> {
  const baseUrls: Record<AIProvider, string> = {
    openrouter: 'https://openrouter.ai/api/v1',
    openai: 'https://api.openai.com/v1',
    anthropic: 'https://api.anthropic.com/v1',
  };

  // OpenRouter y OpenAI comparten formato OpenAI-compatible
  if (config.provider === 'openrouter' || config.provider === 'openai') {
    const res = await fetch(`${baseUrls[config.provider]}/chat/completions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${config.apiKey}`,
        'Content-Type': 'application/json',
        ...(config.provider === 'openrouter' && {
          'HTTP-Referer': process.env.NEXT_PUBLIC_APP_URL,
        }),
      },
      body: JSON.stringify({
        model: config.model,
        messages: [
          { role: 'system', content: systemPrompt },
          { role: 'user', content: userPrompt },
        ],
      }),
    });
    const data = await res.json();
    return data.choices[0].message.content;
  }

  // Anthropic usa su propio formato
  if (config.provider === 'anthropic') {
    const res = await fetch(`${baseUrls.anthropic}/messages`, {
      method: 'POST',
      headers: {
        'x-api-key': config.apiKey,
        'anthropic-version': '2023-06-01',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: config.model,
        max_tokens: 1024,
        system: systemPrompt,
        messages: [{ role: 'user', content: userPrompt }],
      }),
    });
    const data = await res.json();
    return data.content[0].text;
  }
}
```

#### System prompts pre-configurados (`src/lib/ai/prompts.ts`)

```typescript
export const SYSTEM_PROMPTS = {
  improveCopy: `Eres un experto en copywriting para ads de Meta (Facebook/Instagram).
Tu trabajo es tomar un copy base y mejorarlo para maximizar engagement y conversiones.
Reglas:
- Mantén el mensaje core pero hazlo más compelling
- Usa hooks emocionales en la primera línea
- Incluye urgencia o escasez cuando sea apropiado
- Adapta el tono a la plataforma (más casual en IG, más informativo en FB)
- Máximo 125 caracteres para headline, 250 para primary text
- Responde en JSON: { headline, primaryText, description }`,

  generateVideoPrompt: `Eres un experto en prompts para generación de video con IA.
Genera prompts técnicos optimizados para modelos image-to-video.
Incluye: movimiento de cámara, iluminación, duración, aspect ratio, estilo visual.
Formato: prompt listo para usar, en inglés, una sola línea.`,

  generateThumbnailPrompt: `Eres un experto en diseño de thumbnails para ads.
Genera prompts para crear thumbnails que capturen atención en el feed.
Prioriza: contraste alto, texto mínimo, producto destacado, composición limpia.`,
};
```

---

### 4.6 Media Library

**Ruta:** `/media`

```
┌──────────────────────────────────────────────────────────┐
│ Media Library                         [↑ Upload] [Filter]│
│                                                          │
│ [All] [Videos] [Images] [Thumbnails]    🔍 Buscar...    │
│                                                          │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│ │ ▶       │ │ ▶       │ │ 🖼      │ │ ▶       │       │
│ │         │ │         │ │         │ │         │       │
│ │ vid1.mp4│ │ vid2.mp4│ │ thumb1  │ │ vid3.mp4│       │
│ │ 5s 9:16 │ │ 8s 9:16 │ │ 1080²  │ │ 5s 1:1  │       │
│ │ ✓ Meta  │ │ ○ Local │ │ ○ Local │ │ ✓ Meta  │       │
│ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
└──────────────────────────────────────────────────────────┘
```

- Drag & drop upload
- Video preview inline con player
- Tags para organizar
- Indicador de si ya está subido a Meta o solo local
- Acción rápida: "Subir a Meta" para usar en ads

---

## 5. Diseño y UI

### Principios

- **Desktop-first:** Layouts amplios, sidebars fijas, tablas completas
- **AI-first:** Botón de IA contextual en cada campo de texto editable
- **Tema claro:** Background `white`/`zinc-50`, accents en `blue-600`
- **Tipografía:** Inter (UI) + JetBrains Mono (datos numéricos)
- **Densidad:** Compacta pero legible — optimizada para ver mucha data

### shadcn/ui components a usar

```bash
# Core
npx shadcn@latest add button input label card table badge
npx shadcn@latest add dialog sheet dropdown-menu command
npx shadcn@latest add tabs separator avatar tooltip

# Forms
npx shadcn@latest add form select textarea switch checkbox
npx shadcn@latest add slider popover calendar

# Data
npx shadcn@latest add data-table pagination skeleton
npx shadcn@latest add chart                    # Recharts wrapper

# Feedback
npx shadcn@latest add toast alert sonner
```

### Command Palette (⌘K)

Acceso rápido a todo:
- Buscar campañas por nombre
- Acciones: "Crear campaña", "Subir video", "Generar copy"
- Navegar entre secciones
- Toggle status de campañas

---

## 6. Integración Meta Ads API — Detalle

### Endpoints principales a implementar

```typescript
// src/lib/meta/campaigns.ts

// LISTAR campañas
export async function listCampaigns(account: AdAccount) {
  return account.getCampaigns(
    ['id', 'name', 'objective', 'status', 'daily_budget',
     'lifetime_budget', 'created_time', 'updated_time'],
    { limit: 100 }
  );
}

// CREAR campaña
export async function createCampaign(account: AdAccount, data: CreateCampaignInput) {
  return account.createCampaign([], {
    name: data.name,
    objective: data.objective,
    status: data.status,
    special_ad_categories: data.specialCategories || [],
    daily_budget: data.dailyBudget ? Math.round(data.dailyBudget * 100) : undefined,
    lifetime_budget: data.lifetimeBudget ? Math.round(data.lifetimeBudget * 100) : undefined,
  });
}

// ACTUALIZAR campaña (budget, status, nombre)
export async function updateCampaign(campaignId: string, data: Partial<UpdateCampaignInput>) {
  const campaign = new Campaign(campaignId);
  return campaign.update([], data);
}

// ELIMINAR campaña
export async function deleteCampaign(campaignId: string) {
  const campaign = new Campaign(campaignId);
  return campaign.delete();
}

// INSIGHTS
export async function getCampaignInsights(campaignId: string, dateRange: DateRange) {
  const campaign = new Campaign(campaignId);
  return campaign.getInsights(
    ['impressions', 'clicks', 'spend', 'ctr', 'cpc', 'cpm',
     'actions', 'cost_per_action_type', 'video_avg_time_watched_actions'],
    {
      time_range: { since: dateRange.from, until: dateRange.to },
      time_increment: 1,
    }
  );
}
```

### Sync strategy

- **No duplicar Meta como source of truth.** La DB local es cache/sync.
- Al abrir dashboard → fetch fresh de Meta API → update local
- Cambios se hacen directo a Meta API → si éxito → update local
- Sync completo cada 15 min via background job (opcional)

---

## 7. Seguridad

| Aspecto | Solución |
|---|---|
| Access tokens Meta | Encriptados en DB con `aes-256-gcm`, key en env var |
| API keys de AI | Encriptados igual que tokens de Meta |
| Auth de usuario | NextAuth con session strategy JWT |
| CSRF | Manejado por Next.js Server Actions |
| Rate limiting | Middleware con upstash/ratelimit en API routes |
| Input validation | Zod schemas en todos los endpoints |
| Permisos Meta | Validar scopes en cada request, refresh proactivo |

---

## 8. Dependencias Principales

```json
{
  "dependencies": {
    "next": "^15.x",
    "react": "^19.x",
    "typescript": "^5.x",
    "@prisma/client": "^6.x",
    "next-auth": "^5.x",
    "facebook-nodejs-business-sdk": "^22.x",
    "@tanstack/react-query": "^5.x",
    "@tanstack/react-table": "^8.x",
    "recharts": "^2.x",
    "zod": "^3.x",
    "zustand": "^5.x",
    "tailwindcss": "^4.x",
    "sonner": "^2.x",
    "@aws-sdk/client-s3": "^3.x",
    "date-fns": "^4.x",
    "lucide-react": "latest"
  }
}
```

---

## 9. Fases de Implementación

### Fase 1 — Scaffold + Auth (2-3 días)
- [ ] `create-next-app` + shadcn/ui + Tailwind + Prisma setup
- [ ] Schema de DB + migraciones
- [ ] NextAuth con email login
- [ ] Layout base: sidebar, topbar, tema claro
- [ ] Command palette (⌘K)

### Fase 2 — Meta OAuth + Conexión (2-3 días)
- [ ] Flujo OAuth completo con Meta
- [ ] Guardar tokens encriptados
- [ ] Selección de Ad Account y Page
- [ ] Test de conexión: listar campañas existentes

### Fase 3 — Dashboard + Lectura de Campañas (3-4 días)
- [ ] Dashboard overview con stats cards y charts
- [ ] Lista de campañas con DataTable
- [ ] Drill-down: Campaign → Ad Sets → Ads
- [ ] Vista de insights/métricas por campaña
- [ ] Polling de métricas con React Query

### Fase 4 — CRUD Completo de Campañas (4-5 días)
- [ ] Crear campaña (wizard multi-paso)
- [ ] Editar campaña: nombre, budget, status, targeting
- [ ] Budget sliders inline en todos los niveles
- [ ] Crear/editar Ad Sets con audience builder
- [ ] Crear/editar Ads con media selector + copy editor
- [ ] Eliminar/archivar en todos los niveles
- [ ] Ad preview component

### Fase 5 — Media Library (2-3 días)
- [ ] Upload de videos/imágenes a R2
- [ ] Media grid con preview y metadata
- [ ] Upload de media a Meta (para usar en ads)
- [ ] Tags y organización

### Fase 6 — AI Studio (3-4 días)
- [ ] Settings: configurar AI providers (OpenRouter/OpenAI/Anthropic)
- [ ] Copy generator con variaciones
- [ ] Copy improver (botón ✨ contextual en campos de texto)
- [ ] Prompt generator para videos
- [ ] Prompt generator para thumbnails
- [ ] Historial de generaciones

### Fase 7 — Polish + Optimización (2-3 días)
- [ ] Responsiveness para tablet/mobile
- [ ] Loading states y skeletons
- [ ] Error handling con toast notifications
- [ ] Keyboard shortcuts
- [ ] Bulk actions en tablas
- [ ] Performance: lazy loading, optimistic updates

### Total estimado: ~18-25 días para MVP completo

---

## 10. Variables de Entorno

```env
# Database
DATABASE_URL="postgresql://..."

# Auth
NEXTAUTH_SECRET="..."
NEXTAUTH_URL="http://localhost:3000"

# Meta
META_APP_ID="..."
META_APP_SECRET="..."

# Storage
R2_ACCOUNT_ID="..."
R2_ACCESS_KEY="..."
R2_SECRET_KEY="..."
R2_BUCKET_NAME="meta-ads-media"
R2_PUBLIC_URL="..."

# Encryption
ENCRYPTION_KEY="..." # Para encriptar tokens en DB

# AI (opcionales, user configura desde UI)
OPENROUTER_API_KEY="..."
OPENAI_API_KEY="..."
ANTHROPIC_API_KEY="..."
```
