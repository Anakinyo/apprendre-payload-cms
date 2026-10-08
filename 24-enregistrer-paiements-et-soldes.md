# 24. Enregistrer les paiements et suivre le solde

## Objectif

Une facture émise et un encaissement sont deux événements différents. Le PDF décrit
la facture à son émission ; il n'est pas modifié lorsqu'un paiement arrive.

Ce chapitre ajoute un journal privé de paiements, un solde calculé et une page
accessible depuis « Suivre les paiements » sur la facture conservée. La saisie est
manuelle : il n'y a ni connexion bancaire, ni prélèvement, ni mouvement d'argent.

## 1. Créer une collection de journal

`src/collections/InvoicePayments.ts` définit `invoice-payments`, avec :

- la facture émise concernée ;
- le type : encaissement ou annulation de saisie ;
- le montant en centimes entiers et la date d'encaissement ;
- le mode : virement, chèque, espèces, carte ou autre ;
- une référence libre ou un motif de correction ;
- l'auteur et les dates d'enregistrement ;
- l'identifiant de demande et, pour une annulation, le paiement d'origine.

La collection refuse les modifications et suppressions. Le journal n'est accessible
qu'après authentification. L'auteur est déterminé par le serveur, pas par un champ
que le navigateur pourrait falsifier.

Les choix de mode sont descriptifs. Ils ne valident pas les règles particulières
d'un règlement ni les justificatifs associés.

## 2. Convertir les euros sans arrondis cachés

Le formulaire accepte `100,25` ou `100.25`. `paymentCents`, dans
`src/domain/invoicePayments.ts`, sépare les euros et les décimales avant de calculer
le montant entier `10025`.

```ts
const cents = Number(whole) * 100 + Number(decimals.padEnd(2, '0'))
```

La validation préalable limite le format à deux décimales et contrôle le résultat
avec `Number.isSafeInteger`. On refuse les montants nuls, négatifs, la notation
scientifique et les décimales supplémentaires au lieu de les arrondir silencieusement.

L'API reçoit des centimes entiers et les valide également. Une validation dans le
navigateur seule serait contournable. Le formulaire natif de l'administration
affiche explicitement un champ en centimes ; le formulaire métier utilise les euros.

## 3. Calculer plutôt que stocker un statut

`paymentBalance` calcule les encaissements nets, le reste à payer et le statut :

- aucun montant reçu : « À payer » ;
- une partie du montant reçue : « Partiellement payée » ;
- solde nul : « Payée ».

Une échéance dépassée est une information distincte, affichée seulement si le solde
reste positif. Une échéance au jour du contrôle n'est pas considérée dépassée.

Le calcul porte sur le total figé de la facture émise, jamais sur le brouillon
modifiable. Les lectures utilisent `pagination: false` pour compter toutes les
écritures. Oublier ce paramètre pourrait limiter le calcul à la première page.

Cette première version charge tout le journal d'une facture. Si le volume devient
important, on pourra conserver un calcul global fiable tout en paginant l'affichage.

## 4. Refuser les saisies incohérentes

Le hook exige une session, une confirmation explicite et une transaction. Pour un
encaissement, il vérifie notamment :

1. que la facture émise existe ;
2. que le montant est positif et entier ;
3. que le mode de paiement est connu ;
4. que la date est réelle, entre l'émission et aujourd'hui ;
5. que le montant ne dépasse pas le reste à payer.

La date du jour utilise le fuseau Europe/Paris. Les paiements antérieurs à l'émission
sont hors de ce premier parcours : ils demanderaient une gestion dédiée des acomptes.
Les trop-perçus et remboursements ne sont pas représentés par des montants négatifs.

Le calcul du solde et l'insertion utilisent la même transaction SQLite immédiate,
avec la sérialisation déjà mise en place. Deux requêtes concurrentes ne peuvent
donc pas valider chacune un paiement sur le même ancien solde.

## 5. Protéger les doubles soumissions

Le formulaire crée un UUID `requestKey` pour chaque saisie. Cet identifiant possède
un index unique. Une nouvelle soumission de la même demande ne crée pas une seconde
écriture, même si le solde permettrait d'encaisser les deux montants.

Le formulaire renouvelle cet identifiant uniquement après un succès confirmé. En
cas d'erreur réseau, il le conserve. Si le serveur a enregistré l'écriture mais
que la réponse a été perdue, actualiser permet de vérifier l'historique.

Cette protection identifie une demande technique, pas un virement bancaire. Deux
saisies volontaires distinctes du même virement restent possibles : la référence
et la vérification humaine demeurent importantes sans rapprochement bancaire.

## 6. Corriger sans effacer

Pour corriger un montant mal saisi, choisir « Annulation de saisie », sélectionner
l'encaissement et renseigner un motif. Le serveur reprend lui-même le montant,
la date d'encaissement et le mode d'origine. L'annulation possède sa propre date
d'enregistrement et son auteur.

Le paiement d'origine reste consultable et est marqué « Saisie annulée ». Une relation
unique empêche deux annulations du même paiement. Il est impossible d'annuler une
écriture d'une autre facture ou une annulation elle-même.

Saisir ensuite le paiement correct comme un nouvel encaissement. Une annulation
de saisie **ne représente ni un remboursement réel ni un avoir**. Ces opérations
restent à traiter dans un parcours distinct.

## 7. Construire la page privée

`src/app/(frontend)/factures/[id]/paiements/page.tsx` charge la facture émise et ses
écritures après authentification. Ici `id` désigne la facture émise, pas le brouillon.
Les lectures conservent `overrideAccess: false` et `depth: 0`.

La page présente le solde, le formulaire et l'historique. Le formulaire client
`InvoicePaymentForm` utilise l'API Payload, affiche les erreurs et actualise les
données serveur après une saisie réussie. Les champs s'adaptent à l'encaissement
ou à l'annulation de saisie.

Le PDF et le numéro de facture restent inchangés. Aucun message n'est envoyé au client.

## 8. Lancer les vérifications

Depuis l'application, avec les prérequis des chapitres précédents :

```powershell
pnpm install --frozen-lockfile
pnpm generate:types
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm lint
pnpm build
```

Les suites d'intégration et navigateur partagent la base de test jetable ; les
lancer successivement. Ne pas modifier `test.env` pour y placer la base réelle.

Les tests unitaires couvrent les conversions, les soldes, les échéances et les
annulations. Les tests d'intégration réutilisent la création de factures du chapitre
23 pour vérifier les permissions, les dates et montants refusés, les doublons,
les corrections, le calcul au-delà de dix écritures et les requêtes simultanées.
Ils vérifient également que le PDF et le compteur des factures ne changent pas.

Le scénario navigateur enregistre un règlement partiel, annule cette saisie avec
un motif, puis enregistre le règlement complet. Il vérifie le solde après
actualisation et l'affichage sur ordinateur et petit écran.

## 9. Mettre à jour la base locale

Serveur arrêté et chapitre 23 déjà installé :

```powershell
pnpm backup:local
pnpm upgrade:chapter24
pnpm build
pnpm start
```

La migration ajoute la table et ses index, après une sauvegarde supplémentaire.
Elle peut être relancée sans vider le journal. Elle ne crée aucun encaissement et
ne modifie aucune facture existante.

## Suite et limites

Le suivi est disponible facture par facture, pas encore sous forme de tableau de
trésorerie global. Il ne produit pas de déclaration URSSAF, ne réalise pas de
rapprochement bancaire et ne gère pas les avoirs, acomptes ou remboursements.
Les essais sur des exemples représentatifs et la préparation de l'exploitation
locale restent nécessaires avant une utilisation quotidienne.
