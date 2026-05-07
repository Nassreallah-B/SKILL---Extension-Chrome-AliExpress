# Extension Chrome AliExpress — Documentation Technique

## Objectif

Cette documentation décrit l’état réel actuel d’une extension Chrome (Manifest V3) pour:

- détecter les fiches produit AliExpress,
- extraire les données produit et variantes,
- envoyer ces données à un service d’import sécurisé,
- piloter l’import depuis un bouton injecté et un popup.

## Fonctionnalités exactes actuelles

- Détection automatique des pages produit AliExpress.
- Scraping en 3 niveaux de fallback:
  - niveau 1: données globales JavaScript,
  - niveau 2: blocs JSON embarqués,
  - niveau 3: fallback DOM.
- Extraction variantes avancée:
  - attributs (ex: couleur, taille),
  - prix variante,
  - quantité réelle,
  - disponibilité réelle,
  - SKU fournisseur (`sku`, `skuCode`, `sku_code`) quand disponible.
- Injection d’un bouton flottant sur la page produit.
- États du bouton:
  - `idle`,
  - `loading`,
  - `success`,
  - `error`.
- Toasts de feedback en page.
- Messaging extension complet:
  - `PRODUCT_DETECTED`,
  - `GET_PRODUCT`,
  - `IMPORT_PRODUCT`,
  - `CHECK_AUTH`,
  - `GET_CONFIG`.
- Import réseau via `background service worker` avec en-tête `Authorization: Bearer <token>`.
- Notification système après import réussi.
- Badge extension:
  - positionné à `1` quand un produit est détecté,
  - reset lors de navigation.
- Popup avec 3 états:
  - non configuré,
  - produit détecté,
  - hors page produit.
- Options avancées:
  - stockage local de configuration,
  - collage token,
  - affichage/masquage token,
  - slider de marge,
  - test de connexion.
- Build multi-entry Vite + TypeScript.
- Packaging ZIP automatisé.

## Architecture technique

1. Content Script (`src/content/content.ts`)
   - attend le DOM critique (`waitForProductReady`),
   - déclenche `scrapeAliExpressProduct()`,
   - injecte le bouton,
   - expose `GET_PRODUCT`,
   - envoie `PRODUCT_DETECTED` au background.

2. Scraper (`src/content/scraper.ts`)
   - cascade `window data -> script tags -> DOM`,
   - normalise produit + variantes,
   - récupère ID produit depuis le chemin,
   - construit un payload exploitable côté serveur.

3. Background (`src/background/bg.ts`)
   - orchestre les messages runtime,
   - exécute l’import réseau,
   - exécute le check auth,
   - fournit la config au popup,
   - gère notifications et badge.

4. Popup (`src/popup/popup.ts`)
   - sélectionne l’état UI,
   - affiche preview produit,
   - déclenche import manuel,
   - affiche feedback résultat.

5. Options (`src/options/options.ts`)
   - charge/sauvegarde configuration,
   - gère token et marge,
   - déclenche un test de connexion.

## Structure projet

```text
sc-go/
├── manifest.json
├── package.json
├── tsconfig.json
├── vite.config.ts
├── scripts/
│   └── pack.mjs
├── public/
│   └── icons/
├── src/
│   ├── shared/
│   │   └── types.ts
│   ├── background/
│   │   └── bg.ts
│   ├── content/
│   │   ├── scraper.ts
│   │   ├── content.ts
│   │   └── content.css
│   ├── popup/
│   │   ├── popup.html
│   │   ├── popup.ts
│   │   └── popup.css
│   └── options/
│       ├── options.html
│       ├── options.ts
│       └── options.css
└── dist/
```

## Flux de travail détaillé

1. L’utilisateur ouvre une fiche produit AliExpress.
2. Le content script attend les marqueurs DOM (titre + prix).
3. Le scraper tente l’extraction niveau 1.
4. Si échec partiel, fallback niveau 2, puis niveau 3.
5. Si produit valide:
   - bouton injecté,
   - message `PRODUCT_DETECTED` envoyé.
6. L’utilisateur clique “Importer”.
7. Le content script envoie `IMPORT_PRODUCT` au background.
8. Le background:
   - lit la config locale,
   - calcule `sellingPrice` depuis `priceMin * margin`,
   - envoie le payload au service d’import,
   - renvoie succès/erreur.
9. Le content script met à jour l’état visuel + feedback.
10. Le popup reflète l’état et permet un import alternatif.

## Contrat de données (résumé)

- Produit:
  - `productId`, `title`, `priceMin`, `priceMax`, `currency`,
  - `mainImage`, `images`, `description`,
  - `storeId`, `storeName`, référence boutique,
  - `rating`, `reviewCount`,
  - référence page source, `scrapedAt`.
- Variantes:
  - `skuId`,
  - `sku`, `skuCode`, `sku_code`,
  - `name`, `price`,
  - `stock`, `quantity`, `available`,
  - `image`,
  - `attributes`.

## Scripts disponibles

Depuis le dossier `sc-go`:

```bash
npm install
npm run dev
npm run build
npm run pack
```

Notes:

- `dev`: build en watch.
- `build`: bundle extension + copie explicite de `content.css`.
- `pack`: génère un ZIP versionné depuis les artefacts build.

## Installation extension (mode développeur)

1. Exécuter `npm run build`.
2. Ouvrir la page Extensions de Chrome.
3. Activer le mode développeur.
4. Charger l’extension non empaquetée.
5. Sélectionner le dossier `dist`.

## Points de robustesse

- Fallback multi-source pour résister aux changements de structure AliExpress.
- Observer DOM pour réagir aux navigations dynamiques.
- Gestion explicite des erreurs réseau/auth.
- Feedback utilisateur immédiat (état bouton + toast + popup).

## Contraintes et limites actuelles

- La qualité du scraping dépend de la stabilité de la structure source.
- Certaines fiches peuvent fournir des données partielles.
- Le fallback DOM ne reconstruit pas toujours les variantes complètes.

## Sécurité et conformité documentaire

- Aucun secret en clair.
- Aucun token réel.
- Aucune mention de marque/projet interne.
- Aucune adresse externe dans ce document.
