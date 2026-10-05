# Apprendre Payload CMS - Episode 6 : calculer les montants

## Objectif

Calculer les montants du devis côté serveur et tester les règles indépendamment
du CMS. Copier les conditions de l'entreprise à la création du devis pour
conserver les choix faits lors de sa saisie.

## 1. Définir les règles avant le code

Les prix sont des entiers en centimes. Les quantités ont deux décimales au
maximum et valent au moins 0,01. Une prestation gratuite est autorisée.

Chaque montant de ligne est arrondi au centime, une moitié de centime étant
arrondie vers le haut. Le total HT est la somme des lignes arrondies. La TVA
est calculée une fois sur ce total puis arrondie de la même façon.

Le taux est un entier en centièmes de pourcent : `2000` signifie 20 %, `550`
signifie 5,5 %. Cette étape utilise un seul taux pour le devis, entre 0 et
100 %. Le taux par défaut est zéro et doit être configuré selon l'entreprise.
Une mention TVA est conservée séparément du taux. Ces champs ne déterminent
pas automatiquement le régime fiscal de l'entreprise.

Exemple : 1,5 jour à 35000 centimes donne 52500 centimes HT. Avec un taux
de 2000, la TVA vaut 10500 centimes et le total vaut 63000 centimes.

## 2. Isoler une fonction pure

Créer `src/domain/quoteAmounts.ts`. Une fonction pure reçoit des valeurs et
retourne des valeurs : elle ne lit ni la base ni les variables d'environnement.

La fonction `quantityHundredths` transforme la quantité en centièmes et vérifie
qu'elle peut être représentée avec deux décimales. Une petite tolérance permet
les imprécisions binaires de JavaScript. Les prix doivent passer
`Number.isSafeInteger` et être positifs ou nuls.

Pour éviter les erreurs de multiplication décimale, convertir les entiers en
`BigInt` avant les calculs. Pour des montants positifs, la formule de ligne est :

```ts
const lineTotal = (BigInt(quantityHundredths(quantity)) * BigInt(unitPriceCents) + 50n) / 100n
```

L'ajout de 50 avant une division entière par 100 réalise l'arrondi au plus
proche avec les demis vers le haut. La TVA utilise le même principe avec
5000 et 10000. Vérifier les limites avant de reconvertir le résultat en
`number`, car JSON et les champs Payload stockent ici des nombres ordinaires.

Le code complet est disponible dans
[quoteAmounts.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/domain/quoteAmounts.ts).

## 3. Relier le calcul à Payload

Dans `Quotes`, ajouter un hook `beforeChange`. Les champs et les valeurs par
défaut sont déjà validés à cette étape. Le hook récupère les lignes et le taux
dans `data`, ou dans `originalDoc` lors d'une mise à jour partielle.

```ts
const lines = data.lines ?? originalDoc?.lines
const taxRateBps = data.taxRateBps ?? originalDoc?.taxRateBps ?? 0
const amounts = calculateQuoteAmounts(lines, taxRateBps)
```

Ajouter aux lignes `totalCents`, et au devis `subtotalCents`, `taxCents` et
`totalCents`. Le hook remplace toujours ces champs par le calcul du serveur.
Les champs calculés sont `readOnly` dans l'administration. Ce réglage facilite
la saisie; c'est le recalcul serveur qui garantit leur exactitude via l'API.

Ajouter aussi un validateur de quantité utilisant `quantityHundredths`, pour
afficher une erreur sur le champ avant l'enregistrement.

Les montants restent affichés en centimes à cette étape. Une présentation en
euros et le document PDF seront ajoutés dans le parcours suivant.

## 4. Copier les valeurs de l'entreprise

Ajouter `taxRateBps` dans `CompanySettings.quoteDefaults`, puis les champs
suivants au devis : `validityDays`, `paymentTermsDays`, `paymentMethod`,
`vatNotice` et `taxRateBps`.

Un hook `beforeValidate` lit le Global seulement pour `operation === 'create'`.
Il conserve chaque valeur explicite du devis et utilise les réglages de
l'entreprise pour les autres. Employer `??` plutôt que `||` conserve un délai
de zéro jour ou un taux de zéro.

```ts
validityDays: data.validityDays ?? company.quoteDefaults?.validityDays ?? 30
```

Les valeurs de repli sont 30 jours de validité, 15 jours de délai et le
virement. Une mise à jour du Global ne réécrit pas les devis existants.
Les coordonnées de l'entreprise et du client ne sont pas encore figées :
elles le seront lors de la production d'une version du document.

Enregistrer les deux hooks dans `Quotes` et régénérer les types :

```ts
hooks: {
  beforeValidate: [applyCompanyDefaults, validateMissionClient],
  beforeChange: [calculateAmounts],
}
```

```powershell
pnpm generate:types
```

## 5. Ajouter les tests unitaires

Créer `tests/unit/quoteAmounts.unit.spec.ts`. Les dépendances et la configuration
initiale sont décrites dans le [chapitre 4](./04-configurer-entreprise-global.md#installer-et-configurer-les-tests).
Vitest est déjà installé. Un test unitaire complet doit importer le lanceur
de tests et la fonction à vérifier :

```ts
import { describe, expect, it } from 'vitest'
import { calculateQuoteAmounts } from '../../src/domain/quoteAmounts'

describe('Montants du devis', () => {
  it('calcule 1,5 jour avec une TVA de 20 %', () => {
    expect(calculateQuoteAmounts([{ quantity: 1.5, unitPriceCents: 35000 }], 2000))
      .toEqual({ lineTotalsCents: [52500], subtotalCents: 52500, taxCents: 10500, totalCents: 63000 })
  })
})
```

Compléter ce fichier avec les cas suivants :

- 1,5 jour à 350 euros avec une TVA de 20 % ;
- deux demi-centimes, arrondis séparément à un centime chacun ;
- une TVA de 0,5 centime arrondie à un centime ;
- une quantité de 0,29 et une prestation gratuite ;
- zéro, les nombres négatifs, `NaN`, `Infinity` et trois décimales ;
- les prix fractionnaires et les dépassements de capacité ;
- un devis vide et un taux invalide.

Exemple :

```ts
expect(calculateQuoteAmounts([{ quantity: 1.5, unitPriceCents: 35000 }], 2000))
  .toEqual({ lineTotalsCents: [52500], subtotalCents: 52500, taxCents: 10500, totalCents: 63000 })
```

Étendre `vitest.config.mts` :

```ts
include: ['tests/int/**/*.int.spec.ts', 'tests/unit/**/*.unit.spec.ts']
```

Dans `package.json`, séparer les commandes en conservant leur configuration :

```json
{
  "test": "pnpm run test:unit && pnpm run test:int && pnpm run test:e2e",
  "test:unit": "cross-env NODE_OPTIONS=--no-deprecation vitest run --config ./vitest.config.mts tests/unit",
  "test:int": "cross-env NODE_OPTIONS=--no-deprecation DOTENV_CONFIG_PATH=./test.env vitest run --config ./vitest.config.mts tests/int"
}
```

Les tests unitaires ne démarrent pas Payload. Les tests d'intégration vérifient
le recalcul lors d'une mise à jour partielle, le remplacement d'un total fourni
par la requête, la copie des réglages et leur conservation après modification
du Global. Les tests navigateur continuent de contrôler les formulaires.

Pour exécuter uniquement les tests unitaires :

```powershell
pnpm test:unit
pnpm exec vitest --config ./vitest.config.mts tests/unit
```

La première commande exécute puis termine la suite. La deuxième reste en mode
surveillance et relance les tests après une modification; `Ctrl+C` l'arrête.
Les tests unitaires fonctionnent avec `pnpm dev` ouvert; les tests navigateur
exigent de l'arrêter. `pnpm test` enchaîne les trois suites avant publication.

## 6. Vérifier dans l'application

Arrêter le serveur de développement avant les tests navigateur. Exécuter :

```powershell
pnpm test
pnpm lint
pnpm build
pnpm dev
```

Configurer les valeurs par défaut de l'entreprise, puis créer un devis.
Ajouter 1,5 jour à 35000 centimes, enregistrer et vérifier le total HT de
52500. Modifier le taux à 2000 : après enregistrement, le total doit être
63000. Changer ensuite les réglages de l'entreprise et rouvrir le devis :
ses conditions doivent être conservées.

## Prochaine étape

Préparer une version du devis avec les coordonnées figées, puis générer un
aperçu PDF lisible. La validation humaine reste nécessaire avant tout envoi
au client. Les remises et les taux de TVA multiples ne sont pas implémentés
dans cet épisode.
