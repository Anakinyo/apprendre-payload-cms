# Apprendre Payload CMS - Episode 2 : créer une collection Clients

## Objectif

Dans l'épisode précédent, nous avons installé Payload et ouvert son interface
d'administration. Nous allons maintenant créer notre premier véritable modèle
métier : une collection `Clients`.

A la fin de cet épisode, un utilisateur connecté pourra créer et modifier des
fiches clients depuis l'administration. Payload produira également l'API et les
types TypeScript correspondants.

## Ce que nous allons apprendre

- déclarer une collection Payload en TypeScript ;
- choisir les champs adaptés aux données métier ;
- organiser un formulaire avec des lignes et des groupes ;
- afficher certains champs sous condition ;
- ajouter une validation et transformer une valeur ;
- protéger une collection avec les règles d'accès ;
- enregistrer la collection dans la configuration Payload ;
- générer les types et tester le comportement obtenu.

## Concevoir la fiche avant de coder

Pour éviter un formulaire difficile à utiliser, commençons par les informations
réellement nécessaires :

- un nom court pour identifier le client dans les listes ;
- une catégorie : entreprise, collectivité, administration, association ou
  particulier ;
- la raison sociale et le SIRET lorsqu'ils existent ;
- une adresse e-mail et un téléphone ;
- le représentant ou signataire ;
- l'adresse postale ;
- les références Chorus Pro pour les clients publics ;
- des notes internes qui ne seront jamais reprises dans les documents clients.

Seul le nom affiché et la catégorie seront obligatoires. Une fiche pourra ainsi
être créée même si toutes les informations ne sont pas encore connues.

## Créer le fichier de collection

Dans `src/collections`, créer un fichier `Clients.ts` :

```ts
import type {
  Access,
  CollectionConfig,
  TextFieldSingleValidation,
} from 'payload'

const authenticated: Access = ({ req }) => Boolean(req.user)
const validateSiret: TextFieldSingleValidation = (value) =>
  !value ||
  /^\d{14}$/.test(value) ||
  'Le SIRET doit contenir exactement 14 chiffres.'

export const Clients: CollectionConfig = {
  slug: 'clients',
  labels: {
    singular: 'Client',
    plural: 'Clients',
  },
  access: {
    create: authenticated,
    delete: authenticated,
    read: authenticated,
    update: authenticated,
  },
  admin: {
    useAsTitle: 'displayName',
    defaultColumns: ['displayName', 'type', 'siret', 'contactEmail', 'isActive'],
    group: 'Activité commerciale',
  },
  fields: [
    {
      type: 'row',
      fields: [
        {
          name: 'displayName',
          type: 'text',
          label: 'Nom affiché',
          required: true,
          index: true,
          admin: {
            width: '60%',
            description: 'Nom court utilisé dans les listes et les documents.',
          },
        },
        {
          name: 'type',
          type: 'select',
          label: 'Catégorie',
          required: true,
          defaultValue: 'entreprise',
          options: [
            { label: 'Entreprise', value: 'entreprise' },
            { label: 'Collectivité', value: 'collectivite' },
            { label: 'Administration', value: 'administration' },
            { label: 'Association', value: 'association' },
            { label: 'Particulier', value: 'particulier' },
          ],
          admin: {
            width: '40%',
          },
        },
      ],
    },
    {
      name: 'legalName',
      type: 'text',
      label: 'Raison sociale ou nom légal',
    },
    {
      type: 'row',
      fields: [
        {
          name: 'siret',
          type: 'text',
          label: 'SIRET',
          unique: true,
          hooks: {
            beforeValidate: [
              ({ value }) =>
                typeof value === 'string' ? value.replace(/\s/g, '') : value,
            ],
          },
          validate: validateSiret,
          admin: {
            width: '50%',
            description:
              'Facultatif pour les particuliers. Les espaces sont retirés automatiquement.',
          },
        },
        {
          name: 'isActive',
          type: 'checkbox',
          label: 'Client actif',
          defaultValue: true,
          admin: {
            width: '50%',
          },
        },
      ],
    },
    {
      name: 'contactEmail',
      type: 'email',
      label: 'Adresse e-mail',
    },
    {
      name: 'phone',
      type: 'text',
      label: 'Téléphone',
    },
    {
      name: 'representative',
      type: 'group',
      label: 'Représentant ou signataire',
      fields: [
        {
          name: 'title',
          type: 'text',
          label: 'Fonction ou civilité',
          admin: {
            description:
              'Par exemple : Maire, Présidente, Responsable administratif.',
          },
        },
        {
          name: 'name',
          type: 'text',
          label: 'Nom complet',
        },
      ],
    },
    {
      name: 'address',
      type: 'group',
      label: 'Adresse postale',
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
      name: 'chorus',
      type: 'group',
      label: 'Facturation publique - Chorus Pro',
      admin: {
        condition: (data) =>
          ['collectivite', 'administration'].includes(data?.type),
        description:
          'Ces informations sont utilisées pour la facturation des clients publics.',
      },
      fields: [
        {
          name: 'serviceCode',
          type: 'text',
          label: 'Code service',
        },
        {
          name: 'commitmentNumber',
          type: 'text',
          label: "Numéro d'engagement juridique",
        },
      ],
    },
    {
      name: 'internalNotes',
      type: 'textarea',
      label: 'Notes internes',
      admin: {
        description:
          'Informations réservées à la gestion interne et absentes des documents clients.',
      },
    },
  ],
}
```

## Comprendre la configuration

### `slug`

Le `slug` est l'identifiant technique stable de la collection. Avec
`slug: 'clients'`, Payload crée notamment l'URL REST `/api/clients`.

Les libellés français sont séparés du slug : on peut améliorer les textes de
l'interface sans casser les API ni les relations existantes.

### `access`

Les quatre opérations utilisent la fonction `authenticated`. Elle renvoie
`true` uniquement lorsque `req.user` existe, donc lorsqu'un utilisateur est
connecté.

Cette première règle distingue seulement les visiteurs et les utilisateurs
authentifiés. Nous pourrons introduire des rôles plus tard si plusieurs personnes
utilisent l'application avec des responsabilités différentes.

### `admin`

`useAsTitle` indique à Payload quel champ représente une fiche dans les listes
et les relations. `defaultColumns` choisit les colonnes visibles par défaut.
`group` range la collection sous un intitulé métier dans la navigation.

Ces options changent l'expérience du back-office, pas la structure des données.

### `row` et `group`

Un champ `row` place plusieurs champs sur une même ligne du formulaire. Il ne
crée pas de niveau supplémentaire dans les données.

Un champ `group`, au contraire, rassemble les valeurs dans un objet. L'adresse
obtiendra par exemple cette forme :

```json
{
  "address": {
    "street": "1 rue de la Paix",
    "postalCode": "54000",
    "city": "Nancy",
    "country": "France"
  }
}
```

### Validation et hook du SIRET

Le hook `beforeValidate` retire les espaces avant la validation et
l'enregistrement. Une personne peut donc saisir `123 456 789 00012`, tandis que
la base conservera `12345678900012`.

La fonction `validateSiret`, typée avec `TextFieldSingleValidation`, accepte une
valeur vide mais exige exactement 14 chiffres si un SIRET est fourni.
`unique: true` empêche deux clients de partager le même SIRET. Le type explicite
permet aussi à TypeScript de contrôler la fonction de validation pendant la
compilation.

Cette validation contrôle le format, pas l'existence administrative du SIRET.
Une vérification auprès d'un registre externe serait une fonctionnalité séparée.

### Champ conditionnel Chorus Pro

La fonction `admin.condition` affiche le groupe Chorus Pro uniquement lorsque
la catégorie vaut `collectivite` ou `administration`.

Il s'agit d'une condition d'affichage. Si une règle métier devait interdire une
valeur dans tous les contextes, y compris via l'API, il faudrait également la
faire respecter côté serveur avec une validation ou un hook.

## Enregistrer la collection dans Payload

Une collection n'est pas chargée automatiquement parce que son fichier existe.
Dans `src/payload.config.ts`, importer `Clients` :

```ts
import { Clients } from './collections/Clients'
```

Puis l'ajouter au tableau `collections` :

```ts
collections: [Users, Media, Clients],
```

Au redémarrage, Payload compare cette configuration au schéma de la base et
prépare les structures nécessaires.

## Générer les types TypeScript

Après toute modification du modèle Payload, exécuter :

```powershell
pnpm generate:types
```

Payload met à jour `src/payload-types.ts`. Le projet dispose alors d'un type
`Client`, et TypeScript connaît les valeurs autorisées pour sa catégorie ainsi
que la structure de son adresse et de ses autres groupes.

Le fichier est généré automatiquement : il ne faut pas le modifier à la main.

## Vérifier dans l'administration

Démarrer le projet :

```powershell
pnpm dev
```

Ouvrir <http://localhost:3000/admin>, se connecter, puis sélectionner
**Activité commerciale > Clients**.

Créer une fiche d'essai avec, par exemple :

- nom affiché : `Entreprise de démonstration` ;
- catégorie : `Entreprise` ;
- SIRET : `123 456 789 00012` ;
- ville : `Nancy`.

Après l'enregistrement, le SIRET doit apparaître sans espaces. En remplaçant la
catégorie par `Collectivité`, le groupe Chorus Pro doit apparaître.

## Tester automatiquement la collection

Une vérification manuelle confirme que le formulaire est agréable à utiliser.
Un test automatisé confirme que les règles continuent de fonctionner après une
future modification.

Dans un test d'intégration, créer un client avec un SIRET espacé et vérifier la
valeur enregistrée :

```ts
const client = await payload.create({
  collection: 'clients',
  data: {
    displayName: 'Entreprise de test',
    type: 'entreprise',
    siret: '123 456 789 00012',
  },
})

expect(client.siret).toBe('12345678900012')
```

Vérifier également qu'une valeur invalide est refusée :

```ts
await expect(
  payload.create({
    collection: 'clients',
    data: {
      displayName: 'Client invalide',
      type: 'entreprise',
      siret: '123',
    },
  }),
).rejects.toThrow()
```

Lancer ensuite les contrôles :

```powershell
pnpm lint
pnpm test:int
pnpm build
```

## Ce que Payload a produit pour nous

A partir d'un seul fichier de configuration, nous obtenons :

- un écran de liste et un formulaire dans l'administration ;
- une table dans la base de données ;
- une API REST sous `/api/clients` ;
- des requêtes GraphQL ;
- des validations côté serveur ;
- un type TypeScript `Client`.

C'est le principe central de Payload : le modèle TypeScript devient la source
de vérité commune au back-office, aux API et au code de l'application.

## Exercice

Ajouter un champ facultatif `website` de type `text`, avec le libellé `Site web`,
puis régénérer les types. Observer les changements dans le formulaire et dans
le type `Client` de `src/payload-types.ts`.

## Prochaine étape

Dans l'épisode suivant, nous relierons une première collection métier aux
clients. Nous pourrons alors apprendre les relations Payload et commencer à
structurer le workflow qui mènera à la génération d'un devis.
