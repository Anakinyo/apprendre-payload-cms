# Apprendre Payload CMS en français

Ce dépôt propose un parcours pratique pour découvrir Payload CMS en construisant
progressivement une application métier pour une petite entreprise.

Le fil rouge part d'un projet vide et introduit les concepts au moment où ils
deviennent utiles : collections, champs, relations, permissions, hooks, API,
génération de documents, tests et mise en production.

## Public visé

Le cours s'adresse aux développeurs et développeuses qui connaissent les bases
de JavaScript ou TypeScript et souhaitent découvrir Payload CMS avec un exemple
concret. Une première expérience de React ou Next.js est utile, mais elle n'est
pas indispensable pour suivre les premiers épisodes.

## Prérequis

- Windows, macOS ou Linux ;
- une version LTS récente de Node.js ;
- un terminal ;
- un éditeur comme Visual Studio Code ;
- quelques notions de JavaScript ou TypeScript.

## Episodes disponibles

1. [Installer un projet Payload](./01-installation-payload.md)
2. [Créer une collection Clients](./02-creer-collection-clients.md)
3. [Relier des missions aux clients](./03-relier-missions-clients.md)
4. [Configurer les informations de l'entreprise](./04-configurer-entreprise-global.md)
5. [Construire un devis et ses lignes](./05-construire-devis.md)
6. [Calculer et valider les montants](./06-calculer-montants.md)

## Parcours prévu

7. Générer un aperçu PDF
8. Mettre en place une validation interne
9. Envoyer un document après validation humaine
10. Préparer PostgreSQL, les sauvegardes et le déploiement

La facturation et le suivi des paiements formeront une seconde partie, après la
stabilisation du parcours consacré aux devis.

## Philosophie du cours

Chaque épisode suit la même méthode :

1. partir d'un besoin métier compréhensible ;
2. modéliser uniquement les données nécessaires ;
3. implémenter le comportement avec les mécanismes natifs de Payload ;
4. vérifier manuellement l'interface ;
5. ajouter des contrôles automatiques ;
6. expliquer les choix et leurs limites.

Le code présenté privilégie la lisibilité et l'apprentissage. Les aspects plus
avancés, comme les rôles détaillés, les migrations de production ou les services
externes, sont introduits lorsque le projet en a réellement besoin.

## Etat du projet

Ce cours est en cours d'écriture. Les exemples et commandes sont testés au fur
et à mesure sur l'application du fil rouge.
