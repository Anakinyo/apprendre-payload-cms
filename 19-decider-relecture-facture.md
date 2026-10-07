# 19. Enregistrer une décision humaine sur une relecture de facture

## Objectif

L'entreprise peut approuver ou refuser une version conservée, sans émettre de
facture. Une décision concerne une relecture précise, pas un brouillon qui peut
encore évoluer. Le parcours reste local et réservé aux utilisateurs connectés.

Prérequis : avoir terminé le chapitre 18 et disposer d'une base locale à jour.
Cette étape ne remplace pas l'outil utilisé pour les factures réellement émises.

## 1. Séparer relecture et décision

Nous ajoutons une collection `invoice-decisions`. Elle contient :

- `review` : relation unique vers `invoice-reviews` ;
- `decision` : `approved` ou `rejected` ;
- `reviewed` : confirmation explicite de la relecture ;
- `checks` : six confirmations manuelles distinctes ;
- `reason` : commentaire, obligatoire pour un refus ;
- `reviewReference` et `digest` : référence et empreinte de la version relue ;
- `decidedAt`, `decidedBy`, `reviewerEmail` : date et auteur enregistrés par le serveur.

Pourquoi une collection distincte ? La relecture reste immuable. Nous ne lui
ajoutons pas un statut modifiable qui masquerait une décision précédente.
La relation `unique: true` impose une seule décision par relecture dans SQLite.
Une nouvelle décision nécessite une nouvelle relecture, même après un refus.
Voir la documentation officielle du [champ Relationship](https://payloadcms.com/docs/fields/relationship).

## 2. Valider les données métier avant de les enregistrer

Le fichier `src/domain/invoiceDecision.ts` centralise les règles, sans dépendre
de React ni de la base. Il est donc facile à tester indépendamment.

```ts
if (data.reviewed !== true) {
  throw new Error('Confirmez la relecture de cette version.')
}
```

Le booléen doit réellement être `true` : la chaîne `"true"` ne suffit pas.
Un refus exige un motif non vide après suppression des espaces en début et fin.
Le commentaire est limité à 2 000 caractères.

Pour une approbation, les informations manquantes conservées dans la relecture
doivent être absentes. Chaque point doit aussi être confirmé individuellement :

1. identité juridique du fournisseur ;
2. prestation, dates et bon de commande ;
3. régime et mentions TVA ;
4. conditions de règlement ;
5. circuit de transmission ;
6. continuité de la future numérotation.

Ces cases enregistrent une déclaration humaine. Elles ne prouvent pas la
conformité juridique et ne consultent aucun annuaire ou service externe.

## 3. Utiliser un hook Payload pour protéger tous les chemins d'entrée

Dans `src/collections/InvoiceDecisions.ts`, `beforeValidate` vérifie
l'authentification, relit la version choisie avec `overrideAccess: false`, puis
recalcule son empreinte SHA-256. Une empreinte incohérente interdit toute décision.

Le serveur ne fait pas confiance aux champs d'auteur, de date ou d'empreinte
envoyés par le navigateur : il les remplace par ses propres valeurs.
Les mêmes règles s'appliquent au formulaire, à l'API REST et à la Local API.
Le fonctionnement de `beforeValidate` est décrit dans la documentation officielle
des [hooks de collection](https://payloadcms.com/docs/hooks/collections).

Pour une approbation, le hook reconstruit également la relecture à partir du
brouillon, du client et des paramètres actuels de l'entreprise. Il neutralise
uniquement la date de relecture dans cette comparaison. Les données et les
contrôles doivent correspondre à l'empreinte conservée ; sinon il faut figer
une nouvelle version. Un refus reste possible sur une ancienne version.

Les accès `update` et `delete` sont interdits. Des hooks refusent aussi ces
opérations avec `overrideAccess: true`. Ce mécanisme protège l'application,
pas le fichier SQLite contre son propriétaire : ce n'est pas un audit certifié.

Le contrôle d'existence offre un message compréhensible ; la contrainte unique
en base est le dernier garde-fou contre deux décisions simultanées.

## 4. Ajouter le formulaire à la page privée

`src/components/InvoiceDecisionForm.tsx` présente un choix approuver/refuser,
les cases de contrôle et le motif. Par prudence, le choix initial est le refus.
L'approbation est indisponible si la relecture contient des informations manquantes.

Le formulaire envoie une demande explicite à `/api/invoice-decisions`.
Les contrôles HTML facilitent la saisie ; ils ne remplacent jamais les règles
du serveur. Un échec laisse la saisie visible et affiche une erreur.
Après succès, `router.refresh()` recharge la page avec la décision conservée.

La page `src/app/(frontend)/factures/relectures/[id]/page.tsx` vérifie la session
avant de lire les données métier. Une décision existante remplace le formulaire
par son résultat, son auteur, sa date, son commentaire et son empreinte.
Le formulaire est vérifié sur grand écran et sur une largeur de 390 pixels.

## 5. Mettre à jour une base existante

Arrêter le serveur local avant la sauvegarde, la migration et les tests.
Depuis la racine du projet :

```powershell
pnpm backup:local
pnpm upgrade:chapter19
pnpm generate:types
pnpm test
pnpm lint
pnpm build
pnpm start
```

La migration ajoute la table, ses index et la relation des verrous Payload.
Elle crée d'abord une copie SQLite avec un nom unique et ne supprime aucune
donnée existante. Elle est réservée à `file:./payload-archiviste.db` et peut être
relancée. Sur une installation neuve, suivre les étapes initiales du cours pour
créer le schéma complet ; les scripts de mise à jour visent les bases existantes.

## 6. Installer et lancer les tests

Les outils ont été installés dans les premiers chapitres. Sur un nouveau clone,
`pnpm install` installe les dépendances et `pnpm exec playwright install chromium`
installe le navigateur utilisé pour les tests de parcours.

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
```

`pnpm test` lance les trois suites. La préparation réinitialise uniquement
`payload-test.db`, jamais la base métier. Ne pas modifier ce chemin dans `test.env`.
Ne pas lancer plusieurs suites utilisant cette base simultanément.

- `tests/unit/invoiceDecision.unit.spec.ts` : booléens stricts, six contrôles,
  informations manquantes, refus motivé et commentaire trop long ;
- `tests/int/invoiceDecision.int.spec.ts` : accès, métadonnées imposées par le
  serveur, immutabilité, versions obsolètes et décisions simultanées ;
- `tests/int/invoiceDecisionUpgrade.int.spec.ts` : migration relancée, conservation
  des données et unicité en base ;
- `tests/e2e/invoiceDecision.e2e.spec.ts` : refus, nouvelle relecture, approbation
  explicite, persistance et affichage sur ordinateur et petit écran.

## Limites et suite

Une relecture approuvée reste **non émise**. Aucun numéro officiel, PDF de
facture, e-mail, dépôt Chorus Pro ou paiement n'est créé. Une modification
ultérieure ne transfère pas l'approbation : tout futur parcours d'émission devra
revérifier la version approuvée au moment de son utilisation.

La suite préparera l'émission définitive et la numérotation, après confirmation
des mentions applicables, du circuit de transmission et de la dernière facture
déjà émise dans l'outil existant. Aucun changement automatique de TVA n'est prévu.
