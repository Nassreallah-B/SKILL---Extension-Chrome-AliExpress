# 🧩 SKILL — Extension Chrome : Importateur Produits AliExpress

> **Version** : 1.0.0 | **Dernière MAJ** : 2026-05-05
> **Sous-dossier du projet** : `sc-go/`

---

## 📋 TABLE DES MATIÈRES

| # | Section | Fichier |
|---|---------|---------|
| 1 | Objectif & Architecture | PART1 (ce fichier) |
| 2 | Manifest V3 & Configuration | PART1 |
| 3 | Scraper AliExpress (3 niveaux) | PART1 |
| 4 | Content Script & Injection UI | PART1 |
| 5 | Background Service Worker | PART2 |
| 6 | Popup UI | PART2 |
| 7 | Options Page | PART2 |
| 8 | Types partagés | PART2 |
| 9 | Build System (Vite) | PART2 |
| 10 | Packaging & Distribution | PART2 |
| 11 | API Backend (Endpoint réception) | PART2 |
| 12 | Checklist Déploiement | PART2 |

---

## 1. OBJECTIF & ARCHITECTURE

### 1.1 But

Créer une extension Chrome (Manifest V3) qui :
1. Détecte automatiquement les pages produit AliExpress
2. Extrait toutes les données produit (titre, prix, images, variantes/SKUs, vendeur, avis)
3. Injecte un bouton flottant "Ajouter" directement sur la page AliExpress
4. Envoie les données au backend de la boutique via API REST sécurisée (Bearer JWT)
5. Affiche un popup avec preview du produit détecté + bouton d'import
6. Propose une page d'options pour configurer l'URL backend, le token JWT, et la marge par défaut

### 1.2 Architecture

```
sc-go/
├── manifest.json              ← Manifest V3
├── package.json               ← Config npm (dev deps uniquement)
├── tsconfig.json              ← TypeScript strict
├── vite.config.ts             ← Build multi-entry
├── scripts/
│   └── pack.mjs               ← Packaging ZIP
├── public/
│   └── icons/                 ← Icônes 16/32/48/128px
├── src/
│   ├── shared/
│   │   └── types.ts           ← Interfaces partagées
│   ├── background/
│   │   └── bg.ts              ← Service Worker (messages, import, auth)
│   ├── content/
│   │   ├── scraper.ts         ← Extraction DOM AliExpress (3 fallbacks)
│   │   ├── content.ts         ← Injection bouton + logique import
│   │   └── content.css        ← Styles bouton flottant
│   ├── popup/
│   │   ├── popup.html         ← UI popup (3 états)
│   │   ├── popup.ts           ← Logique popup
│   │   └── popup.css          ← Styles popup
│   └── options/
│       ├── options.html       ← Page configuration
│       ├── options.ts         ← Logique options
│       └── options.css        ← Styles options
└── dist/                      ← Build output (chargé dans Chrome)
```

### 1.3 Flux de données

```
┌────────────────────────────────────────────────────────┐
│  PAGE ALIEXPRESS (content script)                      │
│                                                        │
│  1. waitForProductReady() → DOM prêt                   │
│  2. scrapeAliExpressProduct() → 3 niveaux fallback     │
│     ├── Niveau 1: window.runParams / __AE_DATA__       │
│     ├── Niveau 2: <script type="application/json">     │
│     └── Niveau 3: Extraction DOM pure (sélecteurs)     │
│  3. injectButton() → bouton flottant violet            │
│  4. sendMessage('PRODUCT_DETECTED') → badge "1"       │
└──────────────┬─────────────────────────────────────────┘
               │ chrome.runtime.sendMessage
               ▼
┌────────────────────────────────────────────────────────┐
│  BACKGROUND SERVICE WORKER (bg.ts)                     │
│                                                        │
│  • onMessage listener (dispatch par type)              │
│  • IMPORT_PRODUCT → importProduct()                    │
│    ├── getConfig() depuis chrome.storage.local         │
│    ├── Calcul prix vente = priceMin × defaultMargin    │
│    ├── POST /api/admin/aliexpress/import-extension     │
│    │   Headers: Authorization: Bearer <token>          │
│    └── chrome.notifications.create (succès)            │
│  • CHECK_AUTH → GET /api/admin/aliexpress/status       │
│  • GET_CONFIG → chrome.storage.local.get               │
└──────────────┬─────────────────────────────────────────┘
               │ fetch()
               ▼
┌────────────────────────────────────────────────────────┐
│  BACKEND BOUTIQUE (API Vercel/Express)                 │
│                                                        │
│  POST /api/admin/aliexpress/import-extension           │
│  GET  /api/admin/aliexpress/status                     │
│                                                        │
│  → Validation JWT → Insert DB → Return pendingId       │
└────────────────────────────────────────────────────────┘
```

---

## 2. MANIFEST V3 — CONFIGURATION COMPLÈTE

### 2.1 Fichier `manifest.json`

```json
{
  "manifest_version": 3,
  "name": "SC GO — Product Importer",
  "short_name": "SC GO",
  "version": "1.0.0",
  "description": "Importez des produits AliExpress directement dans votre boutique en 1 clic.",
  "icons": {
    "16": "icons/icon16.png",
    "32": "icons/icon32.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  },
  "background": {
    "service_worker": "background.js",
    "type": "module"
  },
  "content_scripts": [
    {
      "matches": [
        "https://www.aliexpress.com/item/*",
        "https://aliexpress.com/item/*",
        "https://fr.aliexpress.com/item/*",
        "https://www.aliexpress.us/item/*"
      ],
      "js": ["content.js"],
      "css": ["content.css"],
      "run_at": "document_idle"
    }
  ],
  "action": {
    "default_popup": "popup/popup.html",
    "default_title": "SC GO — Product Importer",
    "default_icon": {
      "16": "icons/icon16.png",
      "32": "icons/icon32.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },
  "options_page": "options/options.html",
  "permissions": ["storage", "activeTab", "notifications"],
  "host_permissions": [
    "https://YOUR_BACKEND_DOMAIN/*",
    "https://localhost:*/*"
  ]
}
```

### 2.2 Points critiques du manifest

| Champ | Explication |
|-------|-------------|
| `manifest_version: 3` | Obligatoire pour les nouvelles extensions Chrome (MV2 déprécié) |
| `background.type: "module"` | Permet les imports ES Modules dans le service worker |
| `content_scripts.matches` | Pattern match sur les 4 variantes d'URLs AliExpress produit |
| `content_scripts.run_at: "document_idle"` | Injection après chargement complet du DOM |
| `permissions.storage` | Accès à `chrome.storage.local` pour la config |
| `permissions.activeTab` | Accès à l'onglet actif pour communiquer avec le content script |
| `permissions.notifications` | Notifications système après import réussi |
| `host_permissions` | **ADAPTER** : mettre l'URL réelle du backend déployé |

### 2.3 Règles pour `host_permissions`

- Toujours ajouter le domaine du backend (production + staging)
- Format : `https://mondomaine.com/*` (avec wildcard path)
- Ne JAMAIS mettre `<all_urls>` (refusé par Chrome Web Store)
- Le localhost est utile uniquement en développement

---

## 3. SCRAPER ALIEXPRESS — 3 NIVEAUX DE FALLBACK

### 3.1 Stratégie

AliExpress change fréquemment son DOM et ses structures JS. Le scraper utilise **3 niveaux de fallback** pour maximiser la fiabilité :

| Niveau | Source | Fiabilité | Données |
|--------|--------|-----------|---------|
| 1 | `window.runParams` / `__AE_DATA__` / `_dida_config_` | ⭐⭐⭐ Haute | Complètes (SKUs, prix, store, reviews) |
| 2 | Balises `<script>` JSON embarquées | ⭐⭐ Moyenne | Complètes si JSON trouvé |
| 3 | Extraction DOM pure (sélecteurs CSS) | ⭐ Basse | Partielles (pas de SKUs) |

### 3.2 Fichier `src/content/scraper.ts`

```typescript
// Point d'entrée — cascade de 3 niveaux
export function scrapeAliExpressProduct(): ScrapedProduct | null {
  return scrapeFromWindowData() || scrapeFromScriptTags() || scrapeFromDom();
}
```

### 3.3 Niveau 1 : Variables globales JS

AliExpress expose ses données dans des variables globales JavaScript :
- `window.runParams.data` (ancien format)
- `window.__AE_DATA__` (nouveau format React/Next)
- `window._dida_config_` (format alternatif)

```typescript
function scrapeFromWindowData(): ScrapedProduct | null {
  const w = window as Record<string, unknown>;
  const runParams = w['runParams'] as Record<string, unknown> | undefined;
  const data = runParams?.['data'] as Record<string, unknown> | undefined;
  const aeData = w['__AE_DATA__'] as Record<string, unknown> | undefined;
  const didaConfig = w['_dida_config_'] as Record<string, unknown> | undefined;
  const source = data || aeData || didaConfig;
  if (!source) return null;
  return extractFromDataObject(source);
}
```

### 3.4 Extraction depuis l'objet data structuré

Les modules AliExpress suivent une structure prévisible :

```typescript
function extractFromDataObject(data: Record<string, unknown>): ScrapedProduct | null {
  // Modules AliExpress structurés — recherche récursive
  const titleModule     = findDeep(data, 'titleModule');
  const priceModule     = findDeep(data, 'priceModule');
  const imageModule     = findDeep(data, 'imageModule');
  const skuModule       = findDeep(data, 'skuModule');
  const storeModule     = findDeep(data, 'storeModule');
  const feedbackModule  = findDeep(data, 'feedbackModule');
  const descModule      = findDeep(data, 'descriptionModule');

  // Extraction titre
  const title = titleModule?.['subject'] || titleModule?.['title'] || '';

  // Extraction prix (plusieurs formats possibles)
  const priceMin = priceModule?.['minAmount']?.['value']
    || priceModule?.['formatedPrice']
    || priceModule?.['minActivityAmount']?.['value']
    || '0';

  // Extraction images
  const imageList = imageModule?.['imagePathList']
    || imageModule?.['imagePaths']
    || [];

  // Extraction SKUs/variantes
  const skus = extractSkusFromModule(skuModule);

  // Retour objet complet ScrapedProduct
  return { productId, title, priceMin, priceMax, currency, mainImage,
           images, description, skus, storeId, storeName, storeUrl,
           rating, reviewCount, url, scrapedAt };
}
```

### 3.5 Extraction des variantes (SKUs)

```typescript
function extractSkusFromModule(skuModule): ScrapedSku[] {
  const skuList = skuModule['skuPriceList'] || [];
  const props = skuModule['productSKUPropertyList'] || [];

  return skuList.map((sku) => {
    const skuPropIds = String(sku['skuPropIds'] || '').split(',');
    const attributes: Record<string, string> = {};

    // Résoudre les attributs (couleur, taille, etc.)
    for (const prop of props) {
      const propValues = prop['skuPropertyValues'] || [];
      const propName = String(prop['skuPropertyName'] || '');
      for (const val of propValues) {
        if (skuPropIds.includes(String(val['propertyValueId']))) {
          attributes[propName] = String(
            val['propertyValueDisplayName'] || val['propertyValueName']
          );
        }
      }
    }

    // Prix de la variante
    const skuPrice = sku['skuVal'];
    const price = parseFloat(
      skuPrice?.['actSkuCalPrice'] || skuPrice?.['skuCalPrice'] || '0'
    );
    const stock = parseInt(
      skuPrice?.['skuQuantity'] || sku['skuStock'] || '999', 10
    );

    return {
      skuId: String(sku['skuId']),
      name: Object.values(attributes).join(' · ') || 'Variante',
      price, stock,
      image: sku['skuImage'] || undefined,
      attributes,
    };
  });
}
```

### 3.6 Niveau 2 : Balises `<script>` JSON

```typescript
function scrapeFromScriptTags(): ScrapedProduct | null {
  const scripts = Array.from(
    document.querySelectorAll('script[type="application/json"], script:not([src])')
  );
  for (const script of scripts) {
    const text = script.textContent || '';
    if (text.includes('skuModule') || text.includes('titleModule')
        || text.includes('priceModule')) {
      try {
        const parsed = JSON.parse(text);
        const result = extractFromDataObject(parsed);
        if (result) return result;
      } catch { /* JSON invalide, continuer */ }
    }
  }
  return null;
}
```

### 3.7 Niveau 3 : Extraction DOM pure

Sélecteurs CSS utilisés (peuvent changer — à maintenir) :

| Donnée | Sélecteurs CSS |
|--------|---------------|
| **Titre** | `h1[data-pl="product-title"]`, `.product-title-text`, `[class*="title--"] h1`, `h1.product-title` |
| **Prix** | `[class*="price--"] [class*="current"]`, `[class*="uniform-banner-box-price"]`, `.product-price-value`, `[data-pl="product-price"]` |
| **Image principale** | `[class*="magnifier"] img`, `.product-image img`, `meta[property="og:image"]` |
| **Images galerie** | `[class*="slider--"] img`, `.images-view-list img`, `[class*="gallery"] img` |
| **Boutique** | `[class*="shop-name"]`, `[class*="store-header"] a` |
| **Note** | `[class*="score--"]`, `[class*="overview-rating"]` |
| **Nb avis** | `[class*="quantity--"]`, `[class*="review-count"]` |

### 3.8 Helpers du scraper

```typescript
// Extraire l'ID produit depuis l'URL : /item/1234567890.html → "1234567890"
function extractProductIdFromUrl(): string {
  const match = window.location.pathname.match(/\/item\/(\d+)\.html/);
  return match?.[1] || '';
}

// Recherche récursive d'une clé dans un objet imbriqué (max 8 niveaux)
function findDeep(obj: unknown, key: string, depth = 0): unknown {
  if (depth > 8) return null;
  if (typeof obj !== 'object' || obj === null) return null;
  const record = obj as Record<string, unknown>;
  if (key in record) return record[key];
  for (const v of Object.values(record)) {
    const found = findDeep(v, key, depth + 1);
    if (found !== null) return found;
  }
  return null;
}

// Image haute résolution : retirer les suffixes de redimensionnement
imageUrl.replace(/_\d+x\d+.*?\.jpg/, '.jpg');
```

---

## 4. CONTENT SCRIPT & INJECTION UI

### 4.1 Fichier `src/content/content.ts`

#### Initialisation avec attente DOM

```typescript
let currentProduct: ScrapedProduct | null = null;
let buttonInjected = false;

// Attendre que titre + prix soient dans le DOM (max 10s = 20 × 500ms)
function waitForProductReady(callback: () => void, attempts = 0) {
  const hasTitle = !!document.querySelector(
    'h1, [class*="title--"], .product-title-text'
  );
  const hasPrice = !!document.querySelector(
    '[class*="price--"], .product-price-value, [data-pl="product-price"]'
  );
  if (hasTitle && hasPrice) callback();
  else if (attempts < 20) setTimeout(() => waitForProductReady(callback, attempts + 1), 500);
}

function init() {
  waitForProductReady(() => {
    currentProduct = scrapeAliExpressProduct();
    if (currentProduct?.productId) {
      injectButton(currentProduct);
      chrome.runtime.sendMessage({ type: 'PRODUCT_DETECTED', payload: currentProduct });
    }
  });
}
```

#### Injection du bouton flottant

```typescript
function injectButton(product: ScrapedProduct) {
  if (buttonInjected) return;
  buttonInjected = true;

  const btn = document.createElement('div');
  btn.id = 'sc-go-btn';
  btn.innerHTML = `
    <button id="sc-go-trigger" title="Ajouter au catalogue">
      <span class="sc-go-icon">✦</span>
      <span class="sc-go-label">Ajouter au catalogue</span>
    </button>
    <div id="sc-go-toast" class="sc-go-toast"></div>
  `;
  document.body.appendChild(btn);
  document.getElementById('sc-go-trigger')?.addEventListener('click', () => {
    handleImport(product);
  });
}
```

#### Logique d'import + états du bouton

```typescript
async function handleImport(product: ScrapedProduct) {
  setButtonState('loading');
  const response = await chrome.runtime.sendMessage({
    type: 'IMPORT_PRODUCT', payload: product,
  });
  if (response?.success) {
    setButtonState('success');
    showToast('✅ Ajouté au catalogue !', 'success');
    // Lien direct vers l'admin
    if (response.adminUrl) {
      const link = document.createElement('a');
      link.href = response.adminUrl;
      link.target = '_blank';
      link.textContent = '→ Voir dans l\'admin';
      document.getElementById('sc-go-btn')?.appendChild(link);
    }
  } else {
    setButtonState('error');
    showToast(`❌ ${response?.error || 'Erreur'}`, 'error');
    setTimeout(() => setButtonState('idle'), 3000);
  }
}

type ButtonState = 'idle' | 'loading' | 'success' | 'error';

function setButtonState(state: ButtonState) {
  const btn = document.getElementById('sc-go-trigger');
  btn?.setAttribute('data-state', state);
  const labels: Record<ButtonState, string> = {
    idle: 'Ajouter au catalogue',
    loading: 'Synchronisation…',
    success: 'Dans le catalogue !',
    error: 'Erreur — Réessayer',
  };
  const label = btn?.querySelector('.sc-go-label');
  if (label) label.textContent = labels[state];
}
```

#### Gestion SPA (navigation sans rechargement)

```typescript
// Observer pour détecter les changements de page SPA d'AliExpress
const observer = new MutationObserver(() => {
  const isProductPage = /\/item\/\d+\.html/.test(window.location.pathname);
  if (isProductPage && !buttonInjected) init();
});
observer.observe(document.body, { childList: true, subtree: true });
init(); // Premier lancement
```

#### Répondre au popup (GET_PRODUCT)

```typescript
chrome.runtime.onMessage.addListener((message, _sender, sendResponse) => {
  if (message.type === 'GET_PRODUCT') {
    sendResponse(currentProduct); // Réponse synchrone
    return false;
  }
  return false;
});
```

### 4.2 Fichier `src/content/content.css` — Bouton flottant

```css
/* Conteneur fixe en bas à droite, z-index max */
#sc-go-btn {
  position: fixed;
  bottom: 32px;
  right: 32px;
  z-index: 2147483647;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 8px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

/* Bouton principal — gradient violet */
#sc-go-trigger {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 22px;
  background: linear-gradient(135deg, #7c3aed, #4f46e5);
  color: #ffffff;
  border: none;
  border-radius: 50px;
  font-size: 15px;
  font-weight: 700;
  cursor: pointer;
  box-shadow: 0 8px 32px rgba(124, 58, 237, 0.45),
              0 2px 8px rgba(0, 0, 0, 0.15);
  transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}

/* Hover : élévation */
#sc-go-trigger:hover {
  transform: translateY(-2px) scale(1.03);
  box-shadow: 0 12px 40px rgba(124, 58, 237, 0.55);
}

/* États visuels via data-attribute */
#sc-go-trigger[data-state='loading'] {
  background: linear-gradient(135deg, #6d28d9, #4338ca);
  cursor: wait;
  pointer-events: none;
}
#sc-go-trigger[data-state='loading'] .sc-go-icon {
  animation: sc-go-spin 1s linear infinite;
}
#sc-go-trigger[data-state='success'] {
  background: linear-gradient(135deg, #059669, #10b981);
  pointer-events: none;
}
#sc-go-trigger[data-state='error'] {
  background: linear-gradient(135deg, #dc2626, #ef4444);
}

/* Toast notification */
.sc-go-toast {
  display: none; opacity: 0; transform: translateY(4px);
  padding: 12px 18px; border-radius: 12px; font-size: 13px;
  font-weight: 600; max-width: 280px;
  backdrop-filter: blur(8px);
  transition: opacity 0.3s, transform 0.3s;
}
.sc-go-toast--visible { display: block; opacity: 1; transform: translateY(0); }
.sc-go-toast--success { background: rgba(16, 185, 129, 0.95); color: #fff; }
.sc-go-toast--error   { background: rgba(239, 68, 68, 0.95); color: #fff; }
```

---

> **Suite** → [SKILL_CHROME_EXTENSION_ALIEXPRESS_PART2.md](./SKILL_CHROME_EXTENSION_ALIEXPRESS_PART2.md)
