# Chapitre 11 - Construire un espace de travail sur un ordinateur

## Objectif

L'entreprise commence avec un seul ordinateur portable. Pas d'hébergement
distant, pas de synchronisation et pas d'installation sur téléphone pour
l'instant. Le serveur Payload et SQLite fonctionnent sur cet ordinateur.

Nous remplaçons l'accueil du template par une interface métier : devis, clients,
missions et devis finaux. La saisie continue d'utiliser les formulaires Payload,
afin de conserver les validations et les actions déjà testées.

Cet épisode construit une interface web connectée au **serveur local**. Il ne
crée pas un cache autonome : si ce serveur est arrêté, la page ne fonctionne
plus. Internet n'est pas nécessaire pour les opérations purement locales;
un véritable envoi d'e-mail nécessite toujours une connexion.

## 1. Réutiliser le modèle plutôt que le dupliquer

Nous avons déjà les collections et leurs hooks. L'accueil ne calcule pas un
deuxième total et ne réécrit pas les règles de validation. Il lit les montants
enregistrés, affiche les données utiles et ouvre les formulaires existants.

Quatre vues sont proposées :

| Vue | Actions |
| --- | --- |
| Devis | Reprendre un devis ou en créer un |
| Clients | Rechercher, ouvrir une fiche ou créer un client |
| Missions | Consulter le statut, ouvrir ou créer une mission |
| Devis finaux | Ouvrir le document immuable ou son PDF |

Les devis de travail ne sont pas tous des brouillons sans document final :
ils peuvent être à l'origine de versions déjà approuvées. La vue ne leur
attribue donc pas un faux statut « envoyé » ou « signé ».

## 2. Charger les données côté serveur avec les permissions

Reprendre [loadWorkspace.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/workspace/loadWorkspace.ts)
dans `src/workspace`. Il refuse les appels sans utilisateur, charge la
configuration de l'entreprise, trois compteurs et la page demandée.

La Local API de Payload est privilégiée par défaut. Pour respecter les
permissions de l'utilisateur de l'interface, passer **les deux** options :

```ts
const clients = await payload.find({
  collection: 'clients',
  user,
  overrideAccess: false,
  limit: 20,
  page: 1,
})
```

Un simple `if (user)` dans le JSX ne protège pas une lecture effectuée avant
ce contrôle. La vérification doit précéder les requêtes métier. Ne pas mettre
les résultats d'un utilisateur dans un cache partagé sans séparation des droits.

Dans `src/app/(frontend)/page.tsx`, reprendre la
[page complète](https://github.com/Anakinyo/payload-archiviste/blob/main/src/app/%28frontend%29/page.tsx).
Son entrée utilise les cookies de la requête pour retrouver l'utilisateur :

```tsx
const payload = await getPayload({ config })
const { user } = await payload.auth({ headers: await headers() })
if (!user) {
  redirect(`/admin/login?redirect=${encodeURIComponent(workspaceURL(filters))}`)
}
const data = await loadWorkspace(payload, user, filters)
```

`headers()` et `searchParams` sont asynchrones dans la version Next.js du projet.
Lire la documentation incluse dans `node_modules/next/dist/docs` avant d'adapter
un exemple venant d'une autre version. La connexion est celle de Payload :
pas de deuxième mot de passe ou de deuxième système d'authentification.

## 3. Rechercher et paginer les clients

Reprendre [workspace.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/domain/workspace.ts)
dans `src/domain`. Il définit les vues autorisées et normalise les paramètres.

La recherche porte sur le nom affiché, le nom légal, le contact, l'e-mail,
le téléphone, le SIRET et les champs d'adresse. Elle est exécutée par Payload,
pas seulement sur les vingt lignes visibles dans le navigateur :

```ts
const where = {
  or: [
    { displayName: { contains: query } },
    { 'representative.name': { contains: query } },
    { 'address.street': { contains: query } },
  ],
}
```

Le helper complet ajoute les autres champs. Ne pas construire une chaîne SQL
avec la saisie de l'utilisateur. Cette recherche simple n'est pas encore un
moteur tolérant aux fautes ou aux accents; ses performances devront être
évaluées sur l'import réel du tableur.

La page affiche vingt résultats et conserve la recherche au changement de
page. Le tri utilise `updatedAt` puis l'identifiant pour départager les égalités.
Les pages invalides reviennent à 1; une page au-delà de la dernière affiche un
état vide et un lien de retour. La saisie est limitée à 120 caractères.

Les clients archivés restent recherchables et affichent « Archivé ». Les
distances sont des mesures aller simple, pas un calcul automatique de frais.
Une valeur absente est affichée comme absente, jamais remplacée par zéro.

La recherche figure dans l'URL : elle peut être conservée dans l'historique du
navigateur. Ne pas publier une URL contenant des informations de contact privées.

## 4. Mettre en place la navigation et la présentation

Mettre à jour le [layout](https://github.com/Anakinyo/payload-archiviste/blob/main/src/app/%28frontend%29/layout.tsx)
avec une langue `fr` et un titre « Espace de travail ». Reprendre le
[style dédié](https://github.com/Anakinyo/payload-archiviste/blob/main/src/app/%28frontend%29/styles.css).
Les tableaux et les actions remplacent l'écran de bienvenue du template.
Les icônes proviennent de la bibliothèque UI déjà installée de Payload :
aucune police ou image distante n'est nécessaire pour cet accueil.

Les liens vers l'accueil utilisent `next/link`. Les liens vers les formulaires
ouvrent le back-office dans la même fenêtre. Pour revenir après une saisie,
ajouter [WorkspaceNav.tsx](https://github.com/Anakinyo/payload-archiviste/blob/main/src/components/WorkspaceNav.tsx),
puis dans `admin` de `payload.config.ts` :

```ts
components: {
  beforeNavLinks: ['/components/WorkspaceNav#WorkspaceNav'],
},
```

Ajouter le style `.workspace-nav` à `custom.scss`, puis régénérer l'import map :

```powershell
pnpm generate:importmap
pnpm exec tsc --noEmit
```

Aucune nouvelle table n'est ajoutée par ce chapitre. Il ne faut pas effacer
la base ni activer sa synchronisation de schéma pour changer une page React.

## 5. Limiter le serveur à l'ordinateur

Mettre à jour ces scripts dans `package.json` :

```json
"dev": "cross-env NODE_OPTIONS=--no-deprecation next dev --hostname 127.0.0.1",
"start": "cross-env NODE_OPTIONS=--no-deprecation next start --hostname 127.0.0.1"
```

`127.0.0.1` est l'adresse de boucle locale. Le serveur n'écoute plus sur toutes
les interfaces réseau par défaut. Un téléphone ne pourra donc pas se connecter
à ce serveur tel qu'il est lancé ici : c'est volontaire pour ce premier usage.
Cette restriction ne remplace ni le mot de passe ni la protection du portable.

Ne pas ouvrir un port sur la box Internet. Garder le domaine et la messagerie
existants sans les modifier pour installer ce serveur local.

## 6. Préparer le premier essai sur le portable

L'installation de Node.js et pnpm est décrite au chapitre 1. Sur un nouveau
portable, choisir entre un projet neuf et une reprise des données existantes.
Ne pas écraser `.env` ni la base d'une installation déjà utilisée.

Pour un projet neuf : installer les dépendances, configurer `.env`, générer un
secret robuste, puis lancer `pnpm dev` une première fois. Créer le compte et
configurer l'entreprise comme dans les premiers chapitres. Arrêter ensuite le
serveur. SQLite doit exister avant le démarrage en mode production local.

Pour reprendre des données : suivre l'essai de récupération du chapitre 10 et
conserver le même secret, les médias et une version de code compatible avec la
base. Protéger les fichiers transférés; ne pas les placer dans le dépôt Git.

Dans les deux cas, conserver les envois en simulation pour la période d'essai :

```dotenv
QUOTE_EMAIL_MODE=capture
ALLOW_CLIENT_EMAIL_SEND=no
```

## 7. Démarrer la version de travail

Serveur de développement arrêté, vérifier la sauvegarde puis construire :

```powershell
pnpm backup:local
pnpm build
pnpm start
```

Ouvrir `http://127.0.0.1:3000` et se connecter. `pnpm start` sert la version
construite; il ne reconstruit pas le code. Après une mise à jour, arrêter le
serveur, sauvegarder, effectuer les migrations nécessaires, tester et refaire
`pnpm build` avant de relancer.

Le [lanceur Windows](https://github.com/Anakinyo/payload-archiviste/blob/main/Ouvrir-application.cmd)
peut être lancé par double-clic. Il vérifie la présence de Node.js, pnpm, `.env`,
de la base et du build, puis démarre la même commande. Garder sa fenêtre ouverte
pendant l'utilisation; `Ctrl+C` arrête le serveur. Ce n'est pas un installateur,
un service Windows ou un système de mise à jour automatique.

Si le port 3000 est déjà occupé, ne pas arrêter un programme inconnu : choisir
un autre port avec `pnpm start --port 3002`, puis ouvrir cette autre adresse.
La veille ou l'arrêt du portable rend l'application indisponible.
Les sauvegardes restent manuelles et une copie sur un support distinct reste
nécessaire. Le lanceur ne crée pas de sauvegarde en arrière-plan.

## 8. Tester le parcours

```powershell
pnpm test
pnpm exec tsc --noEmit
pnpm lint
pnpm build
```

Les nouveaux tests unitaires couvrent les paramètres, la recherche, les URLs
et le refus d'un appel sans utilisateur. Les tests navigateur vérifient
l'absence de données pour un visiteur anonyme, les montants enregistrés, le
retour depuis l'admin, vingt résultats par page, la recherche de contacts et
d'adresses, les clients archivés et les états vides.

Playwright utilise maintenant un seul worker : ses deux suites partagent la
même base SQLite jetable et ne doivent pas faire des écritures concurrentes
depuis deux processus. Les captures `workspace-desktop.png` et
`workspace-mobile.png` sont produites dans `test-results`, ignoré par Git.
La mise en page mobile est vérifiée, sans annoncer un accès téléphone actif.

Sources : [unitaires](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/unit/workspace.unit.spec.ts),
[navigateur](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/e2e/frontend.e2e.spec.ts).

## Prochaine étape

Le [chapitre 12](./12-importer-clients-csv.md) ajoute l'import des clients du
tableur avec prévisualisation et gestion des doublons. Faire essayer ensuite
le parcours réel sur le portable et affiner les formulaires et les documents.
Téléphone, PWA, synchronisation et hébergement distant
restent des évolutions possibles après validation de cet usage local.
