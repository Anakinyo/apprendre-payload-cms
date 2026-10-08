# 20. Préparer les paramètres de facturation et la reprise des numéros

## Objectif

L'entreprise utilise peut-être déjà un outil de facturation. Avant de préparer
une émission dans notre application, il faut connaître le dernier numéro émis,
sa date et la manière dont ce numéro est construit.

Ce chapitre ajoute un écran privé de paramètres et un aperçu du prochain numéro.
Il permet aussi de préparer les mentions de règlement. Aucun numéro n'est réservé
ou attribué à une facture à cette étape.

Prérequis : application et base locale du chapitre 19, Node.js et pnpm installés.

## 1. Choisir un Global Payload

Une collection contient plusieurs documents : clients, brouillons ou décisions.
Un **Global** contient un seul document de configuration partagé par l'application.
Il convient ici aux paramètres de facturation de l'entreprise.

Nous créons `src/globals/InvoiceSettings.ts`, puis l'ajoutons dans la configuration :

```ts
import { InvoiceSettings } from './globals/InvoiceSettings'

// Dans buildConfig :
globals: [CompanySettings, InvoiceSettings],
```

Le nom légal, l'adresse et les coordonnées restent dans `company-settings`.
Le nouveau Global `invoice-settings` regroupe la forme juridique, le numéro de
TVA éventuellement applicable, les mentions de règlement et le projet de reprise
de numérotation. Tous ses accès sont réservés aux utilisateurs connectés.

Voir la documentation des [Globals Payload](https://payloadcms.com/docs/configuration/globals).

## 2. Préparer les mentions de règlement

Le formulaire propose le mode de règlement et les conditions d'escompte, puis
deux textes distincts : clients publics et clients professionnels privés.
Ils restent libres et limités en longueur. Aucun taux de pénalité ni montant
d'indemnité n'est présumé par l'application.

Le régime de TVA reste choisi sur chaque brouillon. Renseigner un numéro de TVA
dans les paramètres ne change pas les taux, les mentions ou les décisions existantes.
De même, aucune bascule de TVA n'est programmée au changement d'année.

Ces textes sont des paramètres en préparation. Ils ne sont pas encore intégrés
aux relectures conservées des chapitres 18 et 19 et ne bénéficient donc pas de
leurs approbations. Leur intégration à une nouvelle version figée sera une étape
distincte avant toute émission.

## 3. Décrire la numérotation existante

Nous proposons trois situations :

- **À renseigner** : aucune proposition de prochain numéro ;
- **Reprendre la numérotation existante** : numéro exact, compteur et date requis ;
- **Toute première facture de l'entreprise** : compteur à zéro et aucun historique déclaré.

La troisième option concerne une entreprise n'ayant jamais émis de facture,
pas simplement sa première facture dans notre application.

Exemple fictif : le dernier numéro est `F-2026-0044`, émis le `2026-10-01`.

| Champ | Valeur |
| --- | --- |
| Préfixe fixe | `F-2026-` |
| Nombre minimal de chiffres | `4` |
| Suffixe fixe | vide |
| Dernier compteur émis | `44` |
| Dernier numéro émis | `F-2026-0044` |
| Date de la dernière facture | `2026-10-01` |

L'aperçu indique `F-2026-0045`. Un format comme `0044/2026` utilise un préfixe vide,
le même compteur et le suffixe `/2026`.

La largeur est un minimum : après `9999`, le compteur devient `10000`.
Le préfixe est fixe : `F-2026-` ne se transforme pas automatiquement en `F-2027-`.
Le renouvellement des séries annuelles n'est pas implémenté.

## 4. Vérifier la cohérence côté serveur

`src/domain/invoiceNumbering.ts` sépare la validation et le calcul de l'aperçu.
Les entrées sont vérifiées à l'exécution, car les types TypeScript ne protègent
pas une API contre une requête mal formée.

Le mode doit être connu. Le compteur est entier, compris entre 0 et 999 999 998,
et la largeur entre 1 et 9. Préfixe et suffixe sont limités à 32 caractères chacun :
lettres non accentuées, chiffres et caractères `.`, `_`, `/`, `-`.
Un format sortant de ce cadre nécessite une évolution du modèle, pas une modification
arbitraire de l'historique pour le faire entrer dans le formulaire.

Pour une reprise, le format doit reproduire exactement le numéro saisi :

```ts
if (lastReference !== formatInvoiceReference(result, lastSequence)) {
  throw new Error('Le format et le compteur ne reproduisent pas le dernier numéro émis.')
}
```

La date doit être réelle et ne pas être future. Une date vide ou le 29 février
d'une année non bissextile est refusé. Les dates civiles utilisent `AAAA-MM-JJ`.

Le hook `beforeValidate` fusionne une modification partielle avec le groupe
`numbering` déjà enregistré. Modifier le mode de règlement ne doit pas effacer
l'historique de numérotation. Une modification incohérente est refusée avant écriture.
Le hook transforme ces erreurs métier en `APIError` avec un statut HTTP 400 et
un message public. Le formulaire affiche ainsi la raison du refus et conserve
la saisie pour permettre sa correction.

La validation vérifie la cohérence des champs, pas la véracité de l'historique :
elle n'interroge pas l'outil précédent et ne connaît pas les factures qui y seraient
émises après la saisie. Il faudra revérifier le dernier numéro lors du basculement réel.

## 5. Afficher un aperçu sans effet de bord

Depuis **Brouillons de factures**, ouvrir **Paramètres de facturation**.
La page `/factures/parametres` affiche l'historique, le prochain numéro envisagé,
l'identité du fournisseur et les mentions de règlement.

Le lien **Modifier les paramètres** ouvre le formulaire natif Payload.
Après enregistrement, revenir à la page de synthèse pour voir l'aperçu actualisé.

Le serveur vérifie la session avant de lire les Globals avec `overrideAccess: false`.
Le calcul du prochain numéro est une fonction pure : il n'écrit rien en base.
Consulter, recharger ou ouvrir deux fois la page produit le même aperçu.

Cette propriété est essentielle : une simple consultation ne doit pas consommer
un numéro. Le futur compteur officiel devra, lui, être avancé au sein de la même
transaction que l'émission de la facture et protégé contre les doubles émissions.

## 6. Mettre à jour la base locale

Arrêter le serveur, puis exécuter depuis la racine du projet :

```powershell
pnpm backup:local
pnpm upgrade:chapter20
pnpm generate:types
pnpm test
pnpm lint
pnpm build
pnpm start
```

Le script crée une copie SQLite avant d'ajouter la table `invoice_settings`.
Il vérifie qu'une base du chapitre 19 est présente. Une nouvelle exécution
ne réinitialise pas les paramètres enregistrés. Les anciens documents sont conservés.

Les paramètres ne sont pas préremplis avec des données de démonstration : chaque
entreprise renseigne son propre historique.

## 7. Vérifier le fonctionnement

Sur un nouveau clone, `pnpm install` installe les outils et
`pnpm exec playwright install chromium` installe le navigateur de test.
Les tests utilisent `payload-test.db`, qui est réinitialisée par les scripts
prévus à cet effet. Ne pas y substituer la base métier.

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
```

`pnpm test` enchaîne ces trois commandes. Ne pas lancer deux suites utilisant
la base de test simultanément.

- `invoiceNumbering.unit.spec.ts` : reprise, suffixes, incohérences, dates réelles,
  première facture, limites et absence de changement d'année automatique ;
- `invoiceSettings.int.spec.ts` : accès privés, modifications partielles,
  refus d'une configuration incohérente et lectures sans écriture ;
- `invoiceSettingsUpgrade.int.spec.ts` : migration relancée sans perte de données ;
- `invoiceSettings.e2e.spec.ts` : erreur de format visible, correction dans le formulaire, synthèse, rechargement
  sans consommation de numéro et affichage sur ordinateur et petit écran.

## Cadre et suite

Une facture exige une numérotation unique, chronologique et continue. Les champs
de ce chapitre préparent cette reprise ; ils ne constituent pas encore un registre
de factures émises. Les règles et mentions applicables sont détaillées par
[Service Public](https://www.service-public.gouv.fr/entreprendre/vosdroits/F31808?profil=entrepreneur-individuel).

La franchise en base de TVA n'exclut pas l'entreprise de la réforme de la facturation
électronique : voir la [fiche de la DGFiP](https://www.impots.gouv.fr/professionnel/questions/franchise-en-base-micro-entrepreneur-ou-auto-entrepreneur-suis-je-concerne).
La génération future d'un PDF et son circuit de transmission devront être traités
séparément. Sources consultées le 8 octobre 2026.

La prochaine étape intégrera les mentions de facturation dans une nouvelle version
figée à relire. L'émission définitive, son compteur transactionnel et la transmission
restent à construire. Le circuit existant continue à servir pour les factures réelles.
