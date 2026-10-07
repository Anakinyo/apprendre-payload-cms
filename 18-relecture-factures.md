# Chapitre 18 - Conserver une relecture de facture et ses contrôles

## Objectif

Une entreprise doit pouvoir relire précisément les informations envisagées pour
une facture, corriger les sources et retrouver la version qu'elle avait consultée.
Un brouillon modifiable ne suffit pas : un changement de client ou de paramètres
peut modifier ce que l'écran affiche.

Nous ajoutons une **relecture conservée**, distincte du brouillon et d'une facture
émise. Cette étape ne produit ni approbation, numéro officiel, PDF de facture,
paiement ou transmission. L'outil de facturation existant reste nécessaire.

## 1. Définir ce que les contrôles peuvent affirmer

Les mentions d'une facture dépendent de la situation et des opérations : identité
juridique, dates, prestations, TVA, règlement et numérotation, notamment. Les
mentions EI doivent correspondre à l'identité réelle de l'entrepreneur.
[Source officielle consultée le 7 octobre 2026 : Service Public](https://entreprendre.service-public.gouv.fr/vosdroits/F31808?profil=entrepreneur-individuel).

Nous séparons deux listes :

- **Informations à compléter** : données absentes ou incohérences détectables ;
- **Points à vérifier manuellement** : exactitude et règles applicables à l'opération.

Une liste automatique vide ne signifie jamais « facture conforme » ou « prête à
émettre ». Le logiciel ne vérifie pas une identité dans un annuaire et ne connaît
pas encore toutes les mentions, options fiscales, obligations et références
requises pour chaque destinataire.

La relecture couvre les prestations ordinaires du premier modèle, pas les acomptes,
avoirs, multiples taux ou opérations fiscales particulières.

## 2. Construire des données figées, sans notes internes

`src/domain/invoiceReview.ts` contient une fonction pure `buildInvoiceReview`.
Elle reçoit le brouillon, le client, les paramètres de l'entreprise et un jour
civil de référence. Le jour est fourni explicitement pour rendre les tests
reproductibles ; en fonctionnement, le serveur utilise la date de Paris.

Les données retenues sont :

- identités, SIRET et adresses du fournisseur et du client ;
- intitulé, dates envisagées et référence de commande ;
- traitement, taux et mention TVA ;
- descriptions, unités, quantités et prix ;
- totaux recalculés, version du format et date de relecture.

Les notes internes, contacts, journées et autres données inutiles ne sont pas
copiés. Chaque objet est reconstruit avec des champs explicites, plutôt que de
copier toute une fiche par un `spread`. Les prix sont recalculés avec le moteur
déjà testé des devis, sans faire confiance aux totaux stockés ou envoyés.

Les contrôles automatiques signalent notamment un SIRET fournisseur absent ou
mal formé, une adresse incomplète, une TVA à confirmer, une mention manquante en
l'absence de TVA, une échéance absente ou une prestation future dans ce parcours.
Un destinataire public doit aussi avoir un SIRET renseigné dans ce modèle.
La vérification de format des 14 chiffres ne prouve ni l'existence ni l'identité
du fournisseur ou du destinataire : cela reste un contrôle humain.

Les six points humains couvrent l'identité juridique, la prestation, la TVA,
les conditions de règlement, le circuit de transmission et la continuité de
la future numérotation. Ils ne sont pas des cases d'approbation enregistrées.

## 3. Ajouter une collection immuable

`src/collections/InvoiceReviews.ts` définit `invoice-reviews` avec les champs :

| Champ | Usage |
| --- | --- |
| `draft` | Relation obligatoire vers le brouillon |
| `reference` | Référence interne unique `RF-...` |
| `digest` | Empreinte SHA-256 du contenu de la relecture |
| `review` | Données, informations à compléter et points humains figés |

Création et lecture exigent une session. Modification et suppression sont
interdites par les permissions. Les hooks refusent aussi la modification et la
suppression lorsqu'un appel Local API contourne les permissions.

Lors d'une création, le serveur relit les sources avec `req` et
`overrideAccess: false`, construit le contenu, puis génère sa référence et son
empreinte. Toute référence, empreinte ou liste de contrôles envoyée par le client
est remplacée par les valeurs calculées.

```ts
const review = buildInvoiceReview(draft, client, company, parisDay())
const digest = createHash('sha256')
  .update(JSON.stringify(review))
  .digest('hex')
```

La référence interne n'est pas un numéro de facture. L'empreinte identifie un
contenu ; elle n'est ni une signature, une approbation humaine, un horodatage
certifié ou une preuve d'archivage probant.

Une relecture avec des données manquantes peut être conservée : elle trace l'état
consulté, sans autoriser une émission. Deux enregistrements identiques peuvent
avoir des références différentes et la même empreinte le même jour. Ce n'est
pas une duplication de facture, puisqu'aucune facture n'est émise.

## 4. Protéger le lien au brouillon

Le brouillon reste modifiable. Après une correction, créer une nouvelle relecture
pour conserver les nouvelles informations. Une ancienne relecture ne se recalcule
pas à partir du client ou des paramètres courants.

Un hook `beforeDelete` du brouillon recherche ses relectures et refuse sa
suppression si une version est conservée. La consultation historique n'est donc
pas privée de sa relation source par une suppression ordinaire.

L'immuabilité est une règle de l'application, pas une protection contre une
personne disposant directement de SQLite ou des sauvegardes. L'accès au poste
et aux copies doit rester protégé.

## 5. Construire les deux écrans

La liste des brouillons ouvre désormais :
`src/app/(frontend)/factures/[id]/relecture/page.tsx`.

Cet écran affiche les contrôles actuels et les données envisagées. Il offre les
liens vers le brouillon, les paramètres de l'entreprise et le client, ainsi
que le bouton « Figer une relecture ».

Le composant client `InvoiceReviewButton` appelle l'API native :

```ts
fetch(`${api}/invoice-reviews`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ draft: id }),
})
```

Après succès, le navigateur ouvre la version enregistrée à
`/factures/relectures/[id]`. Cet écran lit seulement le contenu conservé ; il
ne relit pas les paramètres ni le client pour reconstruire la relecture.
Un lien permet de revenir au brouillon et à ses contrôles actuels.

Les deux pages vérifient l'identifiant et la session avant toute lecture métier.
Les appels utilisent `overrideAccess: false`. Les 20 versions récentes sont
listées sur la préparation ; toutes les versions restent consultables dans
la liste native Payload. Les références longues et les empreintes peuvent revenir
à la ligne ; les tableaux restent dans une zone à défilement horizontal.

## 6. Mettre à jour le projet local

Partir du chapitre 17. Serveur arrêté :

```powershell
pnpm generate:types
pnpm backup:local
pnpm upgrade:chapter18
pnpm build
pnpm start
```

La mise à jour ajoute `invoice_reviews`, ses index et la relation de verrouillage,
avec une copie SQLite préalable et sans remplacer les tables existantes.
Elle peut être relancée. Le script reste limité à la base locale du fil rouge,
pas à une installation distante.

Le code complet est dans le
[dépôt de l'application](https://github.com/Anakinyo/payload-archiviste).

## 7. Tester et refaire le parcours

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm lint
pnpm build
```

`pnpm test` lance les trois familles. Les tests d'intégration et navigateur
utilisent exclusivement la base jetable `payload-test.db`.

Les nouveaux tests couvrent le recalcul des montants, l'exclusion des notes,
les contrôles automatiques, le maintien des vérifications humaines, les
permissions et la résistance aux données figées falsifiées. Ils modifient les
sources puis vérifient que la première relecture ne change pas. Ils tentent aussi
de modifier ou supprimer cette relecture avec et sans contournement des permissions.

Pour nettoyer leurs versions immuables, les tests utilisent directement
l'adaptateur de base, avec une garde imposant `payload-test.db`. Ce contournement
de nettoyage n'est pas exposé par une route de l'application et ne doit jamais
être utilisé pour effacer des historiques réels.

Pour une démonstration : ouvrir un brouillon fictif, observer les informations
manquantes, figer une relecture puis corriger le titre du brouillon. Revenir sur
la version conservée : l'ancien titre doit rester affiché. Créer une nouvelle
relecture : elle doit présenter le titre corrigé et une autre référence.
Sans session, les pages et l'API doivent refuser l'accès aux informations.

## Suite

La prochaine étape préparera une décision humaine portant sur une version précise,
puis l'émission contrôlée. Il faudra auparavant confirmer les mentions propres
à l'entreprise, le circuit de transmission et la numérotation déjà utilisée.
Ce chapitre ne remplace toujours pas le système de facturation en production.
