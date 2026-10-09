# Bonus : dater les missions et preparer un releve d'elimination

Ce complement prolonge les chapitres 3 et 14. Il ne remplace pas la future
definition du formulaire metier complet.

## Besoin

Une entreprise peut revenir chez le meme client pour une mission similaire
plusieurs annees apres la precedente. Une date de debut distingue les campagnes.
Pendant chaque journee, elle releve des types de documents et leurs annees
extremes : pour des documents de 2010, 2016 et 2020, le recapitulatif affiche
2010 a 2020, sans creer trois rubriques.

## Un calendrier sans changer les anciennes dates

Le champ `day` reste une chaine `AAAA-MM-JJ` dans la base. Le composant client
`CalendarDayField` utilise `useField` de Payload et un champ HTML `type="date"`.
Le navigateur fournit le calendrier, y compris sur mobile. Aucun passage par
un horodatage UTC ne risque de deplacer la journee. Le serveur continue de
verifier que la journee est reelle, non future et unique pour cette mission.

Le meme composant sert au champ facultatif `startDay` de la mission. Le libelle
`displayLabel`, recalcule cote serveur, associe le titre a la date. Payload
l'utilise dans les listes et les selecteurs de relations. Une mission ancienne
sans date reste explicitement a dater : on n'utilise pas sa date de creation
informatique comme date d'intervention.

## Une taxonomie avec une collection

`document-types` joue le role d'un vocabulaire partage. Chaque type a un nom et
une description. Une relation depuis les lignes de journee permet de choisir
un type existant ou d'en creer un avec les controles habituels de Payload.

Une cle normalisee empeche les doublons qui ne different que par la casse ou
les espaces. Les synonymes restent une question metier : ils ne sont pas
fusionnes automatiquement. Un type utilise ne peut pas etre supprime.
La collection et les releves exigent une session authentifiee.

## Relever et regrouper

Dans une journee, le tableau `eliminationEntries` contient une relation vers le
type, une annee extreme de debut, une annee extreme de fin et des precisions.
Ces champs sont reserves aux missions d'elimination ou de classement et
elimination. Les annees doivent etre entieres, ordonnees et non futures.
Un seul type est autorise par journee : modifier ses bornes au lieu de le
saisir une seconde fois.

La page de suivi regroupe ensuite tous les jours de la mission par identifiant
de type. Elle prend le minimum des annees de debut et le maximum des annees
de fin, meme si le journal est pagine. Les lignes sources restent intactes et
peuvent etre corrigees dans l'administration. Il ne s'agit ni d'une detection
des doublons physiques, ni d'une autorisation de destruction. Le formulaire
complet et le bordereau seront definis separement.

Les dates extremes sont pour le moment exprimees en annees, sans inventer le
1er janvier lorsque seul un millesime est connu.

## Mettre a jour une installation existante

Arreter l'application, puis dans son dossier :

```powershell
pnpm upgrade:mission-register
pnpm generate:types
pnpm generate:importmap
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm build
pnpm start
```

La mise a jour cree d'abord une copie SQLite dans le dossier local `.backups`
situe a cote du projet, puis ajoute les colonnes et tables necessaires.
Elle peut etre relancee ; elle n'invente aucune date et ne modifie pas les
anciennes journees. Sur une installation neuve, le schema est cree directement
par Payload, sans cette commande de reprise.

Les nouveaux tests couvrent les dates, les doublons, les bornes, le regroupement,
les acces, la suppression des types utilises et la reprise de la base. Le test
navigateur modifie une journee avec son calendrier puis verifie le recapitulatif
et l'affichage mobile.
