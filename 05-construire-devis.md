# Apprendre Payload CMS - Episode 5 : construire un devis et ses lignes

## Objectif

Un devis rassemble un client, une mission facultative et des prestations.
Cet épisode crée le formulaire dans l'administration Payload et vérifie la
cohérence des relations. Il prépare aussi un annuaire importable depuis un
tableur et l'utilisation de l'administration sur un petit écran.

Les devis sont encore des documents de travail. La numérotation, les calculs,
les instantanés des coordonnées, le PDF et le circuit de validation seront
ajoutés dans les prochains épisodes. Aucune action de cette étape n'envoie
de document au client.

## 1. Modéliser les prestations

Créer `src/collections/Quotes.ts`. La collection utilise le slug `quotes`,
les libellés français « Devis » et les mêmes permissions authentifiées que
les clients. Ajouter les champs suivants :

| Champ | Type Payload | Rôle |
| --- | --- | --- |
| `title` | `text`, requis | Intitulé lisible dans les listes |
| `client` | `relationship`, requis | Relation vers `clients` |
| `mission` | `relationship` | Relation facultative vers `missions` |
| `issuedAt` | `date`, requis | Date du document, préremplie aujourd'hui |
| `description` | `textarea` | Objet de la prestation |
| `lines` | `array`, au moins une ligne | Prestations proposées |
| `internalNotes` | `textarea` | Notes de gestion |

Une ligne contient une description, une quantité positive, une unité
(`day`, `hour`, `flat` ou `km`) et un prix unitaire HT en centimes.

```ts
{
  name: 'lines',
  type: 'array',
  required: true,
  minRows: 1,
  fields: [
    { name: 'description', type: 'textarea', required: true },
    { name: 'quantity', type: 'number', required: true, min: 0.01, defaultValue: 1 },
    {
      name: 'unit', type: 'select', required: true, defaultValue: 'day',
      options: [
        { label: 'Jour', value: 'day' },
        { label: 'Heure', value: 'hour' },
        { label: 'Forfait', value: 'flat' },
        { label: 'Kilomètre', value: 'km' },
      ],
    },
    {
      name: 'unitPriceCents', type: 'number', required: true, min: 0,
      validate: (value) => value == null || Number.isSafeInteger(value) ||
        'Saisir un nombre entier de centimes.',
    },
  ],
}
```

Un prix de 350 euros est stocké sous la forme `35000`. Cela prépare des
calculs monétaires maîtrisés. Les quantités peuvent être fractionnaires,
par exemple 1,5 jour. L'épisode suivant définira explicitement l'arrondi
des montants et ajoutera des tests unitaires. Le champ en centimes est une
interface provisoire; une saisie en euros pourra convertir la valeur ensuite.

## 2. Filtrer la mission et valider côté serveur

Le sélecteur de mission utilise `filterOptions` :

```ts
filterOptions: ({ data }) => data?.client
  ? { client: { equals: typeof data.client === 'object' ? data.client.id : data.client } }
  : false
```

Ce filtre aide la saisie. La règle doit aussi être vérifiée sur le serveur.
Ajouter un hook `beforeValidate` qui lit la mission avec `req.payload.findByID`
et compare son client à celui du devis. En cas de différence, lever une erreur.

La lecture interne utilise `overrideAccess: true` pour permettre aussi les
opérations Local API des tests. Les permissions de la collection Devis restent
appliquées aux opérations entrantes. Cette stratégie convient ici, où tous
les utilisateurs authentifiés accèdent aux mêmes clients et missions; des
permissions par entreprise exigeraient de revoir cette lecture.

Pour une mise à jour partielle, prendre le champ dans `data` lorsqu'il est
fourni, sinon dans `originalDoc`. Attention à `mission: null` : cela signifie
que la relation est volontairement retirée, pas qu'elle doit être conservée.

```ts
const client = data?.client ?? originalDoc?.client
const mission = data?.mission !== undefined ? data.mission : originalDoc?.mission
```

Passer `req` à la lecture Payload conserve le contexte de la requête et de sa
transaction. Le code complet est dans
[Quotes.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/collections/Quotes.ts).

## 3. Enregistrer la collection

Dans `src/payload.config.ts` :

```ts
import { Quotes } from './collections/Quotes'

// Dans buildConfig :
collections: [Users, Media, Clients, Missions, Quotes],
```

Puis régénérer les types :

```powershell
pnpm generate:types
```

Le type `Quote` permet maintenant d'utiliser `payload.create` et
`payload.update` avec les champs du devis vérifiés par TypeScript.

## 4. Préparer un annuaire importable

Un tableur peut contenir une entreprise ou une collectivité par ligne.
Ajouter un champ `importKey`, texte facultatif et unique, à `Clients`.
Il représente une identité stable qualifiée par source, par exemple
`communes:54395`. Le numéro de ligne ne convient pas : un tri le modifie.

Un futur import utilisera cette clé pour mettre à jour un client existant
plutôt que créer un doublon. Sa correspondance exacte avec les colonnes du
tableur sera définie au moment de l'import. Aucun accès Google n'est nécessaire
à ce stade; un export CSV pourra aussi servir de source.

Ajouter un groupe `travel` avec :

- `originLabel` : point de départ de référence ;
- `oneWayDistanceKm` : distance aller simple, nombre positif ou nul ;
- `oneWayDurationMinutes` : durée aller simple, nombre positif ou nul ;
- `measuredAt` : date de la mesure.

Laisser les valeurs inconnues vides. Une distance zéro a un sens différent
d'une distance inconnue. Ces champs représentent un trajet de référence;
une mission sur un autre site devra conserver ses propres mesures. Le futur
calcul de carburant nécessitera aussi la consommation du véhicule, le prix
du carburant et le nombre de déplacements. Ne pas importer une adresse
personnelle dans le code ou les exemples publiés.

## 5. Rechercher les clients

Dans `Clients.admin`, configurer :

```ts
listSearchableFields: [
  'displayName', 'legalName', 'representative.name', 'contactEmail',
  'address.street', 'address.postalCode', 'address.city',
],
```

La recherche de la liste peut ainsi porter sur le nom de la structure, son
signataire ou son adresse. Le champ `representative.name` reste pour l'instant
le contact principal; plusieurs contacts par client pourront être ajoutés
quand le besoin sera précisé. La recherche native ne constitue pas encore
une recherche approximative ou insensible aux accents.

Les données structurées et les API Payload servent aussi une future interface
mobile. L'administration fournit déjà le formulaire; une interface métier
plus courte pourra réutiliser les mêmes collections et validations.

## 6. Vérifier le comportement

Les tests d'intégration couvrent : création du devis, quantité fractionnaire,
prix entier en centimes, refus d'une liste vide, permissions, incohérence
client/mission lors de la création et d'une mise à jour partielle, retrait
explicite d'une mission, recherche par signataire et unicité de la clé d'import.

Un test Playwright ouvre le formulaire sur un écran de 390 × 844 pixels,
saisit un intitulé et vérifie l'absence de débordement horizontal. Ce contrôle
est une première base; il ne valide pas encore tous les parcours tactiles.

Arrêter le serveur de développement avant Playwright, puis exécuter :

```powershell
pnpm lint
pnpm test
pnpm build
pnpm dev
```

Dans l'administration, créer un devis avec un client, sélectionner une de ses
missions et ajouter une prestation de 1,5 jour à 35000 centimes. Enregistrer,
puis rouvrir le document pour vérifier les données. Tester aussi la recherche
d'un client par signataire ou ville.

## Prochaine étape

Calculer les montants avec des règles d'arrondi explicites, ajouter les tests
unitaires correspondants et appliquer les valeurs par défaut de l'entreprise.
Les coordonnées seront ensuite figées au moment de produire une version du
devis, afin qu'une modification ultérieure du client ne réécrive pas le document.
