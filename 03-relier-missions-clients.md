# Apprendre Payload CMS - Episode 3 : relier des missions aux clients

## Objectif

Nous disposons maintenant d'une collection `Clients`. Une application métier ne
se limite toutefois pas à des fiches isolées : ses données sont liées entre
elles.

Dans cet épisode, nous allons créer une collection `Missions`. Chaque mission
sera rattachée à un client existant et décrira son type, son état d'avancement,
son diagnostic et sa planification.

## Ce que nous allons apprendre

- créer une relation entre deux collections ;
- comprendre la différence entre un identifiant et une relation peuplée ;
- utiliser la profondeur de lecture, ou `depth` ;
- représenter une liste avec un champ `array` ;
- relier un fichier avec un champ `upload` ;
- adapter le formulaire au type de mission ;
- modéliser un workflow avec un champ de statut ;
- tester une création qui traverse plusieurs collections.

## Concevoir la mission

Une première version utile de la mission contient :

- le client concerné ;
- un intitulé ;
- un type de mission ;
- un statut d'avancement ;
- un lieu ;
- les informations issues du diagnostic ;
- une estimation de durée ;
- les règles de démarrage, de trajet et de facturation ;
- quelques livrables propres au type de mission ;
- des notes internes.

Nous ne recopions pas l'adresse ou le SIRET du client dans la mission. La mission
référence la fiche client, qui reste la source de vérité pour ces informations.

## Créer la collection

Créer `src/collections/Missions.ts` et commencer par la configuration générale :

```ts
import type { Access, CollectionConfig } from 'payload'

const authenticated: Access = ({ req }) => Boolean(req.user)

const eliminationMissionTypes = ['elimination', 'classement_elimination']
const classificationMissionTypes = ['classement', 'classement_elimination']

export const Missions: CollectionConfig = {
  slug: 'missions',
  defaultSort: '-updatedAt',
  labels: {
    singular: 'Mission',
    plural: 'Missions',
  },
  access: {
    create: authenticated,
    delete: authenticated,
    read: authenticated,
    update: authenticated,
  },
  admin: {
    useAsTitle: 'title',
    defaultColumns: ['title', 'client', 'type', 'status', 'diagnosticDate'],
    group: 'Activité commerciale',
  },
  fields: [
    // Les champs seront ajoutés dans les sections suivantes.
  ],
}
```

`defaultSort: '-updatedAt'` affiche en premier les missions modifiées le plus
récemment. Le signe `-` indique un tri décroissant.

## Ajouter la relation vers le client

Le premier champ relie la mission à la collection `clients` :

```ts
{
  name: 'client',
  type: 'relationship',
  relationTo: 'clients',
  label: 'Client',
  required: true,
  index: true,
}
```

`relationTo` reçoit le slug technique de la collection cible. Le champ est
obligatoire : une mission ne peut pas exister sans client dans notre modèle.

`index: true` demande à la base d'optimiser les recherches par client. Cette
recherche deviendra fréquente lorsque nous afficherons toutes les missions d'un
client ou préparerons un devis.

## Ajouter l'identité et le workflow

Ajouter ensuite le titre, le type et le statut :

```ts
{
  name: 'title',
  type: 'text',
  label: 'Intitulé de la mission',
  required: true,
  index: true,
},
{
  type: 'row',
  fields: [
    {
      name: 'type',
      type: 'select',
      label: 'Type de mission',
      required: true,
      options: [
        { label: 'Élimination', value: 'elimination' },
        { label: 'Classement', value: 'classement' },
        {
          label: 'Classement et élimination',
          value: 'classement_elimination',
        },
        { label: 'Récolement', value: 'recolement' },
        {
          label: 'Maintenance des archives',
          value: 'maintenance_archives',
        },
      ],
      admin: { width: '50%' },
    },
    {
      name: 'status',
      type: 'select',
      label: 'Statut',
      required: true,
      defaultValue: 'draft',
      options: [
        { label: 'Brouillon', value: 'draft' },
        { label: 'Planifiée', value: 'planned' },
        { label: 'En cours', value: 'in_progress' },
        {
          label: 'En attente de validation',
          value: 'pending_validation',
        },
        { label: 'Terminée', value: 'completed' },
        { label: 'Annulée', value: 'cancelled' },
      ],
      admin: { width: '50%' },
    },
  ],
},
```

Les valeurs techniques restent courtes, stables et sans accents. Les libellés
peuvent évoluer sans modifier les données enregistrées.

Le statut ne fait pas encore respecter les transitions. A ce stade, il décrit
l'état courant. Nous ajouterons des règles lorsque le workflow aura des effets
sensibles, comme l'envoi d'un document.

## Structurer le diagnostic

Le diagnostic est un objet composé d'une date, d'un volume, d'une liste de lieux
et d'une image :

```ts
{
  name: 'diagnostic',
  type: 'group',
  label: 'Diagnostic',
  fields: [
    {
      name: 'date',
      type: 'date',
      label: 'Date du diagnostic',
      admin: {
        date: {
          displayFormat: 'dd/MM/yyyy',
          pickerAppearance: 'dayOnly',
        },
      },
    },
    {
      name: 'archiveVolumeLinearMeters',
      type: 'number',
      label: "Volume d'archives estimé (mètres linéaires)",
      min: 0,
    },
    {
      name: 'archiveLocations',
      type: 'array',
      label: 'Lieux de conservation des archives',
      labels: {
        singular: 'Lieu de conservation',
        plural: 'Lieux de conservation',
      },
      fields: [
        {
          name: 'label',
          type: 'text',
          label: 'Lieu',
          required: true,
        },
      ],
    },
    {
      name: 'coverImage',
      type: 'upload',
      relationTo: 'media',
      label: 'Photo de couverture',
    },
  ],
},
```

Un champ `array` représente une liste ordonnée d'éléments de même structure.
Payload ajoute automatiquement les boutons permettant d'ajouter, supprimer et
réordonner les lignes dans l'administration.

Le champ `upload` est une relation spécialisée vers une collection configurée
pour recevoir des fichiers. Ici, il pointe vers la collection `media` créée par
le template Payload.

## Ajouter la planification

```ts
{
  name: 'planning',
  type: 'group',
  label: 'Planification',
  fields: [
    {
      type: 'row',
      fields: [
        {
          name: 'durationDays',
          type: 'number',
          label: 'Durée estimée (jours)',
          min: 0,
          admin: { width: '50%' },
        },
        {
          name: 'hoursPerDay',
          type: 'number',
          label: 'Heures par jour',
          defaultValue: 7,
          min: 1,
          max: 24,
          admin: { width: '50%' },
        },
      ],
    },
    {
      name: 'startCondition',
      type: 'textarea',
      label: 'Conditions de démarrage',
    },
    {
      name: 'travelTimePolicy',
      type: 'textarea',
      label: 'Règle relative au temps de trajet',
    },
    {
      name: 'invoiceTrigger',
      type: 'textarea',
      label: 'Condition de facturation',
    },
  ],
},
```

Les bornes `min` et `max` sont vérifiées côté serveur. Elles protègent donc les
données créées depuis l'administration comme celles envoyées directement à
l'API.

## Adapter le formulaire au type de mission

Certains livrables ne concernent qu'un type de mission :

```ts
{
  name: 'eliminationFollowUpRequired',
  type: 'checkbox',
  label: "Suivi du bordereau d'élimination requis",
  defaultValue: true,
  admin: {
    condition: (data) => eliminationMissionTypes.includes(data?.type),
  },
},
{
  name: 'classificationPlanRequired',
  type: 'checkbox',
  label: 'Plan de classement requis',
  defaultValue: true,
  admin: {
    condition: (data) => classificationMissionTypes.includes(data?.type),
  },
},
{
  name: 'recolementReportRequired',
  type: 'checkbox',
  label: 'Rapport de récolement requis',
  defaultValue: true,
  admin: {
    condition: (data) => data?.type === 'recolement',
  },
},
```

`admin.condition` améliore le formulaire, mais ne constitue pas une règle de
sécurité. Une valeur cachée peut toujours exister dans une requête API. Une règle
qui doit absolument être respectée devra aussi être validée côté serveur.

Terminer avec le lieu et les notes internes :

```ts
{
  name: 'locationLabel',
  type: 'text',
  label: 'Lieu de la mission',
  admin: {
    description: 'Libellé court utilisé sur la couverture du document.',
  },
},
{
  name: 'internalNotes',
  type: 'textarea',
  label: 'Notes internes',
  admin: {
    description:
      'Ces notes ne seront pas reprises dans les documents remis au client.',
  },
},
```

## Enregistrer la collection

Dans `src/payload.config.ts`, importer la collection :

```ts
import { Missions } from './collections/Missions'
```

Puis l'ajouter à la configuration :

```ts
collections: [Users, Media, Clients, Missions],
```

Régénérer les types :

```powershell
pnpm generate:types
```

Le type `Mission` contient alors une propriété relationnelle :

```ts
client: number | Client
```

## Comprendre `depth`

Payload peut renvoyer une relation sous deux formes :

```json
{ "client": 12 }
```

ou sous forme d'objet peuplé :

```json
{
  "client": {
    "id": 12,
    "displayName": "Entreprise de démonstration",
    "type": "entreprise"
  }
}
```

La profondeur de lecture, appelée `depth`, détermine jusqu'où Payload remplace
les identifiants par les documents liés. `depth: 0` conserve les identifiants ;
une profondeur supérieure peuple les relations.

Une profondeur élevée simplifie parfois l'affichage, mais augmente la taille et
le coût des réponses. Il vaut mieux demander uniquement les données nécessaires.

## Tester la relation

Créer d'abord un client, conserver son identifiant, puis créer la mission :

```ts
const mission = await payload.create({
  collection: 'missions',
  draft: false,
  data: {
    client: testClientId,
    title: "Élimination réglementaire des archives",
    type: 'elimination',
    status: 'draft',
    diagnostic: {
      archiveVolumeLinearMeters: 18,
      archiveLocations: [
        { label: "Bureau d'accueil" },
        { label: 'Salle des archives' },
      ],
    },
    planning: {
      durationDays: 2,
    },
  },
})
```

La relation peut être un nombre ou un objet selon la profondeur obtenue :

```ts
const relatedClientId =
  typeof mission.client === 'number' ? mission.client : mission.client.id

expect(relatedClientId).toBe(testClientId)
expect(mission.status).toBe('draft')
expect(mission.planning?.hoursPerDay).toBe(7)
```

Lors du nettoyage du test, supprimer la mission avant le client. Une relation ne
doit pas continuer à pointer vers un document supprimé.

## Vérifier dans l'administration

Démarrer l'application :

```powershell
pnpm dev
```

Ouvrir <http://localhost:3000/admin/collections/missions> et créer une mission.

Vérifier que :

- le champ Client propose les clients existants ;
- le statut vaut `Brouillon` par défaut ;
- `Heures par jour` vaut 7 par défaut ;
- les lieux de conservation peuvent être ajoutés et réordonnés ;
- les champs de livrables changent avec le type de mission ;
- une photo peut être choisie depuis la médiathèque.

Puis exécuter les contrôles techniques :

```powershell
pnpm lint
pnpm test:int
pnpm build
```

## Relation vivante ou instantané ?

La mission utilise une relation vivante : si le nom du client change, la mission
retrouve le nouveau nom lors de la prochaine lecture.

Un devis validé pose un autre problème. Son contenu doit rester identique même
si la fiche client est modifiée plus tard. Nous conserverons donc la relation
pour la navigation, mais la génération du document devra aussi figer un
instantané des informations légales utilisées au moment de la validation.

Cette distinction entre donnée de référence et donnée historique est essentielle
dans les applications commerciales.

## Prochaine étape

Dans l'épisode suivant, nous créerons les paramètres de l'entreprise avec un
`Global` Payload. Nous découvrirons ainsi la différence entre une collection de
plusieurs documents et une configuration unique, avant de construire les devis.
