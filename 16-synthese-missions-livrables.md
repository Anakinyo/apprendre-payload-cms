# Chapitre 16 - Synthétiser le suivi des missions et des livrables

## Objectif

L'entreprise dispose maintenant de missions, d'un journal de journées et de PDF.
Elle souhaite repérer rapidement les missions actives, celles en attente de
validation et les livrables qui ne sont pas encore prêts.

Nous construisons une page de consultation, sans nouveau champ, collection,
migration ni dépendance. Ce chapitre introduit une séparation simple entre les
règles de filtrage, les lectures Payload et l'affichage Next.js.

## 1. Définir exactement les indicateurs

| Vue | Données retenues | Unité du compteur |
| --- | --- | --- |
| Missions actives | Statut `planned` ou `in_progress` | Missions |
| Missions à valider | Statut `pending_validation` | Missions |
| Livrables en brouillon | Nature `deliverable`, état `draft` | Pièces |
| Livrables prêts | Nature `deliverable`, état `ready` | Pièces |

Une mission en brouillon, terminée ou annulée n'est pas une mission active.
Elle reste accessible depuis « Toutes les missions ». Les documents de travail
ne sont jamais comptés comme livrables.

Les livrables sont comptés indépendamment du statut de leur mission. Cela évite
de masquer un PDF resté en brouillon après la clôture de la mission. Si plusieurs
versions existent, chacune compte comme une pièce distincte : nous ne dédupliquons
pas les documents par leur titre.

Ces indicateurs ne mesurent ni le chiffre d'affaires ni l'avancement réel du travail.
Ils ne détectent pas un livrable attendu mais jamais déposé. Cette limite est
importante pour expliquer ce que la page permet réellement de suivre.

## 2. Préparer les filtres

Le fichier `src/domain/missionOverview.ts` contient :

- la liste autorisée des quatre vues ;
- l'analyse de `view` et `page` dans l'URL ;
- la construction des conditions Payload ;
- la génération des liens de pagination.

Les vues inconnues ou répétées reviennent à `active`. Le numéro de page doit
être un entier positif de cinq chiffres maximum ; sinon, nous utilisons 1.
Une page valide mais au-delà des résultats affiche un état vide, avec un lien
pour revenir à la première page.

Extrait de la règle pour les brouillons :

```ts
import type { Where } from 'payload'

const draftDeliverables: Where = {
  and: [
    { kind: { equals: 'deliverable' } },
    { state: { equals: 'draft' } },
  ],
}
```

Le filtre est construit à partir de valeurs définies dans notre code. Nous
n'acceptons pas un objet de requête arbitraire fourni dans l'URL.
`URLSearchParams` produit les liens plutôt que de concaténer manuellement
les paramètres. Changer de vue repart à la première page.

## 3. Lire les compteurs et les listes avec Payload

`src/workspace/loadMissionOverview.ts` refuse immédiatement un appel sans
utilisateur. Chaque lecture Local API précise ensuite :

```ts
const access = { user, overrideAccess: false }
```

Cette précaution est nécessaire : la Local API contourne les permissions par
défaut. Une page privée ne dispense pas de protéger les lectures elles-mêmes.

Pour chaque indicateur, `payload.count` utilise le filtre correspondant. Pour
la liste sélectionnée, `payload.find` réutilise **la même fonction de filtre** :

```ts
const options = {
  ...access,
  where: overviewWhere(filters.view),
  limit: 20,
  page: filters.page,
  sort: ['-updatedAt', '-id'],
  depth: 1,
}
```

Le compteur n'est pas la longueur de `docs` : 21 missions correspondent à un
compteur de 21, même si la première page n'en affiche que 20.
Le second critère de tri rend l'ordre déterministe lorsque les dates sont identiques.
`depth: 1` permet d'afficher le client d'une mission ou la mission d'un livrable.
Pour les pièces, `select` limite les champs récupérés au besoin de la liste.

Le résultat possède un discriminant `kind`, valant `missions` ou `documents`.
TypeScript sait ainsi quel type de ligne afficher, sans multiplier les conversions
de types dans le composant.

La synthèse ne crée ni ne modifie aucune donnée. Les lectures successives ne
constituent pas un instantané transactionnel : si une autre personne modifie
les données pendant le chargement, un rechargement peut être nécessaire.

## 4. Construire la page privée

Le fichier de page est :
`src/app/(frontend)/missions/synthese/page.tsx`.

Dans cette version de Next.js, `searchParams` est une promesse. Nous attendons
ses valeurs, puis nous vérifions la session avec `payload.auth` et les en-têtes
de la requête. Sans session, la page redirige vers la connexion avec une adresse
de retour construite à partir des filtres validés, avant toute lecture métier.

L'écran comprend quatre compteurs cliquables, des onglets, une liste paginée
et des actions pour ouvrir une mission, modifier une fiche ou télécharger un PDF.
Les téléchargements utilisent les routes protégées du chapitre 15.

La liste Missions reçoit un lien « Synthèse des missions ». La fiche de mission
reçoit un lien « Synthèse », pour revenir facilement après consultation.
La route statique `/missions/synthese` coexiste avec `/missions/[id]`.

Les tableaux restent dans une zone à défilement horizontal sur les petits écrans.
Les compteurs passent de quatre à deux colonnes. Les boutons avec icône possèdent
un nom accessible et une infobulle ; l'onglet sélectionné expose `aria-current`.

## 5. Tester les règles et les parcours

Depuis le dossier de l'application :

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm lint
pnpm build
```

Ou lancer toutes les familles de tests avec `pnpm test`.
Les tests d'intégration et navigateur restent limités à `payload-test.db`.
Les PDF de test sont fictifs, créés pour les tests et supprimés avec les fixtures.
Aucun e-mail réel n'est envoyé.

Les tests unitaires vérifient les paramètres invalides, les conditions exactes,
la pagination, le refus anonyme et la présence des permissions dans toutes les
requêtes. Les tests d'intégration utilisent plus de 20 missions pour vérifier
que le compteur porte sur toutes les pages.

Ils vérifient également qu'un livrable en brouillon reste visible sur une mission
terminée et qu'une correction vers `ready` déplace le document entre les listes,
sans modifier la mission. Les tests navigateur parcourent les onglets, les pages,
les téléchargements et les liens de retour ; ils vérifient les états vides et
l'absence de débordement de la page sur un écran étroit.

## 6. Installer et refaire la démonstration

Partir d'une application fonctionnelle au chapitre 15. Arrêter son serveur,
mettre à jour le code puis exécuter les vérifications précédentes. Aucune
commande `upgrade:chapter16` n'est nécessaire, car le schéma ne change pas.

```powershell
pnpm build
pnpm start
```

Le code complet reste dans le
[dépôt de l'application](https://github.com/Anakinyo/payload-archiviste).
La page est accessible à `/missions/synthese` après connexion.

Pour une démonstration reproductible :

1. Créer une mission fictive au statut « Planifiée », puis ouvrir la synthèse.
2. Passer cette mission « En attente de validation » dans son suivi et recharger
   la synthèse : elle doit changer de liste.
3. Déposer un PDF comme livrable en brouillon et vérifier le compteur correspondant.
4. Marquer le livrable « Prêt » dans sa fiche, puis recharger la synthèse : il doit
   quitter les brouillons et apparaître dans les prêts.
5. Vérifier que le statut de la mission n'a pas changé et qu'aucun envoi n'a eu lieu.
6. Ouvrir la synthèse sans session : la connexion doit être demandée.

Une installation sans mission affiche des compteurs à zéro et une liste vide.
Nous n'ajoutons pas de données de démonstration à la base utilisée par l'entreprise.

## Limites et suite

La page se met à jour lors d'un nouveau chargement. Il n'y a ni actualisation
automatique, notification, date d'échéance, détection de retard, liste des
livrables attendus manquants, ni synchronisation entre appareils.
Tous les utilisateurs connectés de l'entreprise disposent encore du même accès.

Le parcours consacré aux devis et au suivi des missions dispose maintenant d'une
vue de consultation transversale. La prochaine partie préparera la facturation
et les paiements, en cadrant d'abord les données et les règles nécessaires.
