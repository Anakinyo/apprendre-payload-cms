# Recette de la première version locale

Cette liste prépare les essais : les cases ne sont pas cochées par les tests
automatiques. La recette doit être effectuée sur une installation de démonstration
séparée, avec des données fictives, sans envoi aux clients ni émission de vraies
factures. Ne pas utiliser les vrais compteurs pour apprendre le parcours.

## Installation et accès

- [ ] Refaire les instructions d'installation sur un dossier neuf.
- [ ] Démarrer l'application, se connecter, fermer puis redémarrer le serveur.
- [ ] Vérifier que les données sont conservées et les pages privées protégées.
- [ ] Essayer sans connexion Internet, serveur local démarré ; l'e-mail est exclu.

## Parcours quotidien

- [ ] Créer un client puis le retrouver par nom, contact et adresse.
- [ ] Importer un petit CSV fictif ; vérifier les erreurs et le deuxième import.
- [ ] Créer une mission et vérifier son intitulé type/client/année.
- [ ] Changer l'année puis essayer un intitulé personnalisé.
- [ ] Ajouter une journée avec le calendrier et vérifier le temps total.
- [ ] Joindre puis télécharger une pièce de mission.
- [ ] Créer une branche catégorie > fond > sous-fond > niveau 4 et un support de conservation.
- [ ] Créer une fiche avec seulement catégorie et fond, puis une fiche à quatre niveaux.
- [ ] Enchaîner deux fiches : valeurs reprises, numéro +1 et deux années obligatoires remises à vide.
- [ ] Vérifier qu'une erreur conserve les champs et qu'un second clic sans années ne crée rien.
- [ ] Rechercher une mission par client et par type ; ouvrir son détail et vérifier son compteur de fiches.
- [ ] Vérifier le refus d'un fond manquant, d'une mauvaise branche et d'un niveau sauté.
- [ ] Créer deux fiches de classement ; vérifier numéros et années extrêmes.
- [ ] Créer une deuxième mission : sa première fiche doit porter le numéro 1.
- [ ] Modifier le prochain numéro et vérifier le refus d'un doublon.

## Documents et règlements, en démonstration uniquement

- [ ] Préparer un devis, vérifier son PDF et le faire relire.
- [ ] Corriger un brouillon sans modifier un document déjà figé.
- [ ] Préparer une facture et vérifier les paramètres et le numéro proposé.
- [ ] Enregistrer deux paiements partiels ; vérifier le solde restant.
- [ ] Annuler une saisie erronée avec un motif ; vérifier l'historique.
- [ ] Configurer une série d'avoirs fictive, prévisualiser et confirmer un avoir partiel.
- [ ] Vérifier le nouveau solde, le PDF initial inchangé et le téléchargement de l'avoir.
- [ ] Tester un avoir total après paiement : un crédit client apparaît, sans remboursement fictif.
- [ ] Distinguer annulation de saisie, avoir et remboursement réellement effectué.

## Sauvegarde et récupération

- [ ] Exécuter `pnpm backup:local` depuis le dossier de l'application.
- [ ] Vérifier le dossier produit avec `pnpm backup:verify "CHEMIN_DU_DOSSIER"`.
- [ ] Copier la sauvegarde sur un autre support et vérifier cette copie.
- [ ] Suivre la procédure de restauration du cours dans un dossier séparé,
      serveur arrêté ; ne jamais écraser le dossier utilisé au quotidien.
- [ ] Vérifier données, pièces et PDF après restauration, pas seulement le démarrage.
- [ ] Conserver les secrets de configuration séparément et en sécurité.

## Décision de mise en service

Noter pour chaque problème : écran, étapes suivies, résultat attendu et résultat
observé. Éviter les captures contenant des données personnelles dans un dépôt public.

Les courriels réels, les avoirs, la reprise des numéros et le circuit de transmission
doivent être validés avant de remplacer le logiciel de facturation utilisé.
La recette des missions peut être acceptée indépendamment de cette bascule.
