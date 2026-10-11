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
- Categorie principale et sous-categorie facultative.
- Support de conservation.

Les dates extremes restent exprimees en annees. Les libelles W et Cadre 1926
sont des choix de saisie : ils ne generent pas de cote reglementaire, ne
prechargent pas un cadre officiel et ne valident pas le classement a la place
de la personne qui realise la mission.

## Deux vocabulaires partages

Dans le groupe Classement de l'administration, creer les categories et les
supports de conservation. On peut aussi les creer depuis les boutons d'ajout
des champs de relation. Une categorie sans parent est une categorie principale.
Une categorie avec un parent est une sous-categorie. Ce premier jet limite
explicitement la hierarchie a deux niveaux, sans cycle possible.

La fiche propose les sous-categories du parent choisi. Le serveur verifie aussi
cette appartenance : filtrer une liste dans l'interface ne suffit pas a proteger
une API. En changeant de categorie sur une fiche, choisir une sous-categorie
compatible ou effacer l'ancienne selection.

Les doublons de noms (casse et espaces) sont refuses au meme niveau du meme
parent. Deux parents peuvent avoir des sous-categories portant le meme nom.
Un support ou une categorie deja utilise ne peut pas etre supprime. Le parent
d'une categorie utilisee, ou ayant des enfants, ne peut pas etre change.
Les noms et descriptions restent modifiables.

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
« Fiches de classement ». Choisir la mission dans le formulaire, renseigner
les champs et enregistrer. La page de mission affiche les fiches par ordre,
avec une pagination de 25 fiches. Chaque lien permet de rouvrir la fiche.

Les fiches restent modifiables et supprimables par les utilisateurs connectes.
Il ne s'agit pas encore d'un inventaire final fige : export, impression,
cotation plus precise et formulaire metier complet restent a definir.

## Installation et tests

Sur une installation existante, arreter le serveur puis executer :

```powershell
pnpm upgrade:classification
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

Les tests couvrent la hierarchie, les annees, les droits d'acces, les creations
simultanees, les numeros manuels et les doublons, les compteurs independants,
les suppressions protegees et la migration. Le test navigateur cree une fiche
complete et controle son affichage sur ordinateur et sur mobile.
