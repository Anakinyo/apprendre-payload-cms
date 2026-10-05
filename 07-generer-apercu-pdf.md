# Apprendre Payload CMS - Episode 7 : figer un aperçu et générer un PDF

## Objectif

Depuis un devis enregistré, générer un aperçu PDF réservé aux utilisateurs
connectés. Conserver les données et le fichier de chaque aperçu pour que le
document reste identique après une correction du devis ou du client.

Chaque PDF porte « Aperçu non validé ». La référence `AP-…` est un identifiant
interne, pas un numéro commercial de devis. La validation et l'envoi seront
traités dans les épisodes suivants. Aucun e-mail n'est envoyé dans cet épisode.

## 1. Installer les bibliothèques

À la racine de l'application :

```powershell
pnpm add @react-pdf/renderer@4.3.1 pdf-lib@1.17.1 --save-exact
```

React PDF transforme des composants React en un document PDF côté serveur.
`pdf-lib` ajoute l'en-tête, le pied de page et les numéros après la mise en page. Cela évite qu'un
texte dynamique de pagination modifie lui-même la hauteur du document.
Les versions et le fichier `pnpm-lock.yaml` sont ceux vérifiés dans le fil rouge.
Pour reprendre directement le dépôt, utiliser simplement `pnpm install`.

Le rendu tourne dans Node.js. Il n'est pas importé dans le composant du bouton
d'administration. La documentation officielle décrit
[le découpage en pages](https://react-pdf.org/docs/v4/advanced/page-wrapping).

## 2. Définir un instantané des données

Créer `src/domain/quoteSnapshot.ts`. La fonction `buildQuoteSnapshot` reçoit
le devis, le client, l'entreprise et une mission facultative. Elle retourne
un nouvel objet composé uniquement des valeurs utiles au document :

- identité et adresse de l'entreprise et du client ;
- signataire du client et coordonnées utiles ;
- intitulé, date, objet et mission ;
- prestations, quantités et prix ;
- totaux recalculés avec la fonction du chapitre 6 ;
- taux, mention TVA, validité et paiement ;
- coordonnées bancaires configurées ;
- version du modèle, ici `templateVersion: 1`.

Les notes internes, les données de trajet et la référence d'import ne sont
pas copiées. Construire explicitement cet objet permet de vérifier ce que
le document peut divulguer; une copie complète du client serait trop large.

```ts
const amounts = calculateQuoteAmounts(quote.lines, quote.taxRateBps ?? 0)
const snapshot = {
  templateVersion: 1,
  sourceQuoteId: quote.id,
  title: quote.title,
  client: {
    name: client.legalName || client.displayName,
    // Copier les valeurs de l'adresse, pas une relation vers le client.
  },
  totalCents: amounts.totalCents,
}
```

Ce fragment montre le principe, pas l'objet complet. Reprendre le fichier
[quoteSnapshot.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/domain/quoteSnapshot.ts)
pour ses champs et son type `QuoteSnapshot`.

L'entreprise doit avoir un nom légal et une adresse e-mail. Le serveur
revérifie également que la mission appartient au client : la mission a pu
être modifiée depuis l'enregistrement du devis. Les autres coordonnées restent
facultatives pour cet aperçu; sa présence ne signifie pas que le devis est validé.

## 3. Créer la collection des aperçus

Créer `src/collections/QuotePreviews.ts`, avec le slug `quote-previews` :

| Champ | Type | Rôle |
| --- | --- | --- |
| `quote` | relation requise | Devis source |
| `reference` | texte unique | Identité interne créée sur le serveur |
| `snapshot` | JSON | Données figées et version du modèle |
| `pdfBase64` | texte masqué | Octets du PDF encodés en base64 |
| `openPDF` | UI | Lien vers le PDF authentifié |

Un hook `beforeValidate` relit les documents sources, construit l'instantané,
génère une référence avec `randomUUID`, produit le PDF et remplace les valeurs
fournies par la requête. Une personne ne peut donc pas publier un faux
instantané en envoyant son propre JSON ou son propre `pdfBase64`.

```ts
const snapshot = buildQuoteSnapshot(quote, client, company, mission)
const reference = `AP-${quote.id}-${randomUUID()}`
const { renderQuotePDF } = await import('../pdf/QuoteDocument')
const pdf = await renderQuotePDF(snapshot, reference)
return { quote: quote.id, reference, snapshot, pdfBase64: pdf.toString('base64') }
```

Les champs calculés ne sont pas obligatoires dans les données de création de
la Local API : le hook garantit qu'ils sont construits avant toute insertion.
La création et la lecture exigent une connexion. La modification et la
suppression sont interdites dans l'API et dans l'administration. Le hook refuse
aussi une mise à jour par la Local API. Les tests suppriment leurs fixtures
avec l'accès serveur privilégié, comme les autres données de test.

Pour ce MVP SQLite, stocker les petits fichiers dans la base évite un lien
public vers un fichier et conserve exactement les octets produits. Le base64
occupe environ un tiers de plus que le fichier binaire. Pour un volume élevé,
prévoir un stockage privé avec sauvegardes et conservation des versions.
Le JSON figé est conservé, mais ouvrir un aperçu sert le PDF enregistré sans
le régénérer : même un changement ultérieur du modèle ne réécrit pas ce fichier.

Enregistrer la collection dans `src/payload.config.ts` puis générer les types :

```ts
import { QuotePreviews } from './collections/QuotePreviews'

collections: [Users, Media, Clients, Missions, Quotes, QuotePreviews],
```

```powershell
pnpm generate:types
```

Le code complet de la collection est dans
[QuotePreviews.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/collections/QuotePreviews.ts).

## 4. Construire le modèle PDF

Créer `src/pdf/QuoteDocument.tsx`. Utiliser les composants de React PDF :

```tsx
import { Document, Page, Text, View, renderToBuffer } from '@react-pdf/renderer'

function QuoteDocument({ snapshot }) {
  return <Document>
    <Page size="A4" wrap>
      <Text>APERÇU NON VALIDÉ</Text>
      <Text>Devis</Text>
      <Text>{snapshot.title}</Text>
      {/* Identités, prestations, totaux et conditions */}
    </Page>
  </Document>
}
```

Ce schéma doit être complété avec le typage `QuoteSnapshot` et les styles du
[modèle complet](https://github.com/Anakinyo/payload-archiviste/blob/main/src/pdf/QuoteDocument.tsx).
`Document` et `Page` ne sont pas des éléments HTML. Une feuille `StyleSheet`
définit les marges A4, les colonnes et la typographie.

Convertir les centimes en euros à l'affichage avec `Intl.NumberFormat('fr-FR')`.
Les espaces insécables de ce formateur sont normalisés pour les polices PDF.
Les dates utilisent UTC pour éviter qu'un changement de fuseau décale la date.

Les blocs d'une ligne sont indivisibles avec `wrap={false}`. Une description
longue est découpée en plusieurs blocs bornés; les blocs suivants portent
« suite » et les prix ne sont affichés qu'une fois. Le total reste groupé.
Les marges réservent l'espace de l'en-tête et du pied de page; `pdf-lib` les
ajoute ensuite à chaque page, ce qui évite leur découpage par le moteur de mise en page.

Après le rendu, ajouter la numérotation avec `pdf-lib` :

```ts
const buffer = await renderToBuffer(<QuoteDocument snapshot={snapshot} />)
const source = await PDFDocument.load(buffer)
const pdf = await PDFDocument.create()
const embeddedPages = await pdf.embedPages(source.getPages())
const font = await pdf.embedFont(StandardFonts.Helvetica)
embeddedPages.forEach((embedded, index) => {
  const page = pdf.addPage([embedded.width, embedded.height])
  page.drawPage(embedded)
  page.drawText(`Page ${index + 1} / ${embeddedPages.length}`, {
  x: 38, y: 22, size: 9, font,
  })
})
return Buffer.from(await pdf.save())
```

Les pages du moteur sont incorporées dans de nouvelles pages : leur état de
découpage graphique ne peut ainsi pas masquer les en-têtes ajoutés ensuite.

Cette première version est un modèle générique : elle ne reproduit pas encore
une charte graphique particulière et n'intègre pas le logo ou les photos.
Le PDF reste un aperçu; signature, numérotation commerciale et contrôle des
mentions seront traités avec la validation.

## 5. Ajouter deux endpoints Payload

Dans la collection Devis, ajouter `POST /:id/preview`, accessible via
`/api/quotes/:id/preview`. Le handler :

1. exige `req.user` ;
2. valide l'identifiant et l'existence du devis ;
3. crée un aperçu avec `req` et `overrideAccess: false` ;
4. retourne l'identifiant et la référence, avec un statut 201.

```ts
const preview = await req.payload.create({
  collection: 'quote-previews', data: { quote: id }, req, overrideAccess: false,
})
return Response.json({ id: preview.id, reference: preview.reference }, { status: 201 })
```

Dans les aperçus, ajouter `GET /:id/pdf`. Le fichier est accessible via
`/api/quote-previews/:id/pdf` uniquement après authentification. La réponse
contient `Content-Type: application/pdf`, `Content-Disposition: inline` et
`Cache-Control: private, no-store`.

Le champ binaire possède `access.read: () => false` : il est absent des réponses
REST ordinaires. Le handler PDF le relit avec l'accès serveur privilégié après
le contrôle d'authentification. Ce choix correspond à notre application à une
seule entreprise, où tous les utilisateurs connectés partagent les documents.
Des permissions par rôle ou par entreprise exigeraient un contrôle plus précis.

Le lien du PDF n'est pas un lien public à envoyer au client. Il exige la session
de l'application. Un envoi après validation sera ajouté dans un autre épisode.

## 6. Ajouter l'action d'administration

Créer `src/components/QuotePreviewActions.tsx` avec `'use client'`. Le composant
utilise `useDocumentInfo`, `useFormModified` et `useConfig` de Payload.

Le bouton appelle l'endpoint POST. Il est désactivé pendant la génération et
tant que des modifications ne sont pas enregistrées. Une erreur affiche un
message; une réussite affiche un lien « Ouvrir le PDF ». L'ouverture se fait
par un lien après la génération, ce qui évite le blocage d'une fenêtre ouverte
automatiquement après une opération asynchrone.

Dans les champs de Devis et d'Aperçus, déclarer un champ UI :

```ts
{
  name: 'previewActions', type: 'ui',
  admin: { components: { Field: '/components/QuotePreviewActions#QuotePreviewActions' } },
}
```

Dans la collection des aperçus, le même composant affiche le lien vers le PDF
enregistré. Le champ UI ne stocke pas de donnée métier. Générer l'import map :

```powershell
pnpm generate:importmap
```

Reprendre le composant complet dans
[QuotePreviewActions.tsx](https://github.com/Anakinyo/payload-archiviste/blob/main/src/components/QuotePreviewActions.tsx).

## 7. Vérifier avec les tests

L'installation et le lancement des tests sont détaillés au
[chapitre 4](./04-configurer-entreprise-global.md#installer-et-configurer-les-tests),
et les tests unitaires au [chapitre 6](./06-calculer-montants.md#5-ajouter-les-tests-unitaires).
Cette étape ajoute :

- des tests d'instantané : copie des valeurs, absence des notes internes,
  contrôle d'une configuration incomplète ;
- un test du PDF : format A4, titre et pagination d'un document long ;
- un test Payload : données forgées remplacées, ancien instantané et octets
  inchangés après modification des sources, nouvel aperçu distinct ;
- un parcours navigateur : génération depuis un devis sur mobile, fichier
  PDF valide, accès anonyme refusé et bouton désactivé après modification.

Les tests du serveur Payload et du PDF utilisent l'environnement Node.js,
comme l'application. Ajouter en première ligne de `tests/int/api.int.spec.ts`
et de `tests/unit/quotePDF.unit.spec.ts` :

```ts
// @vitest-environment node
```

Cela évite notamment des incompatibilités entre les objets binaires de Node.js
et ceux d'un DOM simulé. Les tests de composants peuvent conserver `jsdom`.

Le démarrage E2E prépare maintenant le schéma SQLite avant de lancer le serveur
et les fixtures. Cela évite plusieurs initialisations concurrentes du schéma.
Créer `scripts/prepare-e2e.ts` :

```ts
import 'dotenv/config'
import { rmSync } from 'node:fs'
import path from 'node:path'
import { getPayload } from 'payload'
import config from '../src/payload.config'

if (process.env.DATABASE_URL !== 'file:./payload-test.db') {
  throw new Error('E2E preparation requires the dedicated test database.')
}
const database = path.resolve('payload-test.db')
for (const suffix of ['', '-wal', '-shm']) rmSync(`${database}${suffix}`, { force: true })
const payload = await getPayload({ config })
await payload.destroy()
```

Dans l'adaptateur SQLite, ajouter :

```ts
push: process.env.PAYLOAD_TEST_SCHEMA_READY !== '1',
```

Puis remplacer `test:e2e` dans `package.json` par :

```json
"test:e2e": "cross-env NODE_OPTIONS=--no-deprecation DOTENV_CONFIG_PATH=./test.env tsx scripts/prepare-e2e.ts && cross-env NODE_OPTIONS=\"--no-deprecation --import=tsx/esm\" DOTENV_CONFIG_PATH=./test.env PAYLOAD_TEST_SCHEMA_READY=1 playwright test --config=playwright.config.ts"
```

La préparation réinitialise uniquement la base de test jetable puis active la
synchronisation du schéma; le serveur et les fixtures
utilisent ensuite ce schéma existant. Cette option est réservée aux tests de
développement, pas une stratégie de migration de production.
Ne jamais mettre de données à conserver dans `payload-test.db`. Le script refuse
toute autre valeur de `DATABASE_URL`; lancer la commande à la racine du projet.

Arrêter `pnpm dev` puis exécuter :

```powershell
pnpm test
pnpm lint
pnpm build
```

## 8. Vérifier visuellement le PDF

Un fichier valide peut contenir du texte hors de la page ou un tableau coupé.
Générer les exemples fictifs avec le script du dépôt :

```powershell
pnpm exec tsx scripts/verify-quote-pdf.ts
```

Il crée un devis court et un devis long avec 60 prestations dans `tmp/pdfs`,
dossier ignoré par Git. Ouvrir les fichiers et inspecter toutes les pages :
accents, euros, description longue, colonnes, totaux, pied de page et numéros.

Pour automatiser une partie du contrôle, installer Python et les bibliothèques
optionnelles, puis exécuter :

```powershell
python -m pip install pdfplumber pypdf
python scripts/check-quote-pdf.py
```

Le script vérifie les limites de page, les numéros et la présence de toutes
les prestations et des totaux. Il complète l'inspection visuelle. Python sert
uniquement au contrôle des exemples; l'application génère ses PDFs avec Node.js.
Avec Poppler installé, on peut aussi rendre les pages en images :

```powershell
pdftoppm -png tmp/pdfs/quote-short.pdf tmp/pdfs/short
```

## 9. Essayer le parcours

Relancer `pnpm dev`, compléter la configuration de l'entreprise et enregistrer
un devis. Cliquer « Générer un aperçu PDF » puis « Ouvrir le PDF ».
Modifier le client ou le devis, enregistrer et générer un nouvel aperçu.
Retrouver les deux fichiers dans **Aperçus de devis** : le premier conserve
ses données d'origine, le second reflète les corrections.

## Prochaine étape

Ajouter la validation interne d'une version précise, avec date et auteur de
la décision. Un PDF téléchargé reste pour l'instant un aperçu non validé.
