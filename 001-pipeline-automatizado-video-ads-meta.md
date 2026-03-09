# Pipeline Automatizado: Generación de Videos + Campañas Meta Ads

**Fecha:** 2026-03-09
**Estado:** Investigación inicial

---

## Objetivo

Automatizar el flujo completo:

1. Generar videos de 5-10 segundos desde un frame inicial + prompt (Higgsfield)
2. Organizar los videos con copy/prompts para redes sociales
3. Crear, actualizar, optimizar y eliminar campañas pagas en Meta vía API

---

## 1. Generación de Videos — Higgsfield

### ¿Qué es Higgsfield?

Higgsfield es una plataforma de generación de video con IA que ofrece API para crear videos cortos a partir de imágenes + prompts de texto.

### Capacidades clave

- **Image-to-Video:** Toma un frame inicial (imagen) y genera un video de 5-10 segundos
- **Text-to-Video:** Genera video solo desde un prompt de texto
- **Estilo controlable:** Permite definir movimiento de cámara, estilo visual, etc.
- **API REST:** Permite integración programática (no depende solo de UI)

### Flujo de uso programático

```
Frame inicial (imagen .png/.jpg)
        +
Prompt de texto ("zoom in on product, cinematic lighting, smooth motion")
        ↓
POST /api/v1/generations
        ↓
Poll status hasta completar
        ↓
Download video (.mp4)
```

### Ejemplo de integración (conceptual)

```javascript
// Pseudocódigo - adaptar según docs actuales de Higgsfield
const generateVideo = async (imageUrl, prompt) => {
  const response = await fetch('https://api.higgsfield.ai/v1/generations', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${HIGGSFIELD_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      image_url: imageUrl,
      prompt: prompt,
      duration: 5, // segundos
      aspect_ratio: '9:16' // vertical para Reels/Stories
    })
  });

  const { generation_id } = await response.json();

  // Polling hasta que termine
  let status = 'processing';
  while (status === 'processing') {
    await sleep(5000);
    const check = await fetch(`https://api.higgsfield.ai/v1/generations/${generation_id}`, {
      headers: { 'Authorization': `Bearer ${HIGGSFIELD_API_KEY}` }
    });
    const result = await check.json();
    status = result.status;
    if (status === 'completed') return result.video_url;
  }
};
```

### Consideraciones

- **Tiempo de generación:** Típicamente 1-5 minutos por video
- **Costo:** Verificar pricing actual en higgsfield.ai (modelo por créditos/generación)
- **Formatos de salida:** MP4, distintas resoluciones
- **Aspect ratios:** 9:16 (Reels/Stories), 1:1 (Feed), 16:9 (Horizontal)
- **Limitaciones:** Rate limits de API, calidad variable según prompt

---

## 2. Organización de Contenido con IA

### Generación de Copy para Redes

Usar Claude API o similar para generar automáticamente:

- **Caption/Copy** optimizado por plataforma (Instagram, Facebook)
- **Hashtags** relevantes
- **CTA** (Call to Action) adaptado al objetivo de campaña
- **Variaciones A/B** del copy para testing

```javascript
const generateAdCopy = async (productDescription, platform, objective) => {
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages: [{
      role: 'user',
      content: `Genera 3 variaciones de copy publicitario para ${platform}.
        Producto: ${productDescription}
        Objetivo: ${objective}
        Formato: JSON con campos: headline, primary_text, description, cta, hashtags`
    }]
  });
  return JSON.parse(response.content[0].text);
};
```

### Estructura de datos para organizar assets

```javascript
// Modelo de un "Creative Asset"
{
  id: "creative_001",
  product: "zapatillas-running-x",
  frame_original: "s3://bucket/frames/zap-001.png",
  prompt_video: "slow zoom on running shoes, dynamic lighting, energy",
  video_url: "s3://bucket/videos/zap-001-v1.mp4",
  duration: 5,
  aspect_ratio: "9:16",
  copies: [
    {
      platform: "instagram",
      headline: "Corre sin límites",
      primary_text: "Las nuevas X llevan tu running al siguiente nivel...",
      cta: "SHOP_NOW",
      hashtags: ["#running", "#fitness"]
    }
  ],
  status: "ready", // draft | generating | ready | deployed | archived
  performance: {
    ctr: null,
    cpm: null,
    roas: null
  }
}
```

### Storage recomendado

- **Videos/Imágenes:** Cloudflare R2 o AWS S3
- **Metadata/Estado:** PostgreSQL o MongoDB
- **Cola de trabajos:** BullMQ (Redis) para orquestar generación

---

## 3. Automatización de Campañas Meta — Marketing API

### Estructura de Meta Ads

```
Cuenta Publicitaria
  └── Campaña (objetivo: conversiones, tráfico, etc.)
       └── Ad Set (audiencia, presupuesto, schedule)
            └── Ad (creative: video + copy)
```

### Setup inicial requerido

1. **Meta Business Manager** con cuenta publicitaria
2. **App de Meta Developers** (developers.facebook.com)
3. **Access Token** de larga duración con permisos:
   - `ads_management`
   - `ads_read`
   - `business_management`
4. **App Review** aprobado para permisos de ads (si es producción)

### SDK oficial

```bash
npm install facebook-nodejs-business-sdk
# o
pip install facebook-business
```

### Flujo completo automatizado

#### A. Subir video creative

```javascript
const bizSdk = require('facebook-nodejs-business-sdk');
const AdAccount = bizSdk.AdAccount;
const AdVideo = bizSdk.AdVideo;

const account = new AdAccount(`act_${AD_ACCOUNT_ID}`);

// Subir video
const video = await account.createAdVideo([], {
  file_url: videoUrl, // URL pública del video generado
  title: 'Creative zapatillas v1'
});

const videoId = video.id;
```

#### B. Crear campaña

```javascript
const Campaign = bizSdk.Campaign;

const campaign = await account.createCampaign([], {
  name: 'Auto - Zapatillas Running - 2026-03',
  objective: 'OUTCOME_SALES', // o OUTCOME_TRAFFIC, OUTCOME_AWARENESS
  status: 'PAUSED', // Crear pausada, activar después de revisar
  special_ad_categories: [],
  buying_type: 'AUCTION'
});
```

#### C. Crear Ad Set (audiencia + presupuesto)

```javascript
const AdSet = bizSdk.AdSet;

const adSet = await account.createAdSet([], {
  name: 'AdSet - Runners 25-45 - Colombia',
  campaign_id: campaign.id,
  daily_budget: 2000, // centavos: $20.00 USD
  billing_event: 'IMPRESSIONS',
  optimization_goal: 'OFFSITE_CONVERSIONS',
  bid_strategy: 'LOWEST_COST_WITHOUT_CAP',
  targeting: {
    geo_locations: { countries: ['CO'] },
    age_min: 25,
    age_max: 45,
    interests: [{ id: '6003107902433', name: 'Running' }],
    publisher_platforms: ['facebook', 'instagram'],
    instagram_positions: ['stream', 'story', 'reels'],
    facebook_positions: ['feed', 'story', 'reels']
  },
  start_time: '2026-03-10T00:00:00-0500',
  status: 'PAUSED'
});
```

#### D. Crear Ad (video + copy)

```javascript
const Ad = bizSdk.Ad;
const AdCreative = bizSdk.AdCreative;

// Crear creative
const creative = await account.createAdCreative([], {
  name: 'Creative - Zapatillas v1',
  object_story_spec: {
    page_id: PAGE_ID,
    instagram_actor_id: IG_ACCOUNT_ID,
    video_data: {
      video_id: videoId,
      title: 'Corre sin límites',
      message: 'Las nuevas X llevan tu running al siguiente nivel. Compra ahora.',
      call_to_action: {
        type: 'SHOP_NOW',
        value: { link: 'https://tutienda.com/zapatillas-x' }
      }
    }
  }
});

// Crear ad
const ad = await account.createAd([], {
  name: 'Ad - Zapatillas v1 - Copy A',
  adset_id: adSet.id,
  creative: { creative_id: creative.id },
  status: 'PAUSED'
});
```

#### E. Leer métricas de rendimiento

```javascript
const insights = await campaign.getInsights(
  ['impressions', 'clicks', 'spend', 'ctr', 'cpm', 'actions', 'cost_per_action_type'],
  {
    time_range: { since: '2026-03-10', until: '2026-03-16' },
    time_increment: 1 // Diario
  }
);
```

#### F. Optimización automática (actualizar/pausar/eliminar)

```javascript
// Pausar ads con bajo rendimiento
const autoOptimize = async (campaignId) => {
  const adsets = await campaign.getAdSets(['id', 'name'], {});

  for (const adset of adsets) {
    const insights = await adset.getInsights(['ctr', 'spend', 'actions'], {
      time_range: { since: '2026-03-10', until: '2026-03-16' }
    });

    const ctr = parseFloat(insights[0]?.ctr || 0);
    const spend = parseFloat(insights[0]?.spend || 0);

    // Regla: si gastó más de $10 y CTR < 0.5%, pausar
    if (spend > 10 && ctr < 0.5) {
      await adset.update({ status: 'PAUSED' });
      console.log(`Pausado: ${adset.name} - CTR: ${ctr}%`);
    }

    // Regla: si CTR > 2%, aumentar presupuesto 20%
    if (ctr > 2) {
      const currentBudget = adset.daily_budget;
      await adset.update({ daily_budget: Math.round(currentBudget * 1.2) });
      console.log(`Escalado: ${adset.name} - CTR: ${ctr}%`);
    }
  }
};
```

---

## 4. Arquitectura del Pipeline Completo

```
┌─────────────────────────────────────────────────────────────────┐
│                        ORQUESTADOR (Node.js + BullMQ)          │
│                                                                 │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────────────┐ │
│  │ Input:   │    │ Higgsfield   │    │ Claude API            │ │
│  │ Frames + │───▶│ Video Gen    │───▶│ Generar copy/prompts  │ │
│  │ Prompts  │    │ API          │    │ para cada plataforma  │ │
│  └──────────┘    └──────┬───────┘    └───────────┬───────────┘ │
│                         │                        │              │
│                         ▼                        ▼              │
│                  ┌──────────────┐    ┌───────────────────────┐ │
│                  │ Cloudflare   │    │ Base de datos         │ │
│                  │ R2 / S3      │    │ (PostgreSQL)          │ │
│                  │ (videos)     │    │ (metadata, estado)    │ │
│                  └──────┬───────┘    └───────────┬───────────┘ │
│                         │                        │              │
│                         ▼                        ▼              │
│                  ┌─────────────────────────────────────────┐   │
│                  │        Meta Marketing API               │   │
│                  │  - Upload video creative                │   │
│                  │  - Crear Campaign > AdSet > Ad          │   │
│                  │  - Monitorear métricas                  │   │
│                  │  - Auto-optimizar (pausar/escalar)      │   │
│                  └─────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ CRON JOBS                                               │   │
│  │  - Cada 6h: revisar métricas y auto-optimizar           │   │
│  │  - Cada 24h: generar reporte de rendimiento             │   │
│  │  - Semanal: rotar creatives con bajo rendimiento        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Stack Tecnológico Recomendado

| Componente | Tecnología | Razón |
|---|---|---|
| Runtime | Node.js / Bun | Ecosistema JS, SDK de Meta oficial |
| Video Gen | Higgsfield API | Image-to-video con API programática |
| Copy Gen | Claude API (Sonnet) | Calidad de copy, bajo costo, rápido |
| Cola de trabajos | BullMQ + Redis | Manejo de jobs async, reintentos, prioridad |
| Storage videos | Cloudflare R2 | Económico, compatible S3, CDN incluido |
| Base de datos | PostgreSQL | Relacional, bueno para métricas y reportes |
| Ads | Meta Marketing API + SDK | Control total sobre campañas |
| Scheduler | node-cron o Temporal | Cron jobs de optimización |
| Dashboard | Next.js (opcional) | Visualizar estado del pipeline |

---

## 6. Pasos para Implementar (MVP)

### Fase 1 — Generación de video
- [ ] Obtener API key de Higgsfield
- [ ] Script que toma imagen + prompt → genera video → sube a R2
- [ ] Probar con 5-10 frames distintos

### Fase 2 — Copy automático
- [ ] Integrar Claude API para generar copy por plataforma
- [ ] Template de prompts optimizados para ads (headline, text, CTA)
- [ ] Generar 3 variaciones A/B por creative

### Fase 3 — Meta Ads API
- [ ] Configurar Meta Business App + permisos
- [ ] Script para crear campaña completa (Campaign → AdSet → Ad)
- [ ] Script para subir video y asociar a Ad
- [ ] Probar en modo PAUSED antes de activar

### Fase 4 — Optimización automática
- [ ] Cron job que lee métricas cada 6h
- [ ] Reglas de auto-optimización (pausar bajo CTR, escalar alto ROAS)
- [ ] Rotación de creatives: reemplazar los que bajan rendimiento
- [ ] Alertas (Slack/email) cuando se toman acciones automáticas

### Fase 5 — Dashboard y monitoreo
- [ ] Panel web para ver estado de creatives y campañas
- [ ] Métricas consolidadas: gasto, CTR, ROAS por creative
- [ ] Controles manuales de override

---

## 7. Riesgos y Consideraciones

- **Meta App Review:** Los permisos `ads_management` requieren revisión por Meta. Puede tomar días/semanas.
- **Política de anuncios:** Los videos generados por IA deben cumplir políticas de Meta (no contenido engañoso).
- **Costos:** Higgsfield + Claude API + gasto en ads. Monitorear ROI total.
- **Rate limits:** Meta API tiene rate limits estrictos. Usar batch requests donde sea posible.
- **Calidad de video:** Revisar manualmente los primeros batches antes de automatizar al 100%.
- **Presupuesto de seguridad:** Poner límites de gasto diario máximo a nivel de cuenta para evitar gastos descontrolados en automatización.

---

## 8. Links y Referencias

- Higgsfield: https://higgsfield.ai
- Meta Marketing API: https://developers.facebook.com/docs/marketing-apis/
- Meta Business SDK (Node.js): https://github.com/facebook/facebook-nodejs-business-sdk
- Claude API: https://docs.anthropic.com/
- BullMQ: https://bullmq.io/
- Cloudflare R2: https://developers.cloudflare.com/r2/
