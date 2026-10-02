# Apprendre Payload CMS - Episode 1 : installer un projet

## Objectif du fil rouge

Dans cette série, nous allons construire progressivement une application métier
pour une petite entreprise. Elle permettra notamment de gérer des clients, de
préparer des documents commerciaux à partir de modèles, de produire des PDF et
de suivre leur validation.

Ce premier épisode installe le socle technique. A la fin, nous disposerons d'une
application Payload fonctionnelle, de son interface d'administration et d'une
base de données locale.

## Ce que nous allons apprendre

- créer un projet Payload CMS ;
- comprendre le rôle de Payload, Next.js et TypeScript ;
- utiliser SQLite pour le développement local ;
- découvrir les principaux fichiers du projet ;
- démarrer et vérifier l'application.

## Pourquoi ces choix ?

- **Payload 3** réunit le CMS, son interface d'administration et ses API dans
  une même application TypeScript.
- **Next.js** fournit les pages et les routes serveur utilisées par Payload.
- **TypeScript** permet à Payload de générer les types correspondant au modèle
  de données et aide à détecter les erreurs pendant le développement.
- **SQLite** conserve les données dans un simple fichier. C'est pratique pour
  apprendre et développer sans installer un serveur de base de données.
- Le template **`blank`** fournit le minimum utile, sans contenu de démonstration
  à supprimer.
- **pnpm** installe et gère les dépendances JavaScript du projet.

Pour une véritable mise en production, nous envisagerons PostgreSQL. Commencer
avec SQLite ne nous empêchera pas de faire évoluer l'application plus tard.

## Prérequis

Installer une version LTS récente de Node.js depuis <https://nodejs.org>, puis
ouvrir PowerShell et vérifier l'installation :

```powershell
node --version
npm --version
```

Les deux commandes doivent afficher un numéro de version. Si elles ne sont pas
reconnues, fermer puis rouvrir PowerShell après l'installation de Node.js.

## Bonus Windows : installer Node.js en ligne de commande

Si `node`, `npm` ou `npx` ne sont pas reconnus, Node.js n'est probablement pas
encore installé ou son chemin n'a pas encore été chargé par le terminal.

Windows 10 et Windows 11 permettent d'installer la version LTS de Node.js avec
`winget`. Ouvrir PowerShell, puis exécuter :

```powershell
winget install --id OpenJS.NodeJS.LTS --exact --source winget
```

Le paquet installe **Node.js et npm ensemble** : il n'est pas nécessaire
d'installer npm séparément.

Une fois l'installation terminée, fermer tous les terminaux. Si le terminal est
intégré à Visual Studio Code, fermer complètement Visual Studio Code puis le
rouvrir afin qu'il recharge la variable d'environnement `PATH`.

Vérifier ensuite l'installation dans un nouveau terminal :

```powershell
node --version
npm --version
npx --version
```

Pour utiliser le même gestionnaire de paquets que dans ce tutoriel, installer
ensuite pnpm :

```powershell
npm install --global pnpm@11
pnpm --version
```

Il est maintenant possible de reprendre la création du projet avec
`npx create-payload-app@latest`.

### Si `winget` n'est pas reconnu

`winget` est distribué par Microsoft avec le programme **App Installer**. Mettre
App Installer à jour depuis le Microsoft Store, ou installer Node.js depuis
<https://nodejs.org> si `winget` n'est pas disponible sur la machine.

## Créer le projet

Se placer dans le dossier qui doit contenir l'application, puis lancer :

```powershell
npx create-payload-app@latest application-entreprise
```

Le nom `application-entreprise` est un exemple. Il peut être remplacé par le nom
du projet, de préférence en minuscules et sans espace.

Répondre ainsi aux questions du générateur :

1. choisir le template `blank` ;
2. choisir la base de données `SQLite` ;
3. accepter la chaîne de connexion proposée ;
4. choisir `pnpm` pour installer les dépendances.

Le générateur crée le dossier du projet et prépare automatiquement Payload,
Next.js, TypeScript et la base locale.

## Démarrer l'application

Entrer dans le nouveau dossier et lancer le serveur de développement :

```powershell
cd application-entreprise
pnpm dev
```

Ouvrir ensuite :

- <http://localhost:3000> pour la partie publique de l'application ;
- <http://localhost:3000/admin> pour l'administration Payload.

Au premier accès à l'administration, Payload demande de créer le premier compte
administrateur. Il n'existe volontairement aucun identifiant par défaut.

Le serveur reste actif tant que le terminal est ouvert. Pour l'arrêter, utiliser
`Ctrl+C` dans ce terminal.

## Les fichiers à connaître

```text
application-entreprise/
|-- src/
|   |-- app/                 pages Next.js et routes HTTP
|   |-- collections/         modèles de données Payload
|   |   |-- Media.ts
|   |   `-- Users.ts
|   |-- payload.config.ts    configuration centrale de Payload
|   `-- payload-types.ts     types générés automatiquement
|-- .env                     secrets et connexion à la base locale
|-- application-entreprise.db
|                             base de données SQLite locale
|-- .gitignore               fichiers qui ne doivent pas être publiés
`-- package.json             dépendances et commandes du projet
```

Une **collection** Payload représente un type de contenu administrable, assez
proche d'une table métier. La collection `Users`, par exemple, active
`auth: true`. Payload fournit alors l'authentification et l'utilise pour connecter
les utilisateurs au back-office.

Le fichier `payload.config.ts` assemble l'application : collections, éditeur de
texte, base de données, secret d'authentification, plugins et génération des
types TypeScript.

Le fichier `.env` contient des secrets et la base SQLite contient les données
locales. Ils ne doivent pas être publiés dans un dépôt Git.

## Versionner le projet avec Git

Git et GitHub ne désignent pas la même chose :

- **Git** enregistre localement l'historique des modifications ;
- **GitHub**, **GitLab** ou **Bitbucket** peuvent héberger une copie distante du
  dépôt et faciliter le partage du code.

Le générateur Payload initialise normalement Git automatiquement. La présence
du dossier caché `.git` indique que le projet est déjà un dépôt local. Le fichier
`.gitignore` indique à Git les fichiers locaux ou sensibles qu'il doit ignorer.

Vérifier l'état du dépôt :

```powershell
git status
```

Si `git` n'est pas reconnu sous Windows, installer Git for Windows :

```powershell
winget install --id Git.Git --exact
```

Fermer ensuite tous les terminaux et en ouvrir un nouveau, puis vérifier :

```powershell
git --version
```

Pour publier le projet, créer d'abord un dépôt vide sur l'hébergeur choisi. Ne
pas demander à l'hébergeur de créer un README ou un `.gitignore`, puisque ces
fichiers existent déjà localement. Copier ensuite l'URL du dépôt et exécuter :

```powershell
git remote add origin https://hebergeur.example/compte/application-entreprise.git
git add .
git commit -m "Initialise le projet Payload"
git push -u origin main
```

`origin` est le nom conventionnel donné au dépôt distant. L'option `-u` associe
la branche locale `main` à la branche distante ; les envois suivants pourront
donc se faire avec un simple `git push`.

Avant chaque commit, exécuter `git status` pour vérifier précisément les fichiers
qui seront enregistrés. Ne jamais ajouter manuellement `.env` ou la base SQLite.

## Commandes quotidiennes

Toutes ces commandes doivent être lancées depuis le dossier du projet :

```powershell
# Démarrer le serveur de développement
pnpm dev

# Vérifier la qualité du code
pnpm lint

# Régénérer les types après une modification des collections
pnpm generate:types

# Lancer les tests d'intégration
pnpm test:int

# Vérifier que l'application peut être compilée pour la production
pnpm build
```

## Vérifier l'installation

Le socle est opérationnel lorsque :

- la page <http://localhost:3000> s'affiche ;
- l'administration <http://localhost:3000/admin> s'affiche ;
- `pnpm generate:types` termine sans erreur ;
- `pnpm test:int` réussit ;
- `pnpm build` compile l'application.

Quelques avertissements peuvent être normaux à ce stade. Par exemple, aucun
service d'envoi d'e-mails n'est encore configuré. Les e-mails de développement
sont alors affichés dans le terminal au lieu d'être envoyés.

## Dépannage courant

### Une commande n'est pas reconnue

Après l'installation de Node.js, fermer tous les terminaux puis en ouvrir un
nouveau. Windows ne met pas à jour les variables d'environnement des terminaux
déjà ouverts.

Vérifier ensuite :

```powershell
node --version
npm --version
npx --version
```

### Le port 3000 est déjà utilisé

Next.js propose généralement un autre port, par exemple 3001. Utiliser alors
l'adresse indiquée dans le terminal.

### La base locale ne doit pas être versionnée

Vérifier que le fichier `.gitignore` contient une règle adaptée au nom de la
base SQLite, par exemple :

```gitignore
/*.db*
```

## Comment présenter Payload simplement

Payload est un CMS **code-first** : le modèle métier est défini en TypeScript,
puis Payload en déduit l'interface d'administration, les API et les types. Le
schéma fait donc partie du code : il peut être relu, testé et versionné avec le
reste de l'application.

SQLite réduit le nombre de composants nécessaires pendant l'apprentissage. Un
passage ultérieur à PostgreSQL changera principalement l'adaptateur de base de
données, sans imposer de réécrire les collections métier.

## Prochaine étape

Dans l'épisode suivant, nous créerons une collection `Clients` et l'afficherons
dans l'administration. Nous découvrirons les champs Payload, les validations,
les libellés en français, les permissions et la génération des types.
