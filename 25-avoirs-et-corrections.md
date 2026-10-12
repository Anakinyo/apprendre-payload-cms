# 25. Émettre un avoir sans réécrire la facture

## Objectif et limites

Ce chapitre ajoute des avoirs partiels ou totaux pour les factures déjà émises
dans l'application : prestations en France, fournisseur EI, franchise de TVA.
Il ne gère pas les factures provenant d'un autre logiciel, les corrections de
TVA, les changements d'identité, les remboursements bancaires ni les transferts
de crédit entre factures. Aucun document n'est envoyé automatiquement.

Un avoir réduit une facture ; ce n'est ni une facture modifiée ni un encaissement
négatif. Les notes d'avoir doivent notamment identifier la facture initiale.
Voir les [règles de facturation de Service Public](https://entreprendre.service-public.gouv.fr/vosdroits/F23208?lang=fr)
et le [BOFiP sur les documents rectificatifs](https://bofip.impots.gouv.fr/bofip/142-PGP.html/identifiant%3DBOI-TVA-DECLA-30-20-20-20-20220119),
consultés le 12 octobre 2026. Ces sources encadrent le choix technique ; le cours
ne constitue pas une certification de conformité du logiciel.

## 1. Choisir le bon parcours

- Avant émission : corriger le brouillon puis refaire la relecture si nécessaire.
- Après émission : conserver la facture, saisir un motif et le montant de l'avoir.
- Erreur de saisie d'un paiement : utiliser l'annulation de saisie du chapitre 24.
- Argent réellement remboursé : ne pas l'enregistrer comme une annulation de saisie.
  Le journal de remboursements n'est pas encore implémenté.

Une remise de 5 % convenue avant facturation doit être présentée dans la facture ;
une remise accordée après émission peut donner lieu à un avoir. Par exemple,
sur une base de 1 000 EUR, 5 % représentent 50 EUR, pour un net de 950 EUR.
Les [réductions de prix acquises et chiffrables](https://www.economie.gouv.fr/entreprises/gerer-son-entreprise-au-quotidien/gerer-sa-comptabilite-et-ses-demarches/mentions-obligatoires-dune-facture-tout-savoir)
font partie des mentions de facturation. Le champ de remise avant émission n'est
pas ajouté par ce chapitre : ne pas fabriquer un avoir sur une facture non émise.

Pour les clients publics, conserver l'accord et vérifier la cohérence avec les
documents contractuels et le circuit de transmission retenu.

## 2. Une collection immuable

`src/collections/CreditNotes.ts` définit `credit-notes` : facture initiale,
montant en centimes, motif, numéro, date, copie des données, PDF et empreinte,
auteur et identifiant de demande.

Le serveur refuse toute modification ou suppression, y compris via un appel
Local API privilégié. L'auteur, le numéro, le PDF et les données copiées sont
calculés côté serveur. Le PDF n'est accessible que par un téléchargement
authentifié qui vérifie son empreinte.

## 3. Configurer une série d'avoirs

Dans **Facturation > Numérotation des avoirs**, confirmer soit le tout premier
avoir de l'entreprise, soit la reprise d'une série existante. Renseigner le format,
le dernier compteur, la référence et la date si des avoirs existent déjà.

Exemple fictif : préfixe `AV-DEMO-`, quatre chiffres, premier compteur zéro.
Le premier numéro proposé sera `AV-DEMO-0001`. Ce n'est pas un format imposé.
Le préfixe doit distinguer la série de celle des factures. Les collisions entre
numéros de factures et d'avoirs sont refusées au moment de l'émission.
La migration laisse l'historique à confirmer, sans numéro consommé.

`CreditNoteSettings` réutilise les champs et validations de numérotation existants
avec les libellés propres aux avoirs. Après une première émission, les paramètres
sont verrouillés : seul le compteur transactionnel avance. Le changement d'année
ne remet pas automatiquement le compteur à zéro.

## 4. Préparer puis confirmer

Depuis la facture conservée, ouvrir **Suivre les paiements**, puis la section
**Avoirs et corrections**. Saisir le montant positif et un motif précis.

`POST /api/credit-notes/preview` retourne un PDF provisoire et une empreinte
de relecture sans enregistrer d'avoir ni consommer de numéro. Relire le PDF,
cocher la confirmation puis émettre. Modifier le montant ou le motif invalide
l'aperçu dans le formulaire.

À l'émission, le serveur refait les contrôles. L'empreinte tient compte du numéro
proposé, du jour, du contenu, des avoirs précédents et des encaissements. Si ces
éléments ont changé, une nouvelle relecture est nécessaire. Le PDF définitif
ne porte plus la mention provisoire ; son contenu métier correspond à la relecture.

L'ensemble « compteur + PDF + avoir » est transactionnel. Une erreur PDF annule
aussi l'avancement du compteur. L'identifiant unique de demande et la vérification
de relecture évitent de créer deux avoirs par double clic. Après une réponse réseau
incertaine, vérifier la liste conservée avant de recommencer.

## 5. Calculer le nouveau solde

`paymentBalance` conserve la somme des paiements réels et calcule :

```text
net après avoirs = montant initial - total des avoirs
reste à payer = max(0, net après avoirs - encaissements)
crédit client = max(0, encaissements - net après avoirs)
```

Exemple : facture de 350 EUR, encaissement de 200 EUR, avoir de 200 EUR.
Le net est 150 EUR, le reste à payer est nul, le crédit client est 50 EUR.
L'application affiche ce crédit à régulariser, sans inventer un remboursement.

Le cumul des avoirs ne peut pas dépasser le total de la facture. Un nouvel
encaissement ne peut pas dépasser le solde après avoirs. L'original, son numéro
et ses octets PDF restent inchangés.

## 6. Mettre à jour la base locale

Arrêter le serveur de l'application, puis exécuter dans son dossier :

```powershell
pnpm backup:local
pnpm upgrade:chapter25
pnpm build
pnpm start
```

La migration crée les tables d'avoirs et de paramètres ainsi que leurs index,
avec une sauvegarde SQLite préalable. Elle est additive et réexécutable.
Elle ne modifie ni les factures ni les paiements existants.

## 7. Tester

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
```

- Unitaire : soldes, crédits clients, limites et génération PDF avec motif long.
- Intégration : confirmation, relecture périmée, numérotation, concurrence,
  panne PDF, immutabilité, plafond et effet sur les paiements.
- Migration : deuxième exécution sans perte des factures ni remise à zéro.
- Navigateur : aperçu, confirmation, émission, solde, téléchargement protégé,
  persistance après rechargement et présentation mobile.

Les tests utilisent exclusivement la base de démonstration. Pour une correction
nécessitant une nouvelle facture après neutralisation, le parcours lié et le
transfert éventuel de paiements ne sont pas automatisés dans cette version.
Ne pas utiliser le module comme une correction universelle des factures.

Suite : recette et exploitation de la version locale, avec les limites ci-dessus
et la numérotation réelle validées avant la bascule.
