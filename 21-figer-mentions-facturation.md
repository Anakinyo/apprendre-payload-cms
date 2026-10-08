# 21. Figer les mentions de facturation dans une relecture

## Objectif

Les paramètres du chapitre 20 peuvent encore changer. Une personne qui approuve
une relecture doit pourtant savoir exactement quels textes et quelles coordonnées
elle a vérifiés. Nous allons donc les copier dans chaque nouvelle relecture.

Ce chapitre introduit une deuxième version du contenu conservé. Les anciennes
relectures restent lisibles et leurs décisions restent dans l'historique.
L'application prépare toujours des factures : elle n'en émet pas encore.

Prérequis : avoir terminé le chapitre 20, notamment sa mise à jour de base.

## 1. Distinguer paramètres actuels et données relues

Les paramètres actuels servent à préparer un nouveau document. La relecture
conservée contient une copie des informations effectivement présentées à l'utilisateur.
Sa page ne doit pas relire les paramètres actuels pour afficher les anciens textes.

Exemple : une relecture contient « Conditions A ». L'entreprise remplace ensuite
les paramètres par « Conditions B ». L'ancienne page continue à afficher A ;
la préparation actuelle et la prochaine relecture affichent B.

`src/domain/invoiceReview.ts` ajoute un objet `billing` au contenu figé :

- forme juridique et numéro de TVA du fournisseur ;
- mode de règlement et conditions d'escompte ;
- catégorie de règlement retenue et texte correspondant ;
- banque, IBAN et BIC présents dans les paramètres de l'entreprise.

Les coordonnées bancaires sont facultatives dans cette préparation. Elles sont
copiées telles quelles ; leur présence ne valide ni leur format ni leur titulaire.
Le contrôle humain mentionne désormais aussi les coordonnées bancaires.

Le projet de numérotation est exclu de cette copie. Le prochain numéro envisagé
n'est pas un numéro réservé à la relecture. Son attribution définitive appartiendra
au futur parcours d'émission.

## 2. Choisir les mentions correspondant au client

Le choix repose sur la catégorie enregistrée dans la fiche client :

| Catégorie | Mentions copiées |
| --- | --- |
| Collectivité ou administration | Clients publics |
| Entreprise | Clients professionnels privés |
| Association ou particulier | À qualifier, approbation bloquée dans ce parcours |

Une association n'est pas automatiquement assimilée à un client professionnel.
Les catégories non prises en charge nécessiteront une qualification et des règles
adaptées dans une prochaine évolution. Ne pas changer artificiellement la catégorie
d'un client pour contourner ce contrôle.

Seul le texte sélectionné est copié. Modifier les mentions publiques ne change
donc pas une relecture destinée à une entreprise privée.

## 3. Versionner le contenu conservé

Le champ `review` est déjà un champ JSON Payload. Nous pouvons faire évoluer
son contenu sans ajouter une colonne SQL. Les nouvelles données portent
`snapshot.version: 2`, tandis que les documents précédents gardent `version: 1`.

Le type TypeScript accepte les deux formes :

```ts
export type InvoiceReviewData =
  | InvoiceReviewV1
  | ReturnType<typeof buildInvoiceReview>
```

La valeur `version` permet ensuite de choisir le bon affichage. C'est une union
discriminante : après le test `snapshot.version === 2`, TypeScript sait que les
mentions `billing` sont disponibles. Il n'est pas nécessaire de supposer qu'elles
existent sur toutes les anciennes relectures.

Nous ne réécrivons pas les anciens documents pour leur ajouter les textes actuels.
Cela reviendrait à prétendre qu'une personne a relu des informations absentes au
moment de sa décision.

## 4. Construire la nouvelle version côté serveur

Dans le hook de `src/collections/InvoiceReviews.ts`, le serveur lit aussi le Global
`invoice-settings`, en conservant la requête et les permissions de l'utilisateur :

```ts
const settings = await req.payload.findGlobal({
  slug: 'invoice-settings',
  req,
  overrideAccess: false,
  depth: 0,
})
const review = buildInvoiceReview(draft, client, company, parisDay(), settings)
```

Les montants sont toujours recalculés et les notes internes restent exclues.
L'empreinte SHA-256 couvre le contenu complet, y compris les nouvelles mentions.
Une empreinte est un moyen de comparer des contenus ; elle n'est pas une signature
numérique et ne protège pas la base contre son propriétaire.

La préparation signale les informations manquantes : forme juridique, mode de
règlement, escompte ou texte correspondant au client. Le parcours implémenté
prend actuellement en charge le fournisseur entrepreneur individuel. Une autre
forme juridique reste à modéliser avant approbation.

Pour le parcours avec TVA facturée, le numéro de TVA fournisseur est demandé.
C'est une règle de ce premier périmètre logiciel, pas une description exhaustive
des exceptions légales. Aucune vérification auprès d'un annuaire n'est effectuée.
Le régime et le taux du brouillon restent inchangés.

## 5. Lier la décision aux mentions réellement relues

Avant une nouvelle approbation, `src/collections/InvoiceDecisions.ts` :

1. vérifie l'empreinte de la relecture conservée ;
2. exige la version 2 et les confirmations humaines ;
3. reconstruit les données avec les paramètres actuels ;
4. compare leur empreinte à celle de la relecture.

La date de relecture est neutralisée dans cette dernière comparaison pour qu'un
simple changement de jour ne rende pas la version obsolète. Les autres données
et les contrôles restent comparés.

Modifier le texte utilisé, le mode de règlement ou l'IBAN empêche donc d'approuver
l'ancienne relecture : il faut en figer une nouvelle. Un refus motivé reste possible.
Modifier uniquement un texte destiné à une autre catégorie de client, ou le projet
de numérotation, ne change pas le contenu relu.

Une ancienne version 1 peut toujours être consultée ou refusée. Elle ne peut plus
recevoir de nouvelle approbation. Une approbation enregistrée auparavant est conservée
comme décision historique ; elle ne couvre pas les nouvelles mentions manquantes.

## 6. Vérifier l'interface

Depuis **Brouillons de factures**, ouvrir la préparation d'un brouillon. Le lien
**Paramètres de facturation** permet de compléter les informations nécessaires.

La section **Mentions de facturation relues** affiche les textes et les coordonnées
sur les nouvelles versions. Après **Figer une relecture**, changer un paramètre,
puis recharger la page conservée : le texte doit rester identique.
Revenir à la préparation et figer une nouvelle relecture doit afficher le nouveau texte.

Sur une ancienne version, une indication précise que les mentions ne sont pas
incluses et le choix d'approbation est désactivé. Le serveur impose également
cette règle à l'API, indépendamment de l'état du formulaire.

Les pages restent privées. Les textes sont affichés comme du texte React, sans
interpréter du HTML contenu dans les paramètres. Les retours à la ligne sont conservés.

## 7. Installer et lancer les tests

Sur un nouveau clone, commencer par `pnpm install` et
`pnpm exec playwright install chromium`. Les suites utilisent la base dédiée
`payload-test.db`. Ne pas les lancer simultanément ni remplacer cette base par
la base de travail.

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
```

`pnpm test` exécute les trois suites dans cet ordre.

- Les tests unitaires contrôlent la sélection des textes, les informations
  manquantes, les coordonnées bancaires et le refus d'une nouvelle approbation V1.
- Les tests d'intégration vérifient l'immutabilité, le refus d'une version obsolète,
  la conservation des décisions historiques et l'absence de réservation de numéro.
- Les tests de navigateur vérifient l'affichage des anciens et nouveaux textes,
  le refus d'une ancienne version et les mises en page sur ordinateur et petit écran.

Les anciens enregistrements sont simulés par des écritures directes de l'adaptateur
uniquement dans la base de test, avec une vérification explicite de son chemin.
Ce mécanisme de préparation des tests n'est pas exposé dans l'application.

## 8. Mettre à jour l'application locale

Arrêter le serveur, puis exécuter :

```powershell
pnpm backup:local
pnpm test
pnpm lint
pnpm build
pnpm start
```

Aucune migration SQL supplémentaire n'est nécessaire si le chapitre 20 est à jour.
Les anciennes relectures conservent leur contenu et leur empreinte. Le numéro de
version 2 est attribué uniquement lors de la création d'une nouvelle relecture.

## Suite

Cette étape garantit quel contenu a été présenté à la personne qui décide.
Elle ne certifie pas le caractère légalement adapté des textes saisis. Les
[mentions obligatoires détaillées par Service Public](https://www.service-public.gouv.fr/entreprendre/vosdroits/F31808?profil=entrepreneur-individuel)
restent à vérifier pour la situation réelle de l'entreprise et de ses clients
(source consultée le 8 octobre 2026).

La suite portera sur l'émission définitive, l'attribution transactionnelle du numéro
et le document généré. Aucune facture, transmission ou déclaration n'est déclenchée
par une relecture ou une approbation interne.
