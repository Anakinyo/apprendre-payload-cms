# Apprendre Payload CMS - Episode 8 : valider une version précise

## Objectif

Permettre à une personne connectée de relire un aperçu PDF puis de l'approuver
ou de le refuser. Conserver la décision, son auteur, sa date et l'identité du
fichier relu. La décision est interne : aucun document n'est envoyé au client.

## 1. Choisir ce qui est validé

Le devis est modifiable. Son aperçu du chapitre 7 est figé. La décision doit
donc porter sur **l'aperçu**, pas sur le devis courant.

| Version | Décision | Après correction du devis |
| --- | --- | --- |
| Aperçu A | Approuvée en interne | Reste approuvée, avec son ancien PDF |
| Aperçu B | Refusée avec un motif | Reste refusée |
| Nouvel aperçu C | Aucune décision | Doit être relu séparément |

Il n'y a pas de champ « devis validé » propagé automatiquement. Une nouvelle
version n'hérite jamais de l'approbation d'une version précédente.

Pour ce premier parcours, une version reçoit une seule décision définitive.
Après un refus ou une décision à corriger, générer un nouvel aperçu puis le
relire. Une révocation ou un historique de décisions successives demanderait
un autre modèle; ce n'est pas un simple changement du statut existant.

## 2. Préparer le projet

Partir de l'application à la fin du chapitre 7. Arrêter le serveur avant les
tests et les modifications du schéma. Node.js fournit `createHash` dans
`node:crypto`. Pour la mise à jour d'une base SQLite existante, déclarer aussi
explicitement le client SQLite déjà utilisé par l'adaptateur Payload :

```powershell
pnpm install
pnpm add @libsql/client@0.14.0 --save-exact
```

Créer les fichiers suivants, dont les versions complètes sont liées ci-dessous :

- `src/domain/quoteDecision.ts` : validation et empreinte du PDF ;
- `src/collections/QuoteDecisions.ts` : stockage des décisions ;
- `src/endpoints/quoteReview.ts` : lecture et enregistrement ;
- `src/components/QuoteReviewActions.tsx` : interface de relecture.

## 3. Isoler les règles métier

Dans [quoteDecision.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/domain/quoteDecision.ts),
la fonction `validateQuoteDecision` exige :

- `reviewed === true`, pas la chaîne de caractères `"true"` ;
- une décision `approved` ou `rejected` ;
- un motif non vide pour un refus ;
- un commentaire de 2000 caractères au maximum.

```ts
export function validateQuoteDecision(data: {
  decision?: unknown
  reviewed?: unknown
  reason?: unknown
}) {
  if (data.reviewed !== true) throw new Error('La relecture du PDF doit être confirmée.')
  if (data.decision !== 'approved' && data.decision !== 'rejected') {
    throw new Error('Décision invalide.')
  }
  const reason = typeof data.reason === 'string' ? data.reason.trim() : ''
  if (reason.length > 2000) throw new Error('Le commentaire est limité à 2000 caractères.')
  if (data.decision === 'rejected' && !reason) throw new Error('Un motif de refus est requis.')
  return { decision: data.decision, reviewed: true as const, reason }
}
```

L'empreinte SHA-256 est calculée sur les **octets décodés** du PDF enregistré,
pas sur la représentation base64 ni sur le devis actuel :

```ts
import { createHash } from 'node:crypto'

export function quotePDFHash(base64: string) {
  const bytes = Buffer.from(base64, 'base64')
  if (bytes.subarray(0, 5).toString() !== '%PDF-') {
    throw new Error('PDF indisponible ou invalide.')
  }
  return createHash('sha256').update(bytes).digest('hex')
}
```

L'empreinte identifie les octets associés à la décision. Elle ne prouve pas
qu'une personne a effectivement lu le PDF : la case à cocher reste une
confirmation explicite de cette personne. Ce mécanisme n'est pas une signature
électronique ni un journal inviolable face à l'administrateur de la base.

## 4. Créer la collection des décisions

Créer [QuoteDecisions.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/collections/QuoteDecisions.ts),
avec le slug `quote-decisions` :

| Champ | Type | Origine |
| --- | --- | --- |
| `preview` | relation requise et unique vers `quote-previews` | Version choisie |
| `decision` | select | Approbation ou refus |
| `reviewed` | checkbox | Confirmation de relecture |
| `reason` | textarea | Commentaire / motif |
| `previewReference` | text | Copie de la référence de l'aperçu |
| `decidedAt` | date | Horloge du serveur |
| `decidedBy` | relation vers `users` | Utilisateur de la requête |
| `reviewerEmail` | email | Identité copiée lors de la décision |
| `pdfSHA256` | text | Empreinte du PDF enregistré |

`preview` porte `unique: true`. La contrainte de base de données protège contre
deux décisions concurrentes sur la même version. Vérifier uniquement
« aucune décision n'existe » avant une insertion ne suffit pas : deux requêtes
peuvent faire cette lecture avant que l'une d'elles ait enregistré sa décision.

Les permissions sont simples :

```ts
access: {
  create: ({ req }) => Boolean(req.user),
  read: ({ req }) => Boolean(req.user),
  update: () => false,
  delete: () => false,
},
```

Comme dans les chapitres précédents, l'application concerne une seule entreprise
et tous les utilisateurs connectés peuvent relire les documents. Une séparation
par rôle ou par entreprise devra être appliquée dans les permissions **et**
dans les endpoints si ce besoin apparaît.

### Ne pas faire confiance aux champs en lecture seule

`admin.readOnly` protège l'ergonomie, pas les données envoyées par un programme.
Le hook `beforeValidate` reconstruit les métadonnées :

```ts
if (operation !== 'create' || !data) {
  throw new Error('Une décision enregistrée est immuable.')
}
if (!req.user) throw new Error('Authentification requise pour décider.')
const values = validateQuoteDecision(data)
const id = typeof data.preview === 'object' ? data.preview?.id : data.preview
const preview = await req.payload.findByID({
  collection: 'quote-previews', id, depth: 0, req,
})
if (!preview.pdfBase64) throw new Error('PDF indisponible.')
return {
  ...values,
  preview: preview.id,
  previewReference: preview.reference,
  decidedAt: new Date().toISOString(),
  decidedBy: req.user.id,
  reviewerEmail: req.user.email,
  pdfSHA256: quotePDFHash(preview.pdfBase64),
}
```

Ce fragment est le corps du hook; reprendre le fichier complet pour les imports,
les types et les champs. Il ignore une date, un auteur ou une empreinte forgés.
La Local API relit le champ PDF masqué avec son accès serveur privilégié, après
avoir exigé l'authentification. Ce choix dépend de notre modèle à une entreprise.

Le hook interdit aussi une mise à jour privilégiée. La suppression privilégiée
reste possible pour les fixtures de tests et les opérations d'administration
serveur : les droits applicatifs ne remplacent pas la protection de la base.

L'adresse e-mail copiée reste disponible si l'identité de l'utilisateur change.
Elle n'est pas publiée; sa conservation fait partie des données internes à
prendre en compte dans les sauvegardes et la politique de conservation.

## 5. Enregistrer la collection et les types

Dans `src/payload.config.ts` :

```ts
import { QuoteDecisions } from './collections/QuoteDecisions'

collections: [Users, Media, Clients, Missions, Quotes, QuotePreviews, QuoteDecisions],
```

```powershell
pnpm generate:types
```

L'adaptateur SQLite synchronise le schéma en développement. Pour une installation
de production, utiliser des migrations préparées et des sauvegardes; ce chapitre
ne remplace pas la préparation au déploiement du parcours prévu.

### Mettre à jour la base du chapitre 7 sans perdre ses données

Une base neuve du chapitre 8 et une base existante du chapitre 7 ne suivent pas
le même chemin. Dans les versions du fil rouge, la synchronisation automatique
SQLite peut échouer en reconstruisant les relations des verrous internes
(`quote_decisions_id` manquant ou index déjà existant). **Ne pas supprimer la base
de l'application** pour résoudre ce problème.

Reprendre le script
[upgrade-chapter8.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/scripts/upgrade-chapter8.ts)
et son helper
[upgradeChapter8SQLite.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/database/upgradeChapter8SQLite.ts).
Le script est réservé à la base locale `file:./payload-archiviste.db` du chapitre 7
et refuse le mode production. Ajouter à `package.json` :

```json
"upgrade:chapter8": "cross-env NODE_OPTIONS=--no-deprecation tsx scripts/upgrade-chapter8.ts"
```

Arrêter le serveur et lancer à la racine du projet :

```powershell
pnpm upgrade:chapter8
```

Le script crée d'abord une sauvegarde SQLite avec `VACUUM INTO` dans `.backups`,
à côté du dossier de l'application. Il ajoute la table des décisions, ses index
et une relation nullable dans la table des verrous, sans copier ni remplacer les
anciennes lignes. Les modifications sont appliquées dans une transaction avec
`client.batch`. Une seconde exécution est sans effet sur les données existantes.

S'il rencontre la table temporaire précise laissée par une synchronisation
échouée, il ne la retire que si elle est vide. Si elle contient des données,
il s'arrête sans modification : il faut alors examiner la situation et la
sauvegarde, pas forcer la suppression.

Dans `src/payload.config.ts`, prévoir le mode sans synchronisation automatique :

```ts
push: process.env.PAYLOAD_TEST_SCHEMA_READY !== '1'
  && process.env.PAYLOAD_DISABLE_SCHEMA_PUSH !== '1',
```

Après la mise à jour, ajouter dans le `.env` local :

```dotenv
PAYLOAD_DISABLE_SCHEMA_PUSH=1
```

L'application utilise alors le schéma mis à jour sans relancer sa reconstruction.
Une évolution ultérieure du schéma devra avoir sa propre migration explicite.
Une base neuve peut encore être initialisée avec la synchronisation activée;
ne pas désactiver celle-ci avant d'avoir créé les tables nécessaires.

Le `.env`, les bases et les sauvegardes ne doivent pas être publiés sur GitHub.
Ce script est une transition ciblée pour le chapitre 8, pas un outil général
de migration ni une commande à exécuter sur une base de production.

## 6. Ajouter les endpoints de relecture

Dans [quoteReview.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/endpoints/quoteReview.ts),
définir deux endpoints et les ajouter aux endpoints de `QuotePreviews` :

```ts
import { quoteReviewEndpoints } from '../endpoints/quoteReview'

endpoints: [
  ...quoteReviewEndpoints,
  // Conserver ici l'endpoint PDF du chapitre 7.
],
```

`GET /api/quote-previews/:id/review` retourne la référence et une décision,
ou `decision: null`. Il exige une session et désactive la mise en cache.

`POST /api/quote-previews/:id/decision` accepte ce corps JSON :

```json
{
  "decision": "approved",
  "reviewed": true,
  "reason": "Montants et prestations relus."
}
```

Pour un refus, remplacer `approved` par `rejected` et fournir un motif.
Le handler utilise l'identifiant de la route, pas une relation fournie dans
le corps. Il appelle `payload.create` avec `req` et `overrideAccess: false` :
l'utilisateur authentifié arrive ainsi jusqu'au hook.

| Réponse | Signification |
| --- | --- |
| 201 | Décision enregistrée |
| 400 | Identifiant, JSON ou décision invalide |
| 401 | Pas de session |
| 404 | Version inexistante |
| 409 | Une décision existe déjà pour cette version |

Le handler détecte les doublons connus; la contrainte unique reste la protection
finale en concurrence. Un échec d'enregistrement n'est jamais présenté comme
une approbation. La décision n'envoie pas d'e-mail et ne réécrit pas le PDF.

## 7. Construire l'interface de relecture

Créer [QuoteReviewActions.tsx](https://github.com/Anakinyo/payload-archiviste/blob/main/src/components/QuoteReviewActions.tsx)
avec `'use client'`. Le composant utilise `useDocumentInfo`, `useConfig`,
`useEffect` et `useState`. Il charge la décision de l'aperçu courant, puis affiche :

- la référence de version et l'état en attente ;
- une case « J'ai relu le PDF de cette version » ;
- un commentaire et deux actions, approuver ou refuser ;
- une confirmation explicite avec possibilité d'annuler ;
- après enregistrement, la décision, l'auteur, la date et le commentaire.

La relecture est obligatoire avant de rendre les actions disponibles. Le refus
demande aussi un motif. Ces contrôles existent à nouveau sur le serveur;
désactiver un bouton n'est jamais une protection suffisante.

Un échec de chargement ne doit pas afficher les actions de validation. Un échec
de soumission propose de rafraîchir la décision, par exemple si une autre personne
vient de décider sur la même version. Le composant annule ses mises à jour d'état
si l'utilisateur quitte la page pendant son chargement.
Un sous-composant reçoit une `key` liée à l'identifiant de l'aperçu : React
réinitialise ses états quand on change de version. La relance du chargement
réinitialise les états dans le gestionnaire du bouton, pas synchroniquement dans
l'effet. Le contrôle ESLint de React reste actif.

Dans les champs de `QuotePreviews`, placer le lien PDF puis le champ UI de
relecture au début du formulaire, avant l'instantané JSON :

```ts
{
  name: 'reviewActions', type: 'ui',
  admin: {
    components: { Field: '/components/QuoteReviewActions#QuoteReviewActions' },
  },
},
```

Ajouter dans `QuotePreviewActions` un lien vers
`${config.routes.admin}/collections/quote-previews/${previewId}` après génération.
Ce lien « Relire cette version » évite de rechercher le document dans la liste.

Reprendre les styles `.quote-review` dans
[custom.scss](https://github.com/Anakinyo/payload-archiviste/blob/main/src/app/%28payload%29/custom.scss).
Les actions passent à la ligne sur mobile et les références longues peuvent
se couper sans élargir la page. Les boutons et icônes viennent de l'UI Payload.
Sur la page de relecture, les dates de création et de modification de Payload
passent aussi à la ligne sur petit écran. Le fil d'Ariane a sa propre zone de
défilement pour laisser le bouton du compte visible malgré une référence longue.
Le test compare la largeur du document
à celle de la fenêtre; en cas d'échec, il relève les éléments débordants. Corriger
leur disposition plutôt que masquer le problème avec un `overflow-x: hidden`
global qui pourrait cacher des actions.

```powershell
pnpm generate:importmap
```

## 8. Tester les règles et le parcours

Le [chapitre 4](./04-configurer-entreprise-global.md#installer-et-configurer-les-tests)
explique l'installation des outils; le [chapitre 6](./06-calculer-montants.md)
explique les tests unitaires. Les tests ajoutés ici sont :

- **Unitaires** : relecture explicite, décision connue, motif, limite de taille,
  empreinte calculée sur les octets ;
- **Intégration Payload** : métadonnées forgées ignorées, ancien PDF inchangé,
  nouvelle version sans décision, accès interdit, immutabilité et concurrence ;
- **Mise à jour SQLite** : conservation des anciennes lignes, seconde exécution,
  unicité et protection d'une table temporaire contenant des données ;
- **Navigateur** : approbation sur mobile, annulation sans effet, confirmation,
  persistance après rechargement, refus motivé sur ordinateur et accès anonyme.

Créer `tests/unit/quoteDecision.unit.spec.ts` avec l'environnement Node.js :

```ts
// @vitest-environment node
import { describe, expect, it } from 'vitest'
import { validateQuoteDecision } from '../../src/domain/quoteDecision'

describe('Quote decisions', () => {
  it('requires a rejection reason', () => {
    expect(() => validateQuoteDecision({
      decision: 'rejected', reviewed: true, reason: '  ',
    })).toThrow('motif')
  })
})
```

Les tests d'intégration passent un utilisateur de fixture dans la Local API :

```ts
await payload.create({
  collection: 'quote-decisions',
  user: { ...userFixture, collection: 'users' },
  overrideAccess: false,
  data: { preview: preview.id, decision: 'approved', reviewed: true },
})
```

Ne pas supprimer ce `user` pour contourner les contrôles : même un appel privilégié
sans auteur est refusé par le hook. Dans le nettoyage, supprimer les décisions
avant les aperçus, puis les devis, clients et utilisateurs de test.

Pour le test de concurrence, lancer deux créations avec `Promise.allSettled`
sur le même aperçu, puis vérifier qu'une seule a réussi et qu'une seule décision
est présente dans la base. Reprendre les tests complets du dépôt :
[unitaires](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/unit/quoteDecision.unit.spec.ts),
[intégration](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/int/api.int.spec.ts),
[mise à jour SQLite](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/int/quoteUpgrade.int.spec.ts),
[navigateur](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/e2e/admin.e2e.spec.ts).

Arrêter le serveur de développement, puis lancer :

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm lint
pnpm build
```

`pnpm test` enchaîne les trois suites. La préparation du schéma E2E du chapitre 7
reste en place. Les captures de validation mobile et ordinateur sont générées
dans `test-results`, un dossier ignoré par Git.
La préparation E2E recrée la base de test jetable à chaque lancement pour éviter
qu'un ancien schéma SQLite perturbe les tests après l'ajout d'une collection.
Elle refuse une URL autre que `file:./payload-test.db` et ne modifie pas la base
de l'application. Lancer les suites successivement, pas en parallèle.

## 9. Essayer manuellement

```powershell
pnpm dev
```

1. Enregistrer un devis et générer un aperçu PDF.
2. Ouvrir son PDF puis suivre « Relire cette version ».
3. Confirmer la relecture, cliquer « Approuver en interne », puis annuler.
4. Vérifier que la version attend toujours une décision.
5. Recommencer et confirmer l'approbation; recharger la page.
6. Corriger le devis et générer un nouvel aperçu.
7. Vérifier que le nouvel aperçu attend une décision et que l'ancien conserve
   son approbation et son fichier.
8. Refuser la nouvelle version avec un motif; vérifier son affichage.

Le PDF du chapitre 7 conserve la mention « Aperçu non validé ». L'approbation est
visible dans l'application, mais ne transforme pas ce fichier en document final
à transmettre. Il s'agit de valider le contenu d'une version pour la suite.

## Prochaine étape

Préparer le document final et un envoi explicitement déclenché après validation
humaine, sans utiliser le devis courant à la place de la version relue. Le numéro
commercial, la présentation finale et la configuration de l'e-mail seront traités
avant tout envoi réel.
