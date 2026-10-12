# Avancement et périmètre de la première version locale

État au 12 octobre 2026, après le chapitre 25 et les bonus métier.

## Déjà implémenté

| Domaine | Livré | À valider en usage réel |
| --- | --- | --- |
| Socle | Payload, authentification, accès privés, SQLite locale, tests | Installation et démarrage sur l'ordinateur cible |
| Clients | Fiches, recherche, import CSV | Colonnes et qualité de l'export réel du tableur |
| Devis | Lignes, montants, personnalisation, aperçu PDF, validation interne, numéro final et préparation d'e-mail | Présentation attendue et configuration réelle des courriels |
| Missions | Intitulés automatiques type/client/année, relations clients, dates de campagne, calendrier des journées, temps, pièces, livrables, synthèse et relevés d'élimination | Ergonomie avec quelques missions représentatives |
| Classement | Fiches par mission, compteur indépendant et ajustable, ordre, années extrêmes, catégories sur deux niveaux, supports, choix W ou Cadre 1926 | Premier formulaire à affiner ; cotation détaillée, impression et export non implémentés |
| Courriels | Transport SMTP commun et récupération du mot de passe testés localement | Branchement au fournisseur et réception réelle différés |
| Sauvegardes | Création et vérification locales | Organisation d'une copie hors du disque de travail et exercice de restauration |
| Facturation | Brouillons, relectures, décisions, émission numérotée et PDF immuable pour le parcours EI sans TVA en France ; avoirs partiels/totaux et solde ajusté | Reprise des vrais numéros, mentions, PDF et transmission ; corrections d'identité/TVA et refacturation liée hors périmètre |
| Paiements | Journal immuable, règlements partiels, solde par facture et annulations de saisie motivées | Exemples réels, remboursements et vue globale non couverts |
| Cours | 25 chapitres et bonus SMTP, relevés d'élimination, fiches de classement et intitulés automatiques ; commandes et tests expliqués | Relecture pédagogique par une personne extérieure |

« Implémenté » ne signifie pas encore « accepté par l'utilisatrice ». Les tests
automatiques n'évaluent ni les habitudes de travail, ni les données réelles,
ni l'ensemble des obligations propres à l'entreprise.

## Définition de la version locale terminée

Une personne peut gérer ses clients et missions, préparer et valider ses devis,
émettre les factures du périmètre retenu, enregistrer ses encaissements, retrouver
ses documents et sauvegarder/restaurer son travail sur son ordinateur portable.
Une recette représentative et la procédure de démarrage doivent être validées.

La première version n'inclut pas de synchronisation entre appareils, d'hébergement,
d'application mobile hors ligne, de connexion directe à Google Sheets, de dépôt
automatique Chorus Pro ou de déclaration URSSAF automatique. Le circuit réel de
transmission des factures reste à qualifier avant de remplacer un outil existant.

## Lots restants

Une [liste de recette à remplir](./RECETTE.md) est disponible. Sa préparation
ne signifie pas que les essais sur l'ordinateur cible ont été réalisés.

1. **Validation de la facturation** : confirmer les séries et tester les avoirs ;
   qualifier les besoins de remise avant facturation, de refacturation et de remboursement.
2. **Recette et exploitation locale** : données représentatives, courriel configuré
   sans envoi de test aux clients, démarrage, sauvegarde et restauration, corrections.
3. **Clôture du cours** : parcours reproductible, liens, commandes et limites,
   vérification depuis une installation propre.

Un récapitulatif des encaissements par période peut compléter le lot paiements.
Il ne doit pas être présenté comme une déclaration URSSAF prête à déposer sans
qualification des règles et des données nécessaires.

Le premier jet des fiches de classement doit maintenant être confronté à quelques
dossiers représentatifs. Les éventuels ajouts issus de cette recette constituent
un périmètre complémentaire aux lots de facturation et d'exploitation ci-dessus.

## Estimation

Prévoir encore **3 à 4 lots de travail**, recette comprise, pour cette version locale
bornée. Les paiements et avoirs du périmètre sont implémentés ; la validation en
usage réel et les éventuels besoins complémentaires restent importants. Ce n'est
pas un engagement de date.

À raison de deux à trois lots validés par semaine, cela représente approximativement
**deux à trois semaines**, à condition que les informations de reprise soient
disponibles et qu'une recette puisse être faite rapidement. Une recette différée
repousse la date de validation, même si le développement est terminé.

## Informations à obtenir avant bascule

- Le dernier numéro de facture réellement émis, son format et sa date.
- L'historique et le format retenu pour la série distincte d'avoirs.
- Les mentions et coordonnées de facturation confirmées par l'entreprise.
- Quelques exemples anonymisés : devis, facture, mission et export de clients.
- Le choix du circuit réel de transmission des factures aux destinataires.
- Un retour de l'utilisatrice sur deux ou trois parcours quotidiens.

Ne pas changer de logiciel de facturation sur la seule base du nombre de tests
réussis. La bascule se décide après vérification des données et du parcours complet.
