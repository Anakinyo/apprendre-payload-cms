# Avancement et périmètre de la première version locale

État au 8 octobre 2026, après le chapitre 22.

## Déjà implémenté

| Domaine | Livré | À valider en usage réel |
| --- | --- | --- |
| Socle | Payload, authentification, accès privés, SQLite locale, tests | Installation et démarrage sur l'ordinateur cible |
| Clients | Fiches, recherche, import CSV | Colonnes et qualité de l'export réel du tableur |
| Devis | Lignes, montants, personnalisation, aperçu PDF, validation interne, numéro final et préparation d'e-mail | Présentation attendue et configuration réelle des courriels |
| Missions | Relations clients, journées, temps, pièces, livrables, synthèse | Ergonomie avec quelques missions représentatives |
| Sauvegardes | Création et vérification locales | Organisation d'une copie hors du disque de travail et exercice de restauration |
| Facturation | Brouillons, relectures immuables, décisions, paramètres, mentions figées, bilan avant émission | Reprise des vrais numéros et qualification des mentions |
| Cours | 22 chapitres, commandes et tests expliqués | Relecture pédagogique par une personne extérieure |

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

1. **Émission et PDF de facture** : attribution cohérente et unique des numéros,
   recontrôle à l'émission, conservation du document, double clic/concurrence,
   échecs et récupération. La consultation ne doit jamais émettre une facture.
2. **Paiements et suivi** : encaissements partiels, solde, échéances et corrections
   tracées ; préciser le traitement des avoirs avant l'usage réel de la facturation.
3. **Recette et exploitation locale** : données représentatives, courriel configuré
   sans envoi de test aux clients, démarrage, sauvegarde et restauration, corrections.
4. **Clôture du cours** : parcours reproductible, liens, commandes et limites,
   vérification depuis une installation propre.

Un récapitulatif des encaissements par période peut compléter le lot paiements.
Il ne doit pas être présenté comme une déclaration URSSAF prête à déposer sans
qualification des règles et des données nécessaires.

## Estimation

Prévoir **4 à 6 lots de travail supplémentaires**, recette comprise, pour cette
version locale bornée. Le lot émission est le plus sensible et peut nécessiter
deux passages. Il ne s'agit pas de quatre à six messages ni d'un engagement de date.

À raison de deux à trois lots validés par semaine, cela représente approximativement
**deux à trois semaines**, à condition que les informations de reprise soient
disponibles et qu'une recette puisse être faite rapidement. Une recette différée
repousse la date de validation, même si le développement est terminé.

## Informations à obtenir avant bascule

- Le dernier numéro de facture réellement émis, son format et sa date.
- Les mentions et coordonnées de facturation confirmées par l'entreprise.
- Quelques exemples anonymisés : devis, facture, mission et export de clients.
- Le choix du circuit réel de transmission des factures aux destinataires.
- Un retour de l'utilisatrice sur deux ou trois parcours quotidiens.

Ne pas changer de logiciel de facturation sur la seule base du nombre de tests
réussis. La bascule se décide après vérification des données et du parcours complet.
