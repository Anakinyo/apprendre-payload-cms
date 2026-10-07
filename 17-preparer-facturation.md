# Chapitre 17 - Préparer la facturation sans émettre de facture

## Objectif

L'entreprise souhaite préparer les prestations à facturer, puis suivre les
paiements. Nous commençons par des **brouillons modifiables**, volontairement
distincts des futures factures émises. Le système ne remplace pas encore un
outil de facturation utilisé en production.

Ce chapitre ajoute une collection, des validations métier, des calculs côté
serveur et une liste dans l'espace de travail. Il ne crée ni numéro officiel,
PDF de facture, dépôt sur une plateforme, paiement ou déclaration.

## 1. Cadrer le circuit avant de coder

Les règles ci-dessous ont été consultées le **7 octobre 2026**. Les vérifier à
nouveau avant une utilisation réelle ; les sources officielles et un conseil
adapté à l'entreprise priment sur ce cours.

Pour les fournisseurs du secteur public, la transmission passe par Chorus Pro.
Cela concerne notamment les prestations facturées à des collectivités ou à des
services de l'État. Chorus Pro n'est pas le circuit unique de toutes les factures
privées. [Source : ministère de l'Économie](https://www.economie.gouv.fr/espace-fournisseurs/le-ministere-votre-ecoute/je-souhaite-deposer-ma-facture-sur-chorus-pro).

Pour les opérations entre entreprises relevant de la réforme, la réception
électronique est prévue à partir du 1er septembre 2026 ; l'émission obligatoire
pour les PME et micro-entreprises à partir du 1er septembre 2027. Le périmètre
dépend des opérations. [Source : DGFiP, calendrier](https://www.impots.gouv.fr/professionnel/questions/partir-de-quand-suis-je-concerne-par-la-reforme-de-la-facturation).

La franchise en base de TVA ne dispense pas de la réforme : l'entreprise peut
être assujettie sans être redevable. [Source : DGFiP, franchise et micro-entreprises](https://www.impots.gouv.fr/professionnel/questions/franchise-en-base-micro-entrepreneur-ou-auto-entrepreneur-suis-je-concerne).

Une facture émise demande notamment une numérotation chronologique continue,
les identités, les dates, les prestations et les mentions pertinentes. Les
mentions EI et de franchise doivent correspondre à la situation réelle.
Cette liste n'est pas exhaustive. [Source : Service Public, mentions obligatoires](https://entreprendre.service-public.gouv.fr/vosdroits/F31808?lang=fr).

Notre brouillon n'est donc ni une facture électronique conforme ni une facture
prête à remettre au client. Aucun remplacement de l'outil existant n'est conseillé
tant que l'émission, les contrôles et le circuit de transmission ne sont pas validés.

## 2. Séparer préparation, émission et encaissement

Le parcours cible distingue trois objets :

1. **Brouillon** : modifiable et supprimable, sans numéro officiel.
2. **Facture émise** : document figé après contrôle et confirmation humaine,
   avec une numérotation propre aux factures.
3. **Paiement** : encaissement daté, associé à une facture, éventuellement partiel.

Seul le premier objet est implémenté ici. Le numéro technique Payload dans
l'URL d'un brouillon n'est jamais un numéro de facture. Les séries de numéros
des devis ne sont pas utilisées pour les factures.

La suite devra traiter séparément les avoirs, les acomptes, les annulations,
les règles de dates et les données à transmettre. Un total facturé ne sera pas
assimilé à un montant encaissé pour préparer des données déclaratives.

## 3. Modéliser le brouillon

La collection `src/collections/InvoiceDrafts.ts` utilise le slug `invoice-drafts`.

| Champ | Usage |
| --- | --- |
| `title` | Intitulé du brouillon |
| `client` | Client obligatoire, indexé |
| `mission` | Mission facultative appartenant au client |
| `serviceDay` | Date de prestation envisagée, au format civil `AAAA-MM-JJ` |
| `dueDay` | Échéance envisagée, facultative |
| `purchaseOrder` | Référence du bon de commande, si connue |
| `lines` | De 1 à 100 prestations |
| `vatTreatment` | À confirmer, sans TVA, ou TVA facturée |
| `taxRateBps` | Taux en centièmes de pourcent |
| `vatNotice` | Mention TVA envisagée, à relire |
| `internalNotes` | Notes internes |

Les lignes comportent description, quantité, unité et prix unitaire en centimes.
Les totaux de ligne, HT, TVA et total provisoire sont recalculés côté serveur.

Le brouillon ne contient pas de champ `number`, `issuedAt`, `paidAt` ou de statut
« émis ». Il n'existe pas d'action d'émission ni d'envoi. Une date future de
prestation peut être envisagée dans un brouillon ; elle ne prouve pas que la
prestation a été réalisée ou qu'une émission à cette date serait valable.

## 4. Valider sans présumer le régime fiscal

Les permissions de création, lecture, modification et suppression exigent une
session. Le hook `beforeValidate` vérifie :

- des dates civiles réelles, à partir de l'année 2000 ;
- l'existence du client et de la mission éventuelle ;
- la cohérence entre client et mission, même lors d'une modification partielle ;
- la cohérence du traitement TVA et du taux.

Le fichier `src/domain/invoiceDraft.ts` contient les validations pures et les
libellés TVA. « À confirmer » est le choix initial : nous ne déduisons pas une
franchise d'un taux par défaut à zéro dans les devis.

Dans ce premier modèle, « Sans TVA » et « À confirmer » imposent un taux nul.
« TVA facturée » impose un seul taux positif, entier, jusqu'à 10 000 points de
base. Ce plafond est une borne technique, pas une liste de taux légalement permis.
Par exemple, 2 000 représente 20 %, mais le taux pertinent doit être vérifié.
Les taux multiples et autres cas fiscaux ne sont pas pris en charge.

La mention TVA reste facultative au stade du brouillon. Elle devra être vérifiée
avant l'émission. Choisir « Sans TVA » n'établit pas juridiquement le bénéfice
d'une franchise ou d'une autre exonération.

Les lectures liées utilisent `req` et `overrideAccess: false`. Les contrôles
portent sur les données fusionnées avec le document existant : changer uniquement
le client ou le taux ne permet pas de contourner la validation.

## 5. Réutiliser les calculs testés des devis

Le hook `beforeChange` appelle `calculateQuoteAmounts`, déjà utilisé et testé
au chapitre 6. Nous ne créons pas un second moteur de calcul monétaire.

```ts
const lines = data.lines ?? originalDoc?.lines ?? []
const amounts = calculateQuoteAmounts(
  lines,
  data.taxRateBps ?? originalDoc?.taxRateBps ?? 0,
)
```

Le calcul utilise des centimes entiers, des quantités limitées à deux décimales
et `BigInt` pour les opérations intermédiaires. Chaque ligne est arrondie, puis
la TVA unique est calculée sur le total HT. Les montants dépassant la capacité
d'un entier sûr sont refusés.

Le serveur écrase les totaux envoyés par le navigateur. `admin.readOnly` rend
l'interface plus claire, mais ne suffit pas à sécuriser un calcul reçu par API.
Une modification des notes recalcule aussi les montants à partir des lignes
existantes, sans dépendre d'un total fourni par le client.

## 6. Ajouter la vue et mettre à jour SQLite

Enregistrer la collection dans `src/payload.config.ts`, puis générer les types.
Ajouter `invoice-drafts` à `workspaceViews`. La liste affiche le client, le
total provisoire, le traitement TVA et la date de prestation envisagée. Les liens
ouvrent les formulaires natifs Payload de création et modification.

Pour une installation existante du chapitre 16, **serveur arrêté** :

```powershell
pnpm generate:types
pnpm backup:local
pnpm upgrade:chapter17
pnpm build
pnpm start
```

La mise à jour locale ajoute `invoice_drafts`, sa table de lignes et la relation
de verrouillage. Elle conserve les tables existantes, crée une copie SQLite
avant modification et peut être relancée. Ce script est réservé à la base locale
du fil rouge ; il n'est pas présenté comme une migration d'hébergement distant.

Les brouillons sont inclus dans les sauvegardes SQLite habituelles. Le code
complet est dans le [dépôt de l'application](https://github.com/Anakinyo/payload-archiviste).

## 7. Tester et démontrer

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm lint
pnpm build
```

`pnpm test` lance les trois familles. Les tests d'intégration et navigateur
utilisent exclusivement `payload-test.db`, avec des utilisateurs et clients fictifs.

Les nouveaux tests couvrent les dates impossibles, les combinaisons TVA invalides,
les permissions, le client d'une mission, les modifications partielles,
les montants falsifiés, la migration additive et l'interface sur écran étroit.
Ils vérifient également qu'aucun numéro officiel n'est attribué.

Pour une démonstration : créer un client fictif, préparer 1,5 jour à 35 000 centimes
par jour, puis vérifier le total provisoire de 52 500 centimes. Choisir explicitement
le traitement TVA adapté au scénario fictif. Une TVA de test à 2 000 points de
base porte le total à 63 000 centimes. Corriger le titre et enregistrer le brouillon.
Vérifier qu'aucun PDF, numéro officiel ou paiement n'a été créé.

## Suite et critères de passage en usage réel

La prochaine étape préparera une relecture figée de la facture et les contrôles
préalables. Avant une émission réelle, il faudra notamment confirmer l'identité
juridique, le régime fiscal applicable à l'opération, les mentions et conditions,
la numérotation de départ, les éventuelles factures déjà émises dans l'année et
le circuit de transmission.

Puis viendront l'émission manuelle, le PDF, le dépôt ou l'intégration adaptés,
les paiements partiels, les avoirs et les récapitulatifs d'encaissements.
Le cours reste un fil rouge technique, pas une certification de conformité.
