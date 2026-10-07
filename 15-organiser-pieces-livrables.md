# Chapitre 15 - Organiser les pièces et les livrables des missions

## Objectif

L'entreprise souhaite retrouver les PDF associés à une mission : documents de
travail et livrables. Elle doit pouvoir les télécharger sans les rendre publics.
Un livrable devient « Prêt » uniquement après une action humaine. Aucun fichier
n'est envoyé au client dans ce chapitre.

Nous réutilisons les mécanismes natifs de Payload : une collection avec
`upload`, une relation vers les missions, des permissions et des hooks.

## 1. Préparer le projet

Partir du chapitre 14, avec Node.js, pnpm et une base locale fonctionnelle.
Arrêter le serveur avant les changements de schéma. Le code complet est dans
le dépôt de l'application :
[payload-archiviste](https://github.com/Anakinyo/payload-archiviste).

Les principaux fichiers de cette étape sont :

- `src/collections/MissionDocuments.ts` : configuration de la collection ;
- `src/domain/missionDocumentFile.ts` : validation du PDF ;
- `src/components/MissionDocumentForm.tsx` : formulaire de dépôt ;
- `src/app/(frontend)/missions/[id]/page.tsx` : liste des pièces ;
- `src/database/upgradeChapter15SQLite.ts` : évolution additive de SQLite.

## 2. Modéliser une pièce

Chaque pièce possède un titre, une mission, une nature, un état et des notes
internes facultatives. Payload ajoute les informations du fichier : nom, taille,
type MIME, URL et dates de création et de modification.

| Champ | Type Payload | Règle |
| --- | --- | --- |
| `title` | `text` | Obligatoire, 160 caractères maximum |
| `mission` | `relationship` | Obligatoire, vers `missions`, indexé |
| `kind` | `select` | `work` ou `deliverable` |
| `state` | `select` | `draft` ou `ready` |
| `internalNotes` | `textarea` | 2 000 caractères maximum |

Le dépôt depuis l'espace de travail crée toujours un brouillon. Dans le back-office,
une personne peut ensuite marquer un livrable « Prêt ». Un document de travail
ne peut pas prendre cet état. La mission n'est jamais terminée automatiquement.

## 3. Utiliser une collection de fichiers

Le principe de la configuration est :

```ts
const authenticated = ({ req }) => Boolean(req.user)

// Extrait de configuration, à intégrer dans une CollectionConfig.
const options = {
  slug: 'mission-documents',
  access: {
    create: authenticated,
    read: authenticated,
    update: authenticated,
    delete: authenticated,
  },
  upload: {
    staticDir: path.resolve('media/mission-documents'),
    mimeTypes: ['application/pdf'],
    pasteURL: false,
    hideRemoveFile: true,
  },
}
```

Importer `path` depuis `node:path`, puis enregistrer la collection dans
`src/payload.config.ts`. L'extrait ne remplace pas le fichier complet : celui-ci
contient également les champs, les hooks et les en-têtes de téléchargement.

Le dossier n'est pas placé dans `public`. Payload sert les fichiers par son API
et vérifie la permission `read`, y compris pour une URL de fichier connue.
Les médias existants deviennent eux aussi accessibles uniquement après connexion.
Ce point évite de laisser les photos de diagnostic publiques et protège les
dossiers imbriqués contre un autre chemin de téléchargement.

Dans cette première version, tous les utilisateurs connectés de l'entreprise
partagent le même accès. Nous ne créons pas encore de cloisonnement par mission.

## 4. Contrôler le contenu côté serveur

`accept` sur un champ HTML et `mimeTypes` dans Payload facilitent le choix du
fichier, mais ne constituent pas une validation suffisante.

Le hook `beforeOperation` vérifie le fichier avant son traitement :

1. la taille réelle du contenu est comprise entre 1 octet et 10 Mio ;
2. le nom finit par `.pdf` et le type déclaré est `application/pdf` ;
3. les premiers octets correspondent à la signature PDF ;
4. `PDFDocument.load`, fourni par `pdf-lib`, peut lire le document ;
5. le document contient au moins une page.

Un fichier renommé, illisible, vide ou protégé par mot de passe est refusé.
Ce contrôle n'est pas un antivirus et ne garantit pas qu'un PDF soit sans contenu
malveillant. N'ouvrir que les fichiers provenant de sources fiables.

Lors d'une modification, remplacer le fichier est interdit. Pour une nouvelle
version, créer une nouvelle pièce avec un titre distinct. Les métadonnées métier
restent modifiables ; les informations du fichier ne peuvent pas être réécrites.
Ce comportement n'est pas un historique de versions immuable : la suppression
reste possible pour les utilisateurs connectés.

Le hook `beforeValidate` vérifie également l'existence de la mission et la
cohérence entre nature et état. Un hook de mission empêche sa suppression tant
que des pièces lui sont rattachées.

## 5. Ajouter le formulaire et la liste

Dans l'espace de travail, ouvrir une mission puis la section « Pièces et livrables ».
Le formulaire envoie un `FormData` contenant :

```ts
body.set('file', file)
body.set('_payload', JSON.stringify({
  mission: id,
  title,
  kind,
  state: 'draft',
}))
```

L'API native reçoit `POST /api/mission-documents`. Le navigateur transmet le
cookie de session à la même origine. Ne pas fixer soi-même `Content-Type` : le
navigateur doit ajouter la frontière du formulaire multipart.

Après succès, le formulaire est réinitialisé et la liste est actualisée. En cas
d'erreur, les champs sont conservés. La liste est paginée par 25 pièces et filtrée
sur la mission côté serveur. Les appels Local API utilisent `user` et
`overrideAccess: false`, car cette API contourne les permissions par défaut.

Les téléchargements utilisent `Cache-Control: private, no-store`,
`X-Content-Type-Options: nosniff` et `Content-Disposition: attachment`.

## 6. Mettre à jour la base locale

Dans le dossier de l'application, serveur arrêté :

```powershell
pnpm generate:types
pnpm backup:local
pnpm upgrade:chapter15
pnpm build
pnpm start
```

Le script de mise à jour est réservé à `file:./payload-archiviste.db`, hors mode
production. Il crée une copie SQLite avant modification, ajoute la table des
pièces et la relation de verrouillage, puis conserve les tables existantes.
Il peut être relancé sans recréer les pièces. Une installation neuve peut créer
le schéma avec le mécanisme de développement habituel de Payload.

Les PDF sont dans `media/mission-documents`, déjà inclus par la sauvegarde
récursive du chapitre 10. **Une copie de SQLite seule ne suffit donc pas.**
Pour une restauration complète, conserver la base et tout le dossier `media`,
avec les fichiers de configuration nécessaires conservés séparément et en sécurité.
Une sauvegarde n'est pas chiffrée : son accès doit être protégé.

## 7. Tester et refaire la démonstration

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm lint
pnpm build
```

Les tests d'intégration et navigateur utilisent uniquement la base jetable
`payload-test.db`. Ne pas les pointer vers la base utilisée par l'entreprise.
La suite complète se lance aussi avec `pnpm test`.

Les nouveaux tests couvrent les faux PDF, les tailles invalides, les permissions,
la création en brouillon, l'interdiction du remplacement, la protection des
missions, l'évolution SQLite et le téléchargement des mêmes octets après dépôt.
Les tests navigateur vérifient aussi le refus anonyme et l'affichage étroit.

Pour une démonstration manuelle : créer une mission fictive, déposer un PDF de
test, le télécharger, puis modifier la pièce dans le back-office et choisir
« Livrable » et « Prêt ». Vérifier que la mission n'a pas changé de statut.
Copier l'URL du fichier dans une fenêtre sans session : l'accès doit être refusé.

## Limites et suite

Seuls les PDF sont acceptés pour l'instant. Les fichiers bureautiques, photos
comme pièces, génération de bordereaux, antivirus, validation réglementaire,
envoi des livrables et archivage probant restent à concevoir séparément.
Le statut « Prêt » est un repère interne, pas une autorisation d'élimination.

L'application reste locale, sur un ordinateur. La prochaine étape proposée est
une vue de synthèse des missions et des livrables à préparer, avant d'aborder
les factures et les paiements.
