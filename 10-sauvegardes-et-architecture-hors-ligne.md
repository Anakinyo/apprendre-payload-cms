# Chapitre 10 - Sauvegarder et préparer le travail hors connexion

## Objectif et état réel

L'entreprise souhaite utiliser l'application sur ordinateur et téléphone,
continuer certains travaux sans Internet et synchroniser ensuite les données.
Cela influence le déploiement : nous préparons l'architecture avant de choisir
un hébergeur ou de migrer la base.

**Implémenté dans ce chapitre :** sauvegarde locale de SQLite et des médias,
manifeste d'empreintes, vérification indépendante et tests.
**Prévu, pas encore implémenté :** interface installable, stockage hors ligne,
synchronisation et gestion des conflits. L'admin actuel nécessite toujours
un serveur joignable. Aucun compte cloud n'est nécessaire pour cet épisode.

Sur l'ordinateur qui exécute déjà Payload et SQLite localement, les opérations
locales ne nécessitent pas Internet. Le besoin nouveau concerne surtout un
téléphone autonome et plusieurs appareils dont les modifications se rejoignent.

Le programme initial plaçait PostgreSQL et le déploiement ici. Ces sujets sont
reportés après le cadrage hors ligne; migrer SQLite n'activerait pas une
synchronisation entre appareils.

## 1. Distinguer les trois parties de l'application

```text
Interface de travail        Serveur métier           Stockage central
PC ou téléphone     →       Payload            →     SQLite ou PostgreSQL
```

L'interface pourra fonctionner localement. Payload exécute les permissions,
les calculs, les hooks, la génération finale et les envois. La base conserve
les données : elle ne remplace pas automatiquement ces traitements.
La [documentation de déploiement Payload](https://payloadcms.com/docs/production/deployment)
distingue justement le runtime de l'application et ses stockages.

Brancher une interface directement sur une base distante contournerait les
règles métier existantes. Un service de type BaaS pourrait porter d'autres
fonctions et permissions, mais cela demanderait une nouvelle architecture et
de nouveaux tests. Il ne faut pas embarquer un mot de passe SQL ou SMTP dans
l'application du téléphone.

## 2. Pourquoi sauvegarder avant de synchroniser

Une synchronisation peut propager une erreur ou une suppression sur tous les
appareils. Une sauvegarde conserve un état indépendant permettant de revenir
en arrière. Ces deux mécanismes ne sont pas interchangeables.

Git protège le code, pas les données : `.env`, SQLite et les médias sont ignorés.
Pour récupérer l'application, il faut le code, les données, les médias et une
configuration conservée séparément dans un emplacement protégé.

Le cache hors ligne futur ne sera pas davantage une sauvegarde : un navigateur
peut perdre ses données, et un appareil perdu emporte les brouillons encore
non synchronisés. Un export de ces brouillons sera nécessaire.

## 3. Ajouter les commandes de sauvegarde

Reprendre ces fichiers dans le projet du chapitre 9 :

- [localBackup.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/database/localBackup.ts) dans `src/database`;
- [local-backup.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/scripts/local-backup.ts) dans `scripts`;
- [localBackup.int.spec.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/int/localBackup.int.spec.ts) dans `tests/int`.

Ils utilisent les dépendances déjà installées : `@libsql/client`, `tsx` et
les bibliothèques standard de Node.js. Aucune nouvelle collection ni migration
de schéma n'est nécessaire.

Ajouter à `scripts` dans `package.json`, en conservant les autres commandes :

```json
"backup:local": "cross-env NODE_OPTIONS=--no-deprecation tsx scripts/local-backup.ts create",
"backup:verify": "cross-env NODE_OPTIONS=--no-deprecation tsx scripts/local-backup.ts verify"
```

La commande de création exige l'URL locale du fil rouge :

```dotenv
DATABASE_URL=file:./payload-archiviste.db
```

Elle ne traite pas une base PostgreSQL ou une autre URL. Ne pas changer cette
URL pour contourner une erreur : utiliser une procédure adaptée au stockage.

## 4. Créer une sauvegarde

Arrêter le serveur avec `Ctrl+C` et arrêter les tests, puis exécuter depuis
le dossier du projet :

```powershell
pnpm backup:local
```

Un dossier à nom unique est créé dans `../.backups` :

```text
.backups/
  local-DATE-UUID/
    database.db
    media/
    manifest.json
```

Le résultat affiche son chemin, le nombre de fichiers protégés et leur taille
totale. L'absence de médias produit simplement un dossier `media` vide.
Le script ne remplace pas la base active et ne copie pas `.env`.

Pourquoi arrêter le serveur ? La base et les fichiers sont sauvegardés
successivement. SQLite sait produire un instantané cohérent de sa base, mais
cela ne donne pas une transaction commune avec les fichiers médias. Arrêter
les écritures évite qu'un média change entre les deux opérations.

La base est créée avec une instruction SQLite paramétrée :

```sql
VACUUM INTO ?;
```

Cette méthode inclut les écritures SQLite validées, y compris avec un journal
WAL; copier uniquement un `.db` actif pourrait les omettre. Voir
[VACUUM INTO](https://www.sqlite.org/lang_vacuum.html). Les tests exercent aussi
ce cas avec un journal WAL actif.

## 5. Vérifier séparément

Utiliser le chemin affiché, entre guillemets si nécessaire :

```powershell
pnpm backup:verify "C:\atelier\.backups\local-DATE-UUID"
```

Le chemin ci-dessus est un exemple générique. La vérification :

1. valide le format du manifeste;
2. refuse les chemins sortant du dossier et les liens symboliques;
3. compare le nombre des médias et les tailles;
4. compare les empreintes SHA-256 de la base et des médias;
5. lance `PRAGMA quick_check` et `PRAGMA foreign_key_check` sur la copie.

Un fichier altéré, manquant ou ajouté empêche une vérification réussie.
Un manifeste absent indique une sauvegarde incomplète. Une erreur pendant la
création peut laisser un dossier partiel : ne pas le considérer comme valide.
Ne pas modifier les empreintes pour faire disparaître une alerte.

Les empreintes détectent les différences par rapport au manifeste; elles ne
chiffrent pas les données et ne constituent pas une signature authentifiée.
Une personne capable de modifier ensemble les fichiers et le manifeste peut
remplacer ce dernier. Protéger donc l'accès à tout le dossier.

## 6. Conserver et essayer la récupération

Une sauvegarde sur le même disque ne protège pas contre sa panne. Conserver
une copie du **dossier complet** sur un support distinct, avec des droits
restreints et une protection adaptée aux données de l'entreprise. Ne pas
publier les sauvegardes sur GitHub. Le script n'assure ni chiffrement,
ni sauvegarde externe automatique, ni politique de rétention.

Avant un incident, essayer la récupération dans une copie séparée du projet :

1. installer les dépendances correspondant au code sauvegardé;
2. vérifier le dossier de sauvegarde;
3. dans le projet de récupération, serveur arrêté, placer `database.db` sous
   le nom `payload-archiviste.db` et reprendre le dossier `media`;
4. reconstituer la configuration protégée, notamment le même `PAYLOAD_SECRET`;
5. conserver `PAYLOAD_DISABLE_SCHEMA_PUSH=1` et le mode e-mail `capture`;
6. démarrer cette copie sur un autre port et vérifier les données et documents.

```powershell
pnpm dev --port 3002
```

Ne pas faire cet essai en écrasant la base originale. La commande proposée
ne restaure rien automatiquement : une récupération réelle doit être contrôlée.
La vérification SQLite n'est pas une validation de toutes les règles métier.

Les mises à jour du schéma et les versions de code doivent rester compatibles
avec la sauvegarde. Conserver la référence du commit utilisé, séparément du
manifeste de données. Ne pas lancer une ancienne application sur une base
migrée sans examiner cette compatibilité.

## 7. Concevoir le futur mode hors ligne

Une PWA est une application web installable; ce n'est pas une installation
de Node.js et de Payload sur Android. Nous prévoyons une interface dédiée
pour les tâches courantes, distincte du back-office technique.

Les [service workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
permettent de conserver les ressources de l'interface. Les données métier
iront dans [IndexedDB](https://web.dev/learn/pwa/offline-data), avec une
bibliothèque comme [Dexie](https://dexie.org/docs/Tutorial/Getting-started).
Il faut un premier chargement connecté et une origine HTTPS stable.
Ajouter un manifeste d'installation ne rend pas l'admin existant hors ligne.

Premier périmètre envisagé : consulter les clients et missions téléchargés,
modifier des notes et préparer des brouillons de devis. Les PDF disponibles
hors ligne doivent avoir été téléchargés explicitement.
L'interface distinguera les données locales des données synchronisées.

Les numéros définitifs, la validation interne et les e-mails resteront en ligne
dans cette première version. Un brouillon possède un UUID, pas un numéro de
devis réservé indépendamment par chacun des appareils.

## 8. Reprendre après une coupure, sans perdre des données

Une modification sera enregistrée dans une file locale durable. Le serveur
accusera réception après son écriture. Si le réseau tombe après l'écriture
mais avant la réponse, rejouer le même UUID ne devra pas créer un doublon.

Les données auront une révision serveur. Si deux appareils modifient la même
fiche à partir d'une ancienne version, il faut garder les deux contenus et
montrer le conflit. « La dernière sauvegarde gagne » peut supprimer le travail
d'un appareil. Les dates des téléphones ne sont pas un arbitre fiable.

La reprise visera l'ouverture de l'application, le retour d'un serveur joignable
et une action manuelle. Ne pas garantir la synchronisation application fermée :
les [capacités de synchronisation en arrière-plan](https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API)
varient. Une reconnexion n'enverra jamais automatiquement un devis au client.

Ne pas synchroniser les fichiers `.db` entre deux appareils via un dossier
partagé : ce n'est pas une fusion des modifications. Il faut une API, des
opérations identifiées et une gestion des conflits.
Le [cadrage détaillé](https://github.com/Anakinyo/payload-archiviste/blob/main/docs/offline-and-sync.md)
liste aussi suppressions, dépendances, comptes, révocation et curseurs.

## 9. Peut-on éviter les frais d'hébergement ?

Deux compromis principaux restent ouverts :

| Choix | Conséquence |
| --- | --- |
| Serveur sur l'ordinateur de l'entreprise | Pas d'abonnement distant; il faut que l'ordinateur soit allumé et accessible pour synchroniser. |
| Petit serveur distant et stockage persistant | Fonctionne sans cet ordinateur; un hébergement existe, même si son offre est gratuite. |

Une base cloud seule ne donne pas le second résultat avec l'application
Payload actuelle. Le fournisseur et la topologie seront choisis après avoir
décidé si la synchronisation doit fonctionner ordinateur éteint.

Observations vérifiées le 6 octobre 2026, à recontrôler avant utilisation :

- [Supabase Free](https://supabase.com/pricing) offre une piste PostgreSQL,
  mais avec quotas, pause après inactivité et sans sauvegardes automatiques incluses.
- [Render Free](https://render.com/docs/free) peut servir pour un prototype;
  son stockage local est éphémère et les ports SMTP habituels sont bloqués.
  Le fournisseur déconseille cette offre pour une application de production.
- [Vercel Hobby](https://vercel.com/docs/plans/hobby) est réservé à un usage
  personnel non commercial; il ne convient pas par défaut à ce fil rouge professionnel.

Ce ne sont pas des garanties de gratuité durable ou de disponibilité.
La future architecture doit permettre de changer de fournisseur. Les économies
de fonctionnement doivent aussi tenir compte du temps de maintenance et des
sauvegardes, pas uniquement du montant de l'abonnement.

## 10. Tester

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm exec tsc --noEmit
pnpm lint
pnpm build
```

Les tests de sauvegarde créent uniquement des fichiers temporaires. Ils vérifient
les lignes de la base copiée, les médias imbriqués, les écritures WAL, l'absence
de copie de `.env`, la détection d'une altération et le refus de chemins invalides.
La base active n'est jamais remplacée par ces tests.

La suite inclut également un test d'hydratation du bouton de génération PDF
du chapitre 7. Pour charger les tests de composants `.tsx`, conserver dans
`vitest.config.mts` le motif élargi présenté au chapitre 6 :

```ts
include: ['tests/int/**/*.int.spec.ts', 'tests/unit/**/*.unit.spec.{ts,tsx}']
```

Le parcours navigateur attend maintenant la réponse HTTP de génération et
enchaîne ses étapes dépendantes en mode sériel. En cas d'échec d'une étape,
les étapes suivantes ne tentent pas d'utiliser un identifiant inexistant.

Sous Windows, les handles natifs libSQL peuvent rester ouverts jusqu'à la fin
du processus. Le test principal utilise un sous-processus, comme la commande
réelle, puis vérifie et nettoie ses fichiers temporaires.

Les tests de synchronisation seront ajoutés au moment de son implémentation :
deux appareils, conflits, doubles requêtes, serveur indisponible, réponse perdue,
session expirée, changements de compte et fermeture/réouverture hors ligne.

## Prochaine étape

Construire l'interface PWA et ses premiers brouillons locaux, puis ajouter la
synchronisation et ses protections. PostgreSQL, les migrations de production
et le choix d'hébergement restent des étapes distinctes. Rien dans ce chapitre
ne rend encore l'application utilisable sans connexion au serveur.
