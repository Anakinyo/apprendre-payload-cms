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
- Categorie (niveau 1), obligatoire.
- Fond (niveau 2), obligatoire.
- Sous-fond (niveau 3), facultatif.
- Niveau 4, facultatif et rattache au sous-fond.
- Support de conservation.

Les dates extremes restent exprimees en annees. Les libelles W et Cadre 1926
sont des choix de saisie : ils ne generent pas de cote reglementaire, ne
prechargent pas un cadre officiel et ne valident pas le classement a la place
de la personne qui realise la mission.

## Deux vocabulaires partages

Dans le groupe Classement de l'administration, creer les categories et les
supports de conservation. On peut aussi les creer depuis les boutons d'ajout
des champs de relation. Une categorie sans parent est une categorie principale.
Une categorie avec un parent appartient au niveau suivant. La hierarchie est
limitee a quatre niveaux, sans cycle possible. Le niveau est calcule par le
serveur et ne se saisit pas manuellement.

Exemple : Secretariat general > Affaires generales > Police administrative >
Recensement. On peut arreter une fiche au fond ou au sous-fond. On ne peut pas
saisir un niveau 4 sans sous-fond. Les termes restent libres : l'application
ne precharge pas ces exemples dans les donnees de l'entreprise.

La fiche propose uniquement les enfants du parent choisi. Le serveur verifie aussi
cette appartenance : filtrer une liste dans l'interface ne suffit pas a proteger
une API. En changeant un parent sur une fiche, choisir des descendants compatibles
ou effacer les selections facultatives. Le fond reste obligatoire.

Les doublons de noms (casse et espaces) sont refuses au meme niveau du meme
parent. Deux parents peuvent avoir des sous-categories portant le meme nom.
Un support ou une categorie deja utilise ne peut pas etre supprime. Le parent
d'une categorie utilisee, ou ayant des enfants, ne peut pas etre change.
Les noms et descriptions restent modifiables.

## Contribuer une liste et choisir un referentiel

Dans **Classement > Categories de classement > Create**, saisir un nom et
laisser le parent vide pour le niveau 1. Creer ensuite le fond en choisissant
ce premier terme comme parent, puis le sous-fond avec le fond comme parent,
et enfin le niveau 4 avec le sous-fond comme parent. Le niveau est automatique.
On peut renommer un terme ; deplacer un terme deja utilise ou possedant des
enfants est bloque pour ne pas modifier silencieusement les fiches existantes.

Il existe des cadres communs, notamment le cadre de 1926 pour les archives
communales, mais il ne faut pas les confondre avec une liste universelle de
rubriques metier a quatre niveaux. Les exemples du formulaire ne sont pas
presentes comme un referentiel officiel. Valider le plan retenu avec la personne
qui classe les archives avant de precharger une liste complete.

Sources : [Association des archivistes francais, cotation des fonds](https://www.archivistes.org/ressources/fiche-pratique-3-cotation-des-fonds/)
et [Archives de Toulouse, organisation des fonds et adaptations locales](https://archives.toulouse.fr/organisation-des-fonds/).

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

Les fiches restent modifiables et supprimables par les utilisateurs connectes.
Il ne s'agit pas encore d'un inventaire final fige : export, impression,
cotation plus precise et formulaire metier complet restent a definir.

## Installation et tests

Sur une installation existante, arreter le serveur puis executer :

```powershell
pnpm upgrade:classification
pnpm upgrade:classification-hierarchy
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

La seconde migration ajoute les relations des niveaux 3 et 4 et calcule les
niveaux des categories existantes. Les anciens champs techniques `category`
et `subcategory` sont conserves : `subcategory` porte desormais le libelle Fond.
Les nouveaux champs sont `subfonds` et `classificationDetail`. Les selections
existantes, descriptions, numeros et compteurs sont conserves.

Une ancienne fiche sans sous-categorie reste lisible et signalee « Fond a
completer » dans la mission. Le fond sera exige lors de sa prochaine sauvegarde.
La migration affiche le nombre de ces fiches, sans leur attribuer un fond fictif.
La colonne historique reste nullable en SQLite pour conserver ces anciennes
fiches ; les validations Payload imposent la nouvelle regle aux ecritures.

Les tests couvrent la hierarchie, les annees, les droits d'acces, les creations
simultanees, les numeros manuels et les doublons, les compteurs independants,
les suppressions protegees et la migration. Le test navigateur cree une fiche
complete a quatre niveaux et controle son affichage sur ordinateur et sur mobile.
Les tests refusent aussi les niveaux sautes, les mauvaises branches, les cycles
et un cinquieme niveau ; ils verifient que les niveaux facultatifs peuvent etre
effaces ensemble sans changer le numero de la fiche.
