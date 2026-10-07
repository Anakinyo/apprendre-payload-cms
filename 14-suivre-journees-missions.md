# Chapitre 14 - Suivre les journées réalisées et le temps de mission

## Objectif

L'entreprise peut préparer une mission et produire son devis. Elle souhaite
maintenant noter les interventions réalisées et comparer le temps passé à
l'estimation initiale. Nous ajoutons un journal privé, puis une fiche de suivi
dans l'espace de travail.

Une fiche du journal représente **une mission à une date**, avec total du
travail et total des trajets. Plusieurs interventions sur cette même mission
le même jour sont regroupées dans cette fiche. Une autre mission peut avoir
sa propre fiche à la même date.

Ce journal n'est pas une facture, un relevé de dépenses ou un système de paie.
Il ne modifie aucun devis enregistré et ne déclenche aucun e-mail. Les journées
restent corrigeables et supprimables : aucun journal d'audit immuable n'est
ajouté dans ce chapitre.

## 1. Créer une collection plutôt qu'un tableau dans Mission

Ajouter `src/collections/MissionDays.ts` et l'enregistrer dans `collections`
de `src/payload.config.ts`, à côté de `Missions`.

| Champ | Type Payload | Rôle |
| --- | --- | --- |
| `mission` | relationship | Mission concernée, obligatoire et indexée |
| `day` | text | Date civile au format `AAAA-MM-JJ` |
| `workMinutes` | number | Temps de travail entier, obligatoire |
| `travelMinutes` | number | Temps de trajet entier, zéro par défaut |
| `internalNotes` | textarea | Notes privées, 2 000 caractères maximum |
| `entryKey` | text unique | Clé technique calculée par le serveur |

Une collection séparée permet de paginer le journal, corriger une journée
sans réenregistrer toute la mission et appliquer des permissions propres.
Le journal conserve une relation vers la mission; il ne duplique pas le client.

La date est volontairement du texte : il s'agit d'une journée civile, pas d'un
instant horodaté. Une date `2026-01-01` ne doit pas devenir la veille lors d'une
conversion de fuseau horaire. Cette décision exige une validation serveur du
calendrier, car une chaîne ne suffit pas à garantir une date réelle.

## 2. Valider au niveau du domaine

`src/domain/missionTracking.ts` regroupe validation et calculs.

Règles de chaque fiche :

- une date réelle, entre le 1er janvier 2000 et aujourd'hui ;
- au moins une minute de travail ;
- des minutes entières, sans durée négative ;
- au maximum 1 440 minutes, travail et trajets additionnés ;
- une mission existante et accessible à l'utilisateur.

Pour « aujourd'hui », `parisDay` utilise explicitement `Europe/Paris` avec
`Intl.DateTimeFormat`, indépendamment du fuseau du processus Node.js. La
validation compare ensuite les dates au format ISO et vérifie leur
normalisation : `2025-02-29` et `2026-02-31` sont refusées.

La limite de 24 heures s'applique à une fiche. Cette version ne vérifie pas
les chevauchements entre missions ni un total global par intervenant et par
jour : elle ne gère pas encore les heures de début/fin ou plusieurs salariés.

## 3. Calculer une clé stable dans un hook

Dans `beforeValidate`, résoudre les valeurs du formulaire et celles de
`originalDoc` lorsqu'une modification ne fournit qu'une partie des champs.
Valider les données, vérifier la mission avec la Local API et calculer :

```ts
entryKey: `${missionID}:${day}`
```

Le client ne choisit pas cette clé. Même s'il envoie un `entryKey` arbitraire,
le hook le remplace. Une contrainte unique empêche deux fiches pour la même
mission et la même date, y compris si deux requêtes arrivent simultanément.
Un contrôle JavaScript dans le formulaire ne suffirait pas.

À la création comme à la modification, une requête partielle ne doit pas
contourner les règles. Corriger seulement `workMinutes` garde la date et la
mission actuelles; changer la date ou la mission recalcule la clé et reste
soumis à la contrainte unique.

Les quatre permissions de la collection exigent un utilisateur connecté.
La lecture de la mission depuis le hook passe `req` et `overrideAccess: false`.
Les vues de suivi font également respecter les permissions Payload.

## 4. Protéger la suppression d'une mission

Ajouter à `Missions` un hook `beforeDelete` qui cherche l'existence d'une
journée liée. Si une fiche existe, la suppression de la mission est refusée.
Cela évite de détacher un journal de son contexte.

Ce contrôle d'intégrité interne utilise `overrideAccess: true` pour compter
l'existence d'une relation, même lors d'un appel système à la Local API. Il
ne renvoie aucun contenu du journal. Ce n'est pas le fonctionnement des vues
utilisateur : elles utilisent toujours les permissions et l'utilisateur courant.

Les droits de suppression de la mission elle-même restent inchangés. Pour
supprimer volontairement une mission possédant des fiches, vérifier puis
supprimer ses journées avant la mission. Une sauvegarde reste recommandée
avant des suppressions réelles.

## 5. Comparer temps réalisé et temps prévu

Le récapitulatif additionne les minutes du journal. Les trajets sont affichés
séparément et ne consomment pas automatiquement le budget de travail estimé.

```text
Travail prévu = durée estimée en jours × heures de référence × 60
Équivalent journées = minutes travaillées / (heures de référence × 60)
```

L'estimation est arrondie à la minute la plus proche; les minutes réalisées
restent entières. Par défaut, la journée de référence vaut 7 heures. Une
estimation absente est distinguée d'une estimation explicitement égale à zéro.

Exemple : pour une journée prévue de 7 heures, 8 h 30 de travail et 30 minutes
de trajet donnent :

- travail réalisé : 8 h 30 ;
- trajets : 0 h 30 ;
- équivalent journées : environ 1,21 ;
- dépassement du travail prévu : 1 h 30.

Le pourcentage affiché est plafonné à 100 %, avec dépassement séparé. Il mesure
**la consommation de temps**, pas l'achèvement du classement ou du récolement.
Une mission peut avoir consommé son estimation sans être terminée.

Changer l'estimation dans la mission recalcule la comparaison. Les minutes
réalisées ne changent pas. Ce suivi courant n'est pas un budget figé ni un
calcul automatique de facturation.

## 6. Charger toutes les données sans confondre pagination et total

`src/workspace/loadMissionTracking.ts` exige un utilisateur avant de lire les
données, puis charge la mission et ses fiches via `overrideAccess: false`.

Deux lectures répondent à deux besoins différents :

1. toutes les durées de cette mission, par lots de 1 000, pour calculer les totaux ;
2. 25 fiches, triées par date décroissante et identifiant, pour afficher la page.

Calculer le récapitulatif à partir de la seule page visible donnerait un total
incorrect après 25 fiches. Un test d'intégration couvre précisément ce cas.
La vue locale refuse les journaux de plus de 20 000 fiches au lieu de tronquer
silencieusement les totaux; une exploitation massive demanderait une
agrégation adaptée côté base.

Les notes sont lues pour le tableau courant, mais seules les durées sont
sélectionnées pour l'addition. Aucune donnée n'est placée dans un cache public.

## 7. Construire la fiche de suivi Next.js

La page `src/app/(frontend)/missions/[id]/page.tsx` utilise les paramètres
asynchrones `params` et `searchParams`. Elle authentifie l'utilisateur avec
les en-têtes de la requête avant de charger une mission.

Un visiteur anonyme est envoyé vers la connexion. Un identifiant invalide
ou une mission introuvable produit une page 404. Depuis la liste Missions,
l'intitulé ouvre désormais cette fiche; l'icône de modification continue
d'ouvrir le formulaire Payload.

La fiche présente les totaux, la comparaison avec l'estimation, un statut,
le formulaire de journée et le journal paginé. Le formulaire interactif de
`src/components/MissionTrackingForms.tsx` convertit heures/minutes en minutes
avant d'appeler la REST API Payload existante.

```text
POST /api/mission-days
PATCH /api/missions/:id
```

La création utilise le premier endpoint; le second n'est appelé que lorsque
l'utilisateur enregistre explicitement un changement de statut. Aucun seuil
de temps ne passe automatiquement la mission à « Terminée ».

Comme dans les chapitres précédents, les commandes restent désactivées
jusqu'à l'hydratation React. Après une réussite, `router.refresh()` recharge
les données serveur sans effacer les parties indépendantes de l'interface.
Il n'y a pas de cache serveur partagé à invalider dans cette vue authentifiée.

Les erreurs de création ne sont jamais présentées comme des réussites. Pour
un doublon, ouvrir la fiche existante via son icône de correction. Le formulaire
Payload permet de modifier la journée ou de la supprimer avec sa confirmation
habituelle. Après un retour depuis l'administration, actualiser le suivi pour
recalculer les totaux si nécessaire.

## 8. Mettre à jour la base locale

Sur une base neuve, Payload crée la collection. Sur la base existante du
chapitre 13, serveur arrêté :

```powershell
pnpm backup:local
pnpm upgrade:chapter14
pnpm generate:types
```

Le script d'upgrade vérifie la base cible et crée une copie SQLite avant la
modification. Son helper ajoute `mission_days`, les index, la contrainte de
clé unique et une relation dans `payload_locked_documents_rels` pour les
verrous de l'administration. Les missions existantes ne sont pas réécrites.
L'upgrade peut être exécuté plusieurs fois sans effacer les données.

Conserver `PAYLOAD_DISABLE_SCHEMA_PUSH=1` pour cette base mise à jour. Comme
les upgrades précédents, ce script vise l'installation locale du cours, pas
un processus de migration de production distante.

## 9. Tester et essayer

```powershell
pnpm test
pnpm exec tsc --noEmit
pnpm lint
pnpm build
pnpm start
```

Les tests ajoutés vérifient dates et fuseau, valeurs entières, totaux et
dépassements, estimation manquante, permissions, doublons, correction,
suppression et pagination. Ils vérifient également qu'une mission avec des
journées ne peut pas être supprimée et que les notes du journal ne sont pas
reprises dans le snapshot d'un devis.

Playwright ajoute une journée depuis la fiche de suivi, vérifie le refus d'un
doublon, ajoute une autre date, contrôle le dépassement puis change le statut
explicitement. Les captures grand/petit écran sont produites dans
`test-results`. La mise en page est responsive, sans activer un accès téléphone
ou une synchronisation : le serveur reste sur `127.0.0.1`.

Sources : [domaine](https://github.com/Anakinyo/payload-archiviste/blob/main/src/domain/missionTracking.ts),
[collection](https://github.com/Anakinyo/payload-archiviste/blob/main/src/collections/MissionDays.ts),
[unitaires](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/unit/missionTracking.unit.spec.ts),
[intégration](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/int/missionTracking.int.spec.ts),
[navigateur](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/e2e/missionTracking.e2e.spec.ts).

Ouvrir `http://127.0.0.1:3000`, puis Missions et une fiche. Saisir une date,
le travail effectué, les trajets éventuels et les notes internes; enregistrer
puis comparer le récapitulatif au planning. Le statut reste une décision de
l'entreprise, indépendante du pourcentage de temps.

## Suite

Les pièces de mission et les livrables pourront compléter ce suivi. Les
kilomètres réellement parcourus, dépenses, calendriers détaillés, plusieurs
intervenants, historique d'audit et facturation à partir des journées restent
des évolutions distinctes, non implémentées dans ce chapitre.
