# 🧩 SKILL — Extension Chrome AliExpress — PART 2

> Suite de [PART1](./SKILL_CHROME_EXTENSION_ALIEXPRESS_PART1.md)

---

## 5. BACKGROUND SERVICE WORKER

### 5.1 Fichier `src/background/bg.ts`

Le service worker est le cerveau de l'extension. Il gère :
- Le dispatch des messages (IMPORT_PRODUCT, CHECK_AUTH, GET_CONFIG, PRODUCT_DETECTED)
- L'appel API vers le backend
- Le badge de l'icône extension
- Les notifications Chrome

#### Dispatch des messages

```typescript
chrome.runtime.onMessage.addListener(
  (message: { type: string; payload?: unknown }, _sender, sendResponse) => {
    if (message.type === 'IMPORT_PRODUCT') {
      importProduct(message.payload as ScrapedProduct)
        .then(sendResponse)
        .catch((err) => sendResponse({
          success: false,
          error: err instanceof Error ? err.message : String(err)
        }));
      return true; // ⚠️ OBLIGATOIRE pour réponse asynchrone
    }
    if (message.type === 'CHECK_AUTH') {
      checkAuth().then(sendResponse);
      return true;
    }
    if (message.type === 'GET_CONFIG') {
      getConfig().then(sendResponse);
      return true;
    }
    if (message.type === 'PRODUCT_DETECTED') {
      chrome.action.setBadgeText({ text: '1' });
      chrome.action.setBadgeBackgroundColor({ color: '#7c3aed' });
      return false; // Synchrone
    }
    return false;
  }
);
```

> **⚠️ Règle critique** : `return true` est OBLIGATOIRE quand `sendResponse` est appelé de façon asynchrone (dans un `.then()`). Sans cela, le port de message se ferme avant la réponse.

#### Reset du badge sur navigation

```typescript
chrome.tabs.onUpdated.addListener((tabId, changeInfo) => {
  if (changeInfo.status === 'loading') {
    chrome.action.setBadgeText({ text: '', tabId });
  }
});
```

### 5.2 Fonction `importProduct()`

```typescript
async function importProduct(product: ScrapedProduct): Promise<ImportResult> {
  const config = await getConfig();

  // Validation config
  if (!config.token) {
    return { success: false, error: 'Token manquant. Configurez dans les options.' };
  }
  if (!config.backendUrl) {
    return { success: false, error: 'URL backend manquante. Configurez dans les options.' };
  }

  // Calcul prix de vente avec marge
  const sellingPrice = Math.ceil(product.priceMin * config.defaultMargin);

  const body = {
    productId: product.productId,
    title: product.title,
    price: { min: product.priceMin, max: product.priceMax, currency: product.currency },
    mainImage: product.mainImage,
    images: product.images,
    description: product.description,
    skus: product.skus,
    storeId: product.storeId,
    storeName: product.storeName,
    rating: product.rating,
    reviewCount: product.reviewCount,
    url: product.url,
    sellingPrice,
    source: 'extension',
  };

  const res = await fetch(`${config.backendUrl}/api/admin/aliexpress/import-extension`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${config.token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });

  const data = await res.json();
  if (!res.ok || data['error']) {
    return { success: false, error: String(data['error'] || `HTTP ${res.status}`) };
  }

  // Notification Chrome native
  await chrome.notifications.create({
    type: 'basic',
    iconUrl: 'icons/icon48.png',
    title: 'Produit importé !',
    message: `"${product.title.slice(0, 60)}" ajouté au catalogue.`,
  });

  return {
    success: true,
    pendingId: String(data['pendingId'] || ''),
    adminUrl: `${config.backendUrl}/admin?tab=aliexpress#pending`,
  };
}
```

### 5.3 Fonction `checkAuth()`

```typescript
async function checkAuth(): Promise<AuthStatus> {
  const config = await getConfig();
  if (!config.token || !config.backendUrl) {
    return { authenticated: false, error: 'Non configuré' };
  }
  try {
    const res = await fetch(`${config.backendUrl}/api/admin/aliexpress/status`, {
      headers: { Authorization: `Bearer ${config.token}` },
    });
    if (res.ok) return { authenticated: true, backendUrl: config.backendUrl };
    return { authenticated: false, error: `HTTP ${res.status}` };
  } catch (err) {
    return { authenticated: false, error: err instanceof Error ? err.message : String(err) };
  }
}
```

### 5.4 Fonction `getConfig()`

```typescript
async function getConfig(): Promise<ScGoConfig> {
  const result = await chrome.storage.local.get([
    'scgo_token', 'scgo_backend_url', 'scgo_margin'
  ]);
  return {
    token: String(result['scgo_token'] || ''),
    backendUrl: String(result['scgo_backend_url'] || ''),
    defaultMargin: parseFloat(String(result['scgo_margin'] || '2.5')),
  };
}
```

### 5.5 Clés `chrome.storage.local`

| Clé | Type | Default | Description |
|-----|------|---------|-------------|
| `scgo_token` | string | `''` | JWT admin du backend |
| `scgo_backend_url` | string | `''` | URL complète du backend (sans trailing slash) |
| `scgo_margin` | number | `2.5` | Multiplicateur de prix (ex: ×2.5) |

---

## 6. POPUP UI (3 ÉTATS)

### 6.1 États du popup

| État | Condition | Affichage |
|------|-----------|-----------|
| `unconfigured` | `!auth.authenticated` | Logo + message + bouton "Configurer" |
| `product` | Sur page AliExpress + produit détecté | Preview produit + bouton "Importer" |
| `idle` | Authentifié mais pas sur page produit | Stats du jour + lien catalogue + bouton options |

### 6.2 Structure HTML `popup.html`

```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <link rel="stylesheet" href="popup.css" />
</head>
<body>
  <div id="app">
    <!-- État 1 : Non configuré -->
    <div id="state-unconfigured" class="state" hidden>
      <div class="logo-block">
        <span class="logo-icon">✦</span>
        <span class="logo-text">SC GO</span>
      </div>
      <p class="desc">Configurez l'extension pour importer des produits.</p>
      <button id="btn-open-options" class="btn btn-primary">⚙️ Configurer</button>
    </div>

    <!-- État 2 : Produit détecté -->
    <div id="state-product" class="state" hidden>
      <div class="header">
        <span class="logo-icon">✦</span>
        <span class="logo-text">SC GO</span>
        <span id="status-dot" class="status-dot status-dot--connected"></span>
      </div>
      <div id="product-preview" class="product-preview">
        <img id="product-img" src="" alt="" class="product-thumb" />
        <div class="product-info">
          <div id="product-title" class="product-title"></div>
          <div id="product-price" class="product-price"></div>
          <div id="product-store" class="product-store"></div>
        </div>
      </div>
      <button id="btn-import" class="btn btn-primary btn-import">
        <span class="btn-icon">🚀</span>
        <span class="btn-label">Importer</span>
      </button>
      <div id="import-feedback" class="import-feedback" hidden></div>
      <a id="link-admin" href="#" target="_blank" class="link-admin" hidden>
        → Voir dans l'admin
      </a>
    </div>

    <!-- État 3 : Idle (pas sur AliExpress) -->
    <div id="state-idle" class="state" hidden>
      <div class="header">
        <span class="logo-icon">✦</span>
        <span class="logo-text">SC GO</span>
        <span class="status-dot status-dot--connected"></span>
      </div>
      <p class="hint">Naviguez sur une fiche produit AliExpress pour importer.</p>
      <div class="stats">
        <div class="stat-item">
          <span id="stat-today" class="stat-value">—</span>
          <span class="stat-label">imports aujourd'hui</span>
        </div>
      </div>
      <div class="actions">
        <a id="link-catalog" href="#" target="_blank" class="btn btn-secondary">📦 Catalogue</a>
        <button id="btn-options" class="btn btn-ghost">⚙️</button>
      </div>
    </div>
  </div>
  <script src="popup.js" type="module"></script>
</body>
</html>
```

### 6.3 Logique `popup.ts`

```typescript
async function init() {
  const auth = await chrome.runtime.sendMessage({ type: 'CHECK_AUTH' }) as AuthStatus;
  const config = await chrome.runtime.sendMessage({ type: 'GET_CONFIG' }) as { backendUrl: string };

  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
  const isAliExpress = /aliexpress\.com\/item\/\d+/.test(tab?.url || '');

  if (!auth.authenticated) {
    showState('unconfigured');
    document.getElementById('btn-open-options')?.addEventListener('click', () => {
      chrome.runtime.openOptionsPage();
    });
    return;
  }

  if (isAliExpress && tab?.id) {
    try {
      detectedProduct = await chrome.tabs.sendMessage(tab.id, { type: 'GET_PRODUCT' });
    } catch { detectedProduct = null; }

    if (detectedProduct?.productId) {
      showState('product');
      renderProductPreview(detectedProduct);
      setupImportButton(detectedProduct);
    } else {
      showState('idle');
    }
  } else {
    showState('idle');
  }
}

function showState(name: 'unconfigured' | 'product' | 'idle') {
  document.querySelectorAll<HTMLElement>('.state').forEach(el => el.hidden = true);
  document.getElementById(`state-${name}`)!.hidden = false;
}
```

---

## 7. OPTIONS PAGE

### 7.1 Champs de configuration

| Champ | Type | Description |
|-------|------|-------------|
| URL Backend | `<input type="url">` | URL complète du backend déployé |
| Token Admin JWT | `<input type="password">` | Token JWT récupéré dans l'admin |
| Marge par défaut | `<input type="range" min="1.2" max="5" step="0.1">` | Multiplicateur prix (×1.2 à ×5.0) |

### 7.2 Fonctionnalités

- **Bouton Coller** 📋 : `navigator.clipboard.readText()` pour coller le token
- **Toggle visibilité** 👁 : basculer le champ token entre `password` et `text`
- **Sauvegarde** 💾 : écriture dans `chrome.storage.local`
- **Test connexion** 🔗 : `GET /api/admin/aliexpress/status` avec le token

### 7.3 Logique `options.ts`

```typescript
async function save() {
  const backendUrl = (document.getElementById('backend-url') as HTMLInputElement)
    .value.trim().replace(/\/$/, ''); // Retirer trailing slash
  const token = (document.getElementById('token') as HTMLInputElement).value.trim();
  const margin = parseFloat((document.getElementById('margin') as HTMLInputElement).value);

  await chrome.storage.local.set({
    scgo_backend_url: backendUrl,
    scgo_token: token,
    scgo_margin: margin,
  });
  showFeedback('✅ Configuration sauvegardée !', 'success');
}

async function testConnection() {
  const data = await chrome.storage.local.get(['scgo_token', 'scgo_backend_url']);
  if (!data['scgo_token']) {
    showFeedback('❌ Token manquant.', 'error');
    return;
  }
  const res = await fetch(`${data['scgo_backend_url']}/api/admin/aliexpress/status`, {
    headers: { Authorization: `Bearer ${data['scgo_token']}` },
  });
  if (res.ok) showFeedback('✅ Connexion réussie !', 'success');
  else showFeedback(`❌ Erreur HTTP ${res.status}`, 'error');
}
```

---

## 8. TYPES PARTAGÉS

### 8.1 Fichier `src/shared/types.ts`

```typescript
export interface ScrapedProduct {
  productId: string;
  title: string;
  priceMin: number;
  priceMax: number;
  currency: string;
  mainImage: string;
  images: string[];
  description: string;
  skus: ScrapedSku[];
  storeId: string;
  storeName: string;
  storeUrl: string;
  rating: number;
  reviewCount: number;
  url: string;
  scrapedAt: string; // ISO 8601
}

export interface ScrapedSku {
  skuId: string;
  name: string;        // Ex: "Rouge · XL"
  price: number;
  stock: number;
  image?: string;
  attributes: Record<string, string>; // Ex: { "Couleur": "Rouge", "Taille": "XL" }
}

export interface ScGoConfig {
  backendUrl: string;
  token: string;
  defaultMargin: number; // Multiplicateur ex: 2.5
}

export type MessageType =
  | { type: 'IMPORT_PRODUCT'; payload: ScrapedProduct }
  | { type: 'CHECK_AUTH' }
  | { type: 'GET_CONFIG' }
  | { type: 'GET_PRODUCT' }
  | { type: 'PRODUCT_DETECTED'; payload: ScrapedProduct | null };

export interface ImportResult {
  success: boolean;
  pendingId?: string;
  error?: string;
  adminUrl?: string;
}

export interface AuthStatus {
  authenticated: boolean;
  backendUrl?: string;
  error?: string;
}
```

---

## 9. BUILD SYSTEM (VITE)

### 9.1 `vite.config.ts`

```typescript
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  root: 'src',
  build: {
    outDir: '../dist',
    emptyOutDir: true,
    sourcemap: false,
    rollupOptions: {
      input: {
        background: resolve(__dirname, 'src/background/bg.ts'),
        content:    resolve(__dirname, 'src/content/content.ts'),
        popup:      resolve(__dirname, 'src/popup/popup.html'),
        options:    resolve(__dirname, 'src/options/options.html'),
      },
      output: {
        entryFileNames: (chunk) => {
          const map: Record<string, string> = {
            background: 'background.js',
            content: 'content.js',
            popup: 'popup.js',
            options: 'options.js',
          };
          return map[chunk.name] || '[name].js';
        },
        assetFileNames: (asset) => {
          if (asset.name?.endsWith('.css')) {
            const map: Record<string, string> = {
              'content.css': 'content.css',
              'popup.css': 'popup.css',
              'options.css': 'options.css',
            };
            return map[asset.name] || '[name][extname]';
          }
          return 'assets/[name][extname]';
        },
      },
    },
  },
});
```

### 9.2 Points critiques Vite pour extensions Chrome

| Règle | Explication |
|-------|-------------|
| `root: 'src'` | Le code source est dans `src/` |
| `outDir: '../dist'` | Le build sort dans `dist/` à la racine |
| `sourcemap: false` | Pas de sourcemaps en production |
| `entryFileNames` custom | Les fichiers JS doivent être à la racine de `dist/` (pas dans des sous-dossiers) pour que `manifest.json` les trouve |
| Multi-entry | 4 entrées : background, content, popup, options |

### 9.3 `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "skipLibCheck": true,
    "types": ["chrome"]
  },
  "include": ["src", "vite.config.ts"]
}
```

### 9.4 `package.json`

```json
{
  "name": "sc-go",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite build --watch",
    "build": "vite build && node --input-type=module -e \"import{copyFileSync}from'fs';copyFileSync('src/content/content.css','dist/content.css');console.log('content.css copied')\"",
    "pack": "node scripts/pack.mjs"
  },
  "devDependencies": {
    "@types/chrome": "^0.0.321",
    "typescript": "~5.8.2",
    "vite": "^7.3.2"
  }
}
```

### 9.5 Commandes

```bash
# Développement (watch mode — rebuild automatique)
npm run dev

# Build production
npm run build

# Packaging ZIP pour distribution
npm run pack
```

---

## 10. PACKAGING & DISTRIBUTION

### 10.1 Script `scripts/pack.mjs`

```javascript
import { execSync } from 'child_process';
import { cpSync, mkdirSync } from 'fs';
import { join, resolve } from 'path';

const ROOT = resolve(import.meta.dirname, '..');
const DIST = join(ROOT, 'dist');
const PKG  = join(ROOT, 'package');

// Nettoyer
mkdirSync(PKG, { recursive: true });

// Copier dist/ → package/
cpSync(DIST, PKG, { recursive: true });

// Copier manifest.json
cpSync(join(ROOT, 'manifest.json'), join(PKG, 'manifest.json'));

// Copier icônes
cpSync(join(ROOT, 'public', 'icons'), join(PKG, 'icons'), { recursive: true });

// Créer ZIP
const version = JSON.parse(readFileSync(join(ROOT, 'package.json'), 'utf8')).version;
execSync(`powershell Compress-Archive -Path "${PKG}\\*" -DestinationPath "${join(ROOT, `sc-go-v${version}.zip`)}" -Force`);
```

### 10.2 Installation en mode développeur

1. `npm run build`
2. Ouvrir `chrome://extensions`
3. Activer **Mode développeur**
4. Cliquer **Charger l'extension décompressée**
5. Sélectionner le dossier `dist/`

---

## 11. API BACKEND (ENDPOINT RÉCEPTION)

### 11.1 Endpoints requis

| Méthode | Route | Auth | Description |
|---------|-------|------|-------------|
| `POST` | `/api/admin/aliexpress/import-extension` | Bearer JWT | Reçoit un produit scrapé et l'ajoute en pending |
| `GET` | `/api/admin/aliexpress/status` | Bearer JWT | Vérifie que le token est valide |

### 11.2 Body du POST `import-extension`

```json
{
  "productId": "1234567890",
  "title": "Wireless Bluetooth Headphones",
  "price": { "min": 12.50, "max": 18.90, "currency": "USD" },
  "mainImage": "https://ae01.alicdn.com/...",
  "images": ["url1", "url2", "url3"],
  "description": "...",
  "skus": [
    {
      "skuId": "12345",
      "name": "Noir · M",
      "price": 14.20,
      "stock": 500,
      "image": "https://...",
      "attributes": { "Couleur": "Noir", "Taille": "M" }
    }
  ],
  "storeId": "9876",
  "storeName": "TechStore Official",
  "rating": 4.8,
  "reviewCount": 2340,
  "url": "https://www.aliexpress.com/item/1234567890.html",
  "sellingPrice": 32,
  "source": "extension"
}
```

### 11.3 Réponse attendue

```json
// Succès
{ "pendingId": "uuid-xxx", "status": "pending" }

// Erreur
{ "error": "Token invalide" }
```

### 11.4 Implémentation backend recommandée

```typescript
// POST /api/admin/aliexpress/import-extension
export default async function handler(req, res) {
  // 1. Vérifier JWT
  const token = req.headers.authorization?.replace('Bearer ', '');
  const user = await verifyAdminToken(token);
  if (!user) return res.status(401).json({ error: 'Non autorisé' });

  // 2. Valider le body
  const { productId, title, price, mainImage, images, skus, sellingPrice } = req.body;
  if (!productId || !title) return res.status(400).json({ error: 'Données manquantes' });

  // 3. Vérifier doublon
  const existing = await db.query(
    'SELECT id FROM aliexpress_pending WHERE ali_product_id = $1 AND user_id = $2',
    [productId, user.id]
  );
  if (existing.rows.length > 0) {
    return res.status(409).json({ error: 'Produit déjà importé', pendingId: existing.rows[0].id });
  }

  // 4. Insérer en pending
  const result = await db.query(
    `INSERT INTO aliexpress_pending
     (user_id, ali_product_id, title, price_min, price_max, currency,
      main_image, images, description, skus, store_id, store_name,
      rating, review_count, ali_url, selling_price, source, status)
     VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13,$14,$15,$16,$17,'pending')
     RETURNING id`,
    [user.id, productId, title, price.min, price.max, price.currency,
     mainImage, JSON.stringify(images), req.body.description,
     JSON.stringify(skus), req.body.storeId, req.body.storeName,
     req.body.rating, req.body.reviewCount, req.body.url, sellingPrice, 'extension']
  );

  return res.status(201).json({ pendingId: result.rows[0].id, status: 'pending' });
}
```

### 11.5 Table SQL `aliexpress_pending`

```sql
CREATE TABLE IF NOT EXISTS aliexpress_pending (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id),
  ali_product_id TEXT NOT NULL,
  title TEXT NOT NULL,
  price_min NUMERIC(10,2),
  price_max NUMERIC(10,2),
  currency TEXT DEFAULT 'USD',
  main_image TEXT,
  images JSONB DEFAULT '[]',
  description TEXT,
  skus JSONB DEFAULT '[]',
  store_id TEXT,
  store_name TEXT,
  rating NUMERIC(3,2),
  review_count INTEGER DEFAULT 0,
  ali_url TEXT,
  selling_price NUMERIC(10,2),
  source TEXT DEFAULT 'extension',
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending','approved','rejected','published')),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, ali_product_id)
);

CREATE INDEX idx_ali_pending_user ON aliexpress_pending(user_id);
CREATE INDEX idx_ali_pending_status ON aliexpress_pending(status);
```

---

## 12. CHECKLIST DÉPLOIEMENT

### 12.1 Avant publication

- [ ] Remplacer `host_permissions` dans `manifest.json` avec les domaines de production
- [ ] Générer les icônes PNG (16, 32, 48, 128px) — format PNG obligatoire
- [ ] Configurer l'endpoint backend `POST /api/admin/aliexpress/import-extension`
- [ ] Configurer l'endpoint backend `GET /api/admin/aliexpress/status`
- [ ] Créer la table `aliexpress_pending` en base de données
- [ ] Tester le scraper sur 10+ produits AliExpress différents
- [ ] Tester les 3 niveaux de fallback du scraper
- [ ] Vérifier que `npm run build` produit un `dist/` fonctionnel
- [ ] Charger en mode développeur et tester import complet
- [ ] Vérifier les notifications Chrome

### 12.2 Maintenance du scraper

> ⚠️ **AliExpress modifie régulièrement son DOM et ses structures JS.**

Éléments à surveiller :
1. Les noms de variables globales (`runParams`, `__AE_DATA__`, `_dida_config_`)
2. Les noms des modules (`titleModule`, `priceModule`, `skuModule`, etc.)
3. Les sélecteurs CSS pour le fallback DOM (Niveau 3)
4. Les formats d'URL (`/item/XXXXX.html`)

### 12.3 Sécurité

| Aspect | Mesure |
|--------|--------|
| Token JWT | Stocké dans `chrome.storage.local` (jamais exposé au DOM) |
| HTTPS | Toutes les requêtes via HTTPS uniquement |
| CORS | Le backend doit autoriser l'origin de l'extension Chrome |
| Validation | Le backend doit TOUJOURS revalider les données reçues |
| Rate limiting | Implémenter côté backend (max 60 imports/heure) |

### 12.4 Commandes récapitulatives

```bash
# Installation des dépendances
npm install

# Dev avec hot-reload
npm run dev

# Build production
npm run build

# Packaging ZIP
npm run pack
```

---

## 📐 DESIGN SYSTEM

### Palette de couleurs

| Usage | Couleur | Code |
|-------|---------|------|
| Primary gradient | Violet | `#7c3aed → #4f46e5` |
| Success | Vert | `#059669 → #10b981` |
| Error | Rouge | `#dc2626 → #ef4444` |
| Background (popup/options) | Noir profond | `#0f0f1a` / `#0a0a14` |
| Text principal | Gris clair | `#e2e8f0` |
| Text secondaire | Gris | `#64748b` |
| Accent | Violet clair | `#a78bfa` |

### Typographie

- **Font** : `-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`
- **Poids** : 600 (semi-bold), 700 (bold), 800 (extra-bold)
- **Popup** : largeur fixe `320px`, min-height `200px`

---

> **Fin du skill** — Retour à [PART1](./SKILL_CHROME_EXTENSION_ALIEXPRESS_PART1.md)
