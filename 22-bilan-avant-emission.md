# 22. Vérifier la situation avant émission

## Objectif

Une relecture approuvée reste un événement historique. Elle ne garantit pas que
les données actuelles sont encore identiques : une adresse, les conditions de
règlement ou le brouillon peuvent avoir changé depuis la décision.

Ce chapitre ajoute un **diagnostic en lecture seule**, affiché sur la page de
chaque relecture conservée. Il ne crée pas de facture, ne réserve pas de numéro
et ne transmet aucun document. L'émission définitive reste le prochain chantier.

## 1. Séparer historique et situation actuelle

On conserve sans modification la relecture et sa décision. Au chargement de la
page, le serveur récupère aussi le brouillon, le client et les paramètres actuels.

Les lectures Payload utilisent l'utilisateur authentifié, `overrideAccess: false`
et `depth: 0`. Sans session, la page redirige vers la connexion. Les relations sont
résolues explicitement pour ne charger que les documents nécessaires.

Le serveur reconstruit ensuite la relecture actuelle avec `buildInvoiceReview`.
Un brouillon ou un client indisponible devient un point bloquant, pas une raison
d'effacer l'historique.

## 2. Isoler les règles dans une fonction testable

Le fichier `src/domain/invoiceReadiness.ts` expose `invoiceReadiness`. Ses entrées
sont la relecture conservée, son identifiant et son empreinte, la décision, la
relecture actuelle, les paramètres de numérotation et la date du jour.

La fonction vérifie :

- l'empreinte du contenu conservé ;
- la présence de la version 2, qui inclut les mentions de facturation ;
- une approbation liée à cette relecture et à cette empreinte ;
- les confirmations humaines, via `validateInvoiceDecision` ;
- l'absence de modification du contenu depuis la relecture ;
- une échéance non dépassée pour ce parcours ;
- des paramètres permettant de proposer un prochain numéro.

L'empreinte détecte une incohérence de contenu ; ce n'est pas une signature
électronique ni une protection contre une personne ayant accès à toute la base.

## 3. Comparer sans modifier

La date du contrôle courant change naturellement. On neutralise uniquement
`reviewedDay` dans une copie du contenu courant avant de calculer son empreinte.
Les dates de prestation et d'échéance restent comparées, comme les montants,
les parties et les mentions de règlement.

```ts
const comparable = structuredClone(current)
comparable.snapshot.reviewedDay = review.snapshot.reviewedDay
```

Il ne faut pas modifier `current` ou `review` directement : un diagnostic ne doit
pas altérer les données qu'il examine. Un test vérifie cette absence de mutation.

La règle sur l'échéance est un choix de ce parcours applicatif : une échéance
antérieure à aujourd'hui demande une correction et une nouvelle approbation.
Ce contrôle ne remplace pas la qualification des conditions de paiement.

## 4. Proposer n'est pas réserver

On réutilise `invoiceNumberPreview` du chapitre 20. Un compteur à 44 peut proposer
`F-0045`, mais il reste à 44 après consultation ou actualisation de la page.

Deux consultations peuvent donc afficher le même numéro. C'est normal pour un
aperçu ; ce serait incorrect pour deux émissions définitives.

Le futur traitement d'émission devra relire et vérifier les données au moment de
l'action, dans une transaction qui attribue le numéro et crée la facture de façon
cohérente. Il faudra aussi gérer le double clic, les demandes concurrentes et les
échecs de génération du document. Ne pas transformer ce diagnostic de page en
autorisation d'émettre reçue du navigateur.

## 5. Afficher le diagnostic

Dans `src/app/(frontend)/factures/relectures/[id]/page.tsx`, la section
« Bilan avant émission » présente les points bloquants et le numéro proposé.

L'approbation historique reste visible même si les données actuelles ont changé.
Le message « Aucun blocage détecté » concerne uniquement ces contrôles ; l'émission
reste explicitement indisponible. Aucun bouton d'émission n'est ajouté.

## 6. Installer et lancer les tests

Depuis la racine de l'application, après installation des prérequis du chapitre 1 :

```powershell
pnpm install --frozen-lockfile
pnpm exec playwright install chromium
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm lint
pnpm build
```

Les scripts d'intégration et de navigateur préparent la base de test isolée.
Ne pas les lancer simultanément : ils utilisent la même base. Conserver la
configuration `test.env` décrite dans les chapitres précédents ; ne jamais pointer
les tests vers les données de l'entreprise.

Le fichier `tests/unit/invoiceReadiness.unit.spec.ts` couvre notamment l'absence
d'approbation, le refus, les confirmations incomplètes, une décision liée à une
autre relecture, les modifications, l'historique V1, l'échéance et la numérotation.

Le scénario navigateur `tests/e2e/invoiceDecision.e2e.spec.ts` vérifie successivement
une approbation sans historique de numéros, une reprise correctement renseignée,
puis un changement des mentions après approbation. Il confirme aussi que la
consultation ne consomme aucun numéro et conserve la décision historique.

## 7. Mettre à jour l'installation locale

Arrêter le serveur avant de reconstruire l'application, puis exécuter :

```powershell
pnpm backup:local
pnpm build
pnpm start
```

Aucune migration SQL n'est nécessaire pour ce chapitre si le chapitre 20 est déjà
installé. Les collections et les documents historiques ne changent pas.

## Limites et suite

Ce chapitre ne livre pas l'émission définitive promise dans le parcours initial :
il en isole le contrôle préalable. Le reste est regroupé dans un lot d'émission,
puis un lot paiements, pour éviter de multiplier les petits chapitres sans finir
le parcours utilisateur. Voir la [feuille de route](./AVANCEMENT.md).
