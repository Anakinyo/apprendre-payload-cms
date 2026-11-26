# Bonus : des fiches de classement numerotees par mission

Ce premier inventaire prolonge le suivi des missions. Chaque dossier a sa propre
fiche, rattachee directement a une mission de classement ou de classement et
elimination, independamment des journees de travail.

## Les champs

- Mission : une mission compatible deja creee.
- Numero de fiche : automatique si laisse vide a la creation, modifiable ensuite.
- Numero d'ordre : independant du numero de fiche ; initialise avec celui-ci si vide.
- Type de classement : W ou Cadre 1926.
- Description du contenu du dossier.
- Annee extreme de debut et annee extreme de fin.
- Série et sous-série, toutes deux obligatoires.
- Support de conservation.

Les dates extremes restent exprimees en annees. Les libelles W et Cadre 1926
sont des choix de saisie : ils ne generent pas de cote reglementaire, ne
prechargent pas un cadre officiel et ne valident pas le classement a la place
de la personne qui realise la mission.

## Préparer les séries et sous-séries

Depuis l’espace de travail, ouvrir **Administration**, puis **Classement → Séries et sous-séries**. Choisir W ou Cadre 1926, saisir le nom du terme et laisser la série parente vide pour créer une série. Pour créer une sous-série, choisir sa série parente dans le même plan.

Par exemple, on peut créer « Affaires générales » puis « Police administrative » dans le plan Cadre 1926. Les termes sont à renseigner selon le plan utilisé pour la mission.

Dans une fiche, le type de classement détermine les séries proposées. La série détermine les sous-séries proposées. Le serveur vérifie aussi cette cohérence lorsque la fiche est créée via l’API. Un même nom peut exister dans les deux plans.

En création comme en modification d’un terme, **Enregistrer** revient à la liste. **Enregistrer et créer un nouvel élément** sauvegarde la saisie puis ouvre un formulaire vide. Les supports de conservation et les types de documents proposent les mêmes actions grâce au composant Payload `VocabularySaveButtons`.

La liste affiche séparément les plans W et Cadre 1926. Cliquer sur le nom d’un terme pour le modifier. Les sous-séries sont en retrait sous leur série parente. **Vue tableau** ouvre la liste habituelle de Payload, avec ses filtres et actions de gestion. **Vue hiérarchique** revient à l’arborescence.

## Un compteur propre a chaque mission

La mission porte le champ `nextClassificationNumber`, affiche comme
« Prochain numero de fiche », initialise a 1. Pour reprendre un inventaire a 40,
modifier ce champ dans la mission puis creer une fiche sans saisir son numero.

Le hook de la collection `classification-records` lit ce compteur, attribue le
numero, puis avance le compteur dans la meme transaction SQLite. Si la creation
echoue, l'ensemble de la transaction est annule. L'adaptateur SQLite existant
serialise les transactions d'ecriture dans ce processus.

Une cle unique `mission:numero` protege aussi la base. Deux missions peuvent
chacune avoir une fiche 1 ; deux fiches de la meme mission ne le peuvent pas.
Une saisie manuelle de numero plus eleve avance le compteur. Une suppression
ne le fait pas reculer. Une reprise manuelle du compteur sur un numero deja
utilise produit une erreur et ne remplace jamais la fiche existante.

Le numero d'ordre sert au tri, puis le numero de fiche departage les egalites.
Modifier le contenu ou l'ordre ne renumerote pas la fiche. Une fiche existante
ne peut pas etre deplacee vers une autre mission dans ce premier jet.

## Parcours utilisateur

Ouvrir une mission de classement, puis « Nouvelle fiche » dans la section
« Fiches de classement ». La mission est deja choisie dans l'ecran de saisie
en chaine. Renseigner les champs puis cliquer sur « Enregistrer et continuer ».
La page de mission affiche les fiches par ordre,
avec une pagination de 25 fiches. Chaque lien permet de rouvrir la fiche.

### Enchainer les dossiers

Apres confirmation du serveur, le formulaire prepare la fiche suivante :
numero enregistre + 1, memes selections, description et numero d'ordre,
mais annees extremes de debut et de fin vides et obligatoires. Le curseur
revient sur l'annee de debut. Aucune nouvelle fiche n'est enregistree sans
une nouvelle saisie des deux annees et une action sur le bouton.

Une erreur conserve tous les champs. Le bouton est bloque pendant l'envoi ;
le serveur refuse un numero deja utilise. Si la connexion est interrompue
apres l'enregistrement, verifier la mission avant de choisir un autre numero.
Les valeurs reprises restent seulement en memoire dans cet ecran : recharger
la page ne restaure pas un brouillon. Un avertissement protege une saisie
modifiee lors de la fermeture ou du retour vers la mission.

Le formulaire classique de l'administration reste disponible pour creer une
fiche ponctuelle et corriger une fiche existante. La saisie en chaine reutilise
la meme API et les memes validations, sans modifier la base de donnees.

Dans l’administration, **Enregistrer** sauvegarde la fiche et revient à la liste des fiches. **Enregistrer et créer une nouvelle fiche** reprend la mission, le type de classement, la série et la sous-série après la sauvegarde. Le numéro de fiche et le numéro d’ordre augmentent chacun de 1 à partir des valeurs enregistrées. La description, les deux années et le support restent à saisir.

Dans le formulaire de l’administration, le sélecteur **Mission** permet de chercher par nom de client. Les choix affichent le client en premier, suivi de la date et de l’intitulé de la mission. Par exemple, « Mairie du Moulin · 12/03/2026 · Classement des délibérations ». Le nom affiché se met à jour si le client est renommé.

### Construire cette saisie dans le fil rouge

1. La page serveur `src/app/(frontend)/missions/[id]/classement/page.tsx`
   verifie la connexion et le type de mission, puis charge les vocabulaires
   autorises et le prochain numero.
2. Le composant React `src/components/ClassificationEntryForm.tsx` conserve
   les champs dans un etat local. Changer un parent vide les niveaux enfants,
   pour eviter de conserver une selection appartenant a une autre branche.
3. Le bouton envoie une requete POST a `/api/classification-records`.
   Les validations de la collection restent la reference, y compris lorsqu'un
   utilisateur contourne les controles du navigateur.
4. Uniquement apres une reponse reussie, copier l'etat, remplacer le numero
   par celui retourne + 1 et vider les deux annees. Une erreur laisse l'etat intact.

Pour verifier ce parcours seul, executer `pnpm test:e2e --grep "chains records"`.
Le scenario cree deux fiches dans la base de test, controle les annees vides,
la reprise des autres valeurs et simule une erreur avant de reessayer.
Il controle ensuite la recherche de mission et son compteur de fiches.

Dans le tableau de bord, l'onglet Missions recherche le client, l'intitule,
le contact, le lieu ou le libelle du type de mission. Un filtre par type peut
etre combine avec la recherche. Le tableau affiche le client, le type,
la date de creation et, pour le classement, le nombre reel de fiches.
Ce nombre n'est pas deduit du prochain numero : une mission commencant a 40
avec deux fiches en affiche bien deux, pas quarante et une.

## Imprimer les etiquettes des boites

Dans la mission, la zone « Fiches de classement » propose « Imprimer les
etiquettes ». Elle produit un PDF calibre pour le format **Avery L7165** :
une feuille A4 de huit etiquettes, en deux colonnes et quatre lignes
(99,1 x 67,7 mm). Ce format est aussi disponible dans les modeles Word et
chez plusieurs fabricants compatibles.

Laisser « De la fiche » et « A la fiche » vides imprime toutes les fiches de
la mission. Pour reimprimer une partie, saisir les deux bornes incluses, par
exemple 103 et 144. Le PDF ne contient alors que les fiches dont le numero est
compris entre 103 et 144, avec huit emplacements par feuille.

Ouvrir le PDF et imprimer en A4, a taille reelle (100 %). Faire un premier essai
sur papier ordinaire et le superposer a une feuille d'etiquettes a contre-jour.
Pour une autre planche A4 de deux colonnes et quatre lignes, ouvrir
« Dimensions de la planche ». Saisir la largeur et la hauteur des etiquettes,
les marges gauche et haute, puis les ecarts entre les etiquettes, en millimetres.
Les descriptions longues sont abregees sur l'etiquette.

Le formulaire transmet la plage et le gabarit au serveur. Celui-ci valide que
la grille tient sur la feuille, puis lit les fiches de la mission dans l'ordre
de leurs numeros. Le generateur convertit les millimetres en points PDF et
place chaque etiquette a sa position sur une page A4.

Les fiches restent modifiables et supprimables par les utilisateurs connectes.
L'export de l'inventaire complet et la cotation detaillee restent a definir.

## Installation et tests

Sur une installation existante, arreter le serveur puis executer :

```powershell
pnpm upgrade:classification
pnpm upgrade:classification-hierarchy
pnpm upgrade:classification-schemes
pnpm upgrade:mission-labels
pnpm generate:types
pnpm generate:importmap
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm build
pnpm start
```

La reprise cree une sauvegarde SQLite avant d'ajouter les tables. Elle peut
etre relancee sans remettre a 1 les compteurs deja modifies. Sur une installation
neuve, Payload cree directement le schema.

La migration de la hiérarchie met à niveau les anciennes installations. Celle des plans sauvegarde la base, rattache les termes existants à W et copie dans Cadre 1926 les séries et sous-séries déjà utilisées par des fiches de ce plan. Les anciens niveaux 3 et 4 restent consultables sur les fiches concernées.

Les tests vérifient les deux plans, le refus d’un troisième niveau, la conservation des données pendant la migration et les deux destinations après la création d’un terme. Pour les parcours navigateur, exécuter `pnpm test:e2e --grep "classification record|chains records"`.
