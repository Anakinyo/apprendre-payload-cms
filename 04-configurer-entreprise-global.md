# Apprendre Payload CMS - Episode 4 : configurer l'entreprise avec un Global

## Objectif

Les clients et les missions existent en plusieurs exemplaires : ce sont donc des
collections. L'application ne gère en revanche qu'une seule identité
d'entreprise, avec un nom légal, une adresse, un logo et des préférences de
devis.

Dans cet épisode, nous allons utiliser un `Global` Payload pour représenter cette
configuration unique. Nous améliorerons également l'isolation des tests afin
qu'ils ne puissent jamais modifier les données de développement.

## Ce que nous allons apprendre

- distinguer une collection d'un Global ;
- déclarer et enregistrer un Global Payload ;
- protéger une configuration avec les règles d'accès ;
- mutualiser une validation entre plusieurs modèles ;
- lire et modifier un Global avec la Local API ;
- isoler une base de données de test ;
- choisir entre tests unitaires, tests d'intégration et tests end-to-end.

## Collection ou Global ?

Une collection contient plusieurs documents : plusieurs clients, plusieurs
missions ou plusieurs devis. Chaque document possède son propre identifiant.

Un Global représente un seul document de configuration. Il convient par exemple
à l'identité de l'entreprise, aux réglages du site ou à une page d'accueil
unique.

Nous utiliserons ici le slug `company-settings`. Payload exposera notamment :

- un écran unique dans l'administration ;
- l'API REST `/api/globals/company-settings` ;
- les méthodes `findGlobal` et `updateGlobal` dans la Local API ;
- un type TypeScript généré.

## Mutualiser la validation du SIRET

Le client et l'entreprise possèdent tous les deux un SIRET. Copier la validation
dans plusieurs fichiers rendrait les futures corrections plus difficiles.

Créer `src/fields/siret.ts` :

```ts
import type { FieldHook, TextFieldSingleValidation } from 'payload'

export const normalizeSiret: FieldHook = ({ value }) =>
  typeof value === 'string' ? value.replace(/\s/g, '') : value

export const validateSiret: TextFieldSingleValidation = (value) =>
  !value ||
  /^\d{14}$/.test(value) ||
  'Le SIRET doit contenir exactement 14 chiffres.'
```

Dans la collection `Clients`, importer ces fonctions :

```ts
import { normalizeSiret, validateSiret } from '../fields/siret'
```

Puis les utiliser dans le champ :

```ts
{
  name: 'siret',
  type: 'text',
  label: 'SIRET',
  unique: true,
  hooks: {
    beforeValidate: [normalizeSiret],
  },
  validate: validateSiret,
}
```

Cette petite extraction est justifiée parce qu'elle centralise une véritable
règle métier utilisée à plusieurs endroits.

## Créer le Global

Créer `src/globals/CompanySettings.ts` :

```ts
import type { Access, GlobalConfig } from 'payload'

import { normalizeSiret, validateSiret } from '../fields/siret'

const authenticated: Access = ({ req }) => Boolean(req.user)

export const CompanySettings: GlobalConfig = {
  slug: 'company-settings',
  label: "Paramètres de l'entreprise",
  access: {
    read: authenticated,
    update: authenticated,
  },
  admin: {
    group: 'Configuration',
  },
  fields: [
    {
      name: 'legalName',
      type: 'text',
      label: 'Nom légal',
      required: true,
    },
    {
      name: 'brandName',
      type: 'text',
      label: 'Nom commercial',
    },
    {
      name: 'representativeName',
      type: 'text',
      label: 'Représentant ou signataire',
    },
    {
      name: 'siret',
      type: 'text',
      label: 'SIRET',
      hooks: {
        beforeValidate: [normalizeSiret],
      },
      validate: validateSiret,
      admin: {
        description: 'Les espaces sont retirés automatiquement.',
      },
    },
    {
      name: 'contact',
      type: 'group',
      label: 'Coordonnées',
      fields: [
        {
          name: 'email',
          type: 'email',
          label: 'Adresse e-mail principale',
          required: true,
        },
        {
          name: 'reviewEmail',
          type: 'email',
          label: 'Adresse de validation interne',
          admin: {
            description:
              'Les aperçus de documents seront envoyés à cette adresse.',
          },
        },
        {
          name: 'phone',
          type: 'text',
          label: 'Téléphone',
        },
      ],
    },
    {
      name: 'address',
      type: 'group',
      label: 'Adresse',
      fields: [
        {
          name: 'street',
          type: 'text',
          label: 'Adresse',
        },
        {
          type: 'row',
          fields: [
            {
              name: 'postalCode',
              type: 'text',
              label: 'Code postal',
              admin: { width: '30%' },
            },
            {
              name: 'city',
              type: 'text',
              label: 'Ville',
              admin: { width: '45%' },
            },
            {
              name: 'country',
              type: 'text',
              label: 'Pays',
              defaultValue: 'France',
              admin: { width: '25%' },
            },
          ],
        },
      ],
    },
    {
      name: 'vatNotice',
      type: 'text',
      label: 'Mention relative à la TVA',
      admin: {
        description: 'Par exemple : TVA non applicable, art. 293 B du CGI.',
      },
    },
    {
      name: 'bankDetails',
      type: 'group',
      label: 'Coordonnées bancaires',
      fields: [
        { name: 'bankName', type: 'text', label: 'Banque' },
        { name: 'iban', type: 'text', label: 'IBAN' },
        { name: 'bic', type: 'text', label: 'BIC' },
      ],
    },
    {
      name: 'logo',
      type: 'upload',
      relationTo: 'media',
      label: 'Logo',
    },
    {
      name: 'quoteDefaults',
      type: 'group',
      label: 'Valeurs par défaut des devis',
      fields: [
        {
          name: 'validityDays',
          type: 'number',
          label: 'Durée de validité (jours)',
          required: true,
          defaultValue: 30,
          min: 1,
        },
        {
          name: 'paymentTermsDays',
          type: 'number',
          label: 'Délai de paiement (jours)',
          required: true,
          defaultValue: 15,
          min: 0,
        },
        {
          name: 'paymentMethod',
          type: 'text',
          label: 'Mode de paiement',
          required: true,
          defaultValue: 'Virement',
        },
        {
          name: 'discountPolicy',
          type: 'text',
          label: "Politique d'escompte",
        },
      ],
    },
  ],
}
```

Le Global ne contient aucune donnée réelle dans le code. Les coordonnées, le
SIRET, l'IBAN et les adresses e-mail seront saisis dans l'administration et
stockés dans la base de données locale ou de production.

Cette séparation évite de publier des informations sensibles dans le dépôt Git.

## Enregistrer le Global

Dans `src/payload.config.ts`, importer le Global :

```ts
import { CompanySettings } from './globals/CompanySettings'
```

Puis ajouter la propriété `globals` :

```ts
collections: [Users, Media, Clients, Missions],
globals: [CompanySettings],
```

Régénérer les types :

```powershell
pnpm generate:types
```

Payload ajoute alors une interface `CompanySetting` dans
`src/payload-types.ts`.

## Installer et configurer les tests

Le modèle Blank fournit déjà les outils et plusieurs fichiers de test. Voici
comment les remettre en place dans un projet qui ne les possède pas. Exécuter
ces commandes à la racine de l'application, où se trouve `package.json` :

```powershell
pnpm install
pnpm add -D vitest @vitejs/plugin-react vite-tsconfig-paths jsdom @playwright/test tsx
pnpm add cross-env dotenv
pnpm exec playwright install chromium
```

Pour un clone du dépôt du fil rouge, `pnpm install` suffit : les dépendances
sont déjà déclarées. Installer ensuite Chromium avec la dernière commande.
Conserver `pnpm-lock.yaml` dans Git pour reproduire les versions installées.

Créer `vitest.setup.ts` :

```ts
import 'dotenv/config'
```

Créer `vitest.config.mts` :

```ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./vitest.setup.ts'],
    include: ['tests/int/**/*.int.spec.ts'],
  },
})
```

Vitest reconnaît les fichiers `tests/int/*.int.spec.ts`. `tsconfigPaths` résout
les alias comme `@/payload.config`. Créer un premier fichier
`tests/int/company.int.spec.ts` :

```ts
import { getPayload, type Payload } from 'payload'
import config from '@/payload.config'
import { beforeAll, describe, expect, it } from 'vitest'

let payload: Payload

describe('Configuration entreprise', () => {
  beforeAll(async () => {
    payload = await getPayload({ config })
  })

  it('enregistre le nom et normalise le SIRET', async () => {
    const settings = await payload.updateGlobal({
      slug: 'company-settings',
      data: {
        legalName: 'Entreprise de test',
        siret: '123 456 789 00012',
        contact: { email: 'contact@example.com' },
      },
    })
    expect(settings.legalName).toBe('Entreprise de test')
    expect(settings.siret).toBe('12345678900012')
  })
})
```

Utiliser exclusivement la base de test configurée ci-dessous. Pour les
collections, supprimer les documents créés dans un `afterAll` en utilisant
leurs identifiants. Le dépôt regroupe déjà ces cas dans
`tests/int/api.int.spec.ts`; ne pas ajouter deux suites qui modifient le même
Global simultanément. La Local API contourne les permissions par défaut :
utiliser `overrideAccess: false` pour tester explicitement les accès.

## Tester le Global avec la Local API

La Local API permet d'utiliser Payload directement côté serveur, sans requête
HTTP. Pour modifier le Global :

```ts
const updatedSettings = await payload.updateGlobal({
  slug: 'company-settings',
  data: {
    legalName: 'Entreprise de test',
    siret: '123 456 789 00012',
    contact: {
      email: 'contact@example.com',
      reviewEmail: 'validation@example.com',
    },
    address: {
      country: 'France',
    },
    quoteDefaults: {
      paymentMethod: 'Virement',
      paymentTermsDays: 15,
      validityDays: 30,
    },
  },
})
```

Puis le relire :

```ts
const settings = await payload.findGlobal({
  slug: 'company-settings',
  depth: 0,
})

expect(updatedSettings.siret).toBe('12345678900012')
expect(settings.legalName).toBe('Entreprise de test')
expect(settings.contact.email).toBe('contact@example.com')
expect(settings.quoteDefaults.validityDays).toBe(30)
```

Ce test protège la normalisation du SIRET, la structure des groupes et les
valeurs nécessaires aux futurs devis.

## Isoler la base de test

Un test ne doit jamais modifier les données utilisées pendant le développement.
Créer ou compléter `test.env` :

```dotenv
NODE_OPTIONS="--no-deprecation --no-experimental-strip-types"
PAYLOAD_SECRET=integration-test-secret-not-for-production
DATABASE_URL=file:./payload-test.db
```

Modifier la commande de test dans `package.json` :

```json
{
  "scripts": {
    "test": "pnpm run test:int && pnpm run test:e2e",
    "test:e2e": "cross-env NODE_OPTIONS=\"--no-deprecation --import=tsx/esm\" DOTENV_CONFIG_PATH=./test.env playwright test --config=playwright.config.ts",
    "test:int": "cross-env NODE_OPTIONS=--no-deprecation DOTENV_CONFIG_PATH=./test.env vitest run --config ./vitest.config.mts"
  }
}
```

Enfin, ignorer la base temporaire dans `.gitignore` :

```gitignore
/payload-test.db*
```

Les tests peuvent désormais créer, modifier et supprimer leurs propres données
sans toucher à `payload-archiviste.db`.

Les tests Playwright doivent également démarrer leur propre serveur. Dans
`playwright.config.ts`, utiliser par exemple le port `3001` :

```ts
import { defineConfig, devices } from '@playwright/test'
import 'dotenv/config'

const testServerURL = 'http://127.0.0.1:3001'

export default defineConfig({
  testDir: './tests/e2e',
  reporter: 'html',
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
  use: { baseURL: testServerURL },
  webServer: {
    command: 'pnpm exec next dev --hostname 127.0.0.1 --port 3001',
    reuseExistingServer: false,
    url: testServerURL,
  },
})
```

Créer un premier fichier `tests/e2e/login.e2e.spec.ts` :

```ts
import { expect, test } from '@playwright/test'

test('affiche le formulaire de connexion', async ({ page }) => {
  await page.goto('/admin/login')
  await expect(page.locator('#field-email')).toBeVisible()
  await expect(page.locator('#field-password')).toBeVisible()
})
```

Cet exemple suppose que la base possède déjà un utilisateur. Sur une base
neuve, Payload propose d'abord sa création. Le dépôt fournit
`tests/helpers/seedUser.ts` et `tests/helpers/login.ts` pour créer puis
supprimer un utilisateur de test et se connecter de façon reproductible.
Ne pas utiliser de véritables identifiants dans ces fichiers.

Le serveur de test reçoit `DOTENV_CONFIG_PATH=./test.env` et utilise donc la
base dédiée. Next.js n'autorise qu'un serveur de développement par projet : il
faut arrêter temporairement `pnpm dev` avant de lancer Playwright, puis le
redémarrer après les tests.

Au premier lancement, installer le navigateur de test si Playwright le demande :

```powershell
pnpm exec playwright install chromium
```

## Lancer et lire les résultats

Arrêter `pnpm dev` avec `Ctrl+C` avant les tests navigateur. À la racine du
projet, exécuter :

```powershell
pnpm test:int
pnpm test:e2e
pnpm test
pnpm exec playwright show-report
```

La première commande vérifie les modèles et les hooks; la deuxième pilote
le navigateur; `pnpm test` enchaîne les suites. Vitest affiche les assertions
en échec. Playwright crée un rapport ouvrable avec la dernière commande.
Une commande réussie retourne un code zéro; corriger les échecs avant publication.

Ajouter `/playwright-report/` et `/test-results/` au `.gitignore`, ainsi que
la base temporaire. Versionner les tests et leur configuration, mais pas
les rapports ni la base. Relancer `pnpm dev` après les tests.

Si « No test files found » apparaît, vérifier le nom, le dossier et `include`.
Si Chromium est absent, relancer sa commande d'installation. Si Next.js
annonce qu'un serveur existe déjà, arrêter le serveur du projet avant de
réessayer. Des migrations explicites seront nécessaires avant la production.

## Quels tests utiliser ?

### Tests unitaires

Ils vérifient une petite fonction isolée. Ils seront utiles lorsque nous aurons
des calculs de montants, de dates ou de numérotation suffisamment complexes pour
être testés sans démarrer Payload.

### Tests d'intégration

Ils démarrent Payload avec une base dédiée et vérifient plusieurs composants
ensemble : configuration, hooks, validations, relations et stockage. C'est le
niveau principal de notre projet actuel, car nos premiers risques se trouvent
précisément dans ces interactions.

### Tests end-to-end

Ils pilotent l'application dans un navigateur comme le ferait une personne. Ils
sont plus lents, mais précieux pour les parcours essentiels : connexion,
création d'un devis, aperçu PDF et validation interne.

Notre stratégie sera donc :

- tests unitaires pour les calculs métier purs ;
- tests d'intégration pour les modèles Payload et leurs règles ;
- quelques tests end-to-end ciblés pour les parcours critiques ;
- `pnpm build` pour vérifier TypeScript et la compilation de production.

## Vérifier dans l'administration

Démarrer l'application :

```powershell
pnpm dev
```

Ouvrir <http://localhost:3000/admin>, puis sélectionner
**Configuration > Paramètres de l'entreprise**.

Vérifier que :

- il existe un seul écran de configuration, sans liste de documents ;
- le SIRET accepte les espaces puis les retire ;
- le logo utilise la médiathèque ;
- les valeurs par défaut du devis sont préremplies ;
- un visiteur non connecté ne peut pas lire ces informations via l'API.

Exécuter ensuite :

```powershell
pnpm generate:types
pnpm lint
pnpm test
pnpm build
```

## Prochaine étape

Dans l'épisode suivant, nous créerons la collection `Devis`, ses relations vers
le client et la mission, ainsi que ses lignes de prestations. Nous préparerons
la structure avant d'ajouter les calculs automatiques dans l'épisode suivant.
