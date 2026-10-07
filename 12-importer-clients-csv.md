# Chapitre 12 - Importer des clients depuis un tableur

## Objectif

L'entreprise possède un tableur de contacts et de trajets. Ressaisir chaque
client serait fastidieux. Nous ajoutons un import en quatre étapes : fichier,
correspondance des colonnes, prévisualisation, confirmation de la création.

Cette version importe un **CSV UTF-8**, pas un classeur Excel ni un document
Google Sheets directement. Exporter la feuille concernée en CSV, puis utiliser
le fichier téléchargé. Aucun compte Google, accès distant ou abonnement n'est
nécessaire dans l'application. Le fichier réel de l'entreprise ne doit jamais
être ajouté au dépôt Git ni publié avec le cours.

L'import crée des clients. Il ne modifie, ne fusionne et ne supprime aucune fiche
existante. Il ne génère pas de devis et n'envoie pas d'e-mail.

## 1. Préparer le fichier

Le fichier comporte une ligne d'en-têtes puis au maximum 500 clients, dans
2 Mo. Les séparateurs proposés sont virgule, point-virgule et tabulation.
Un [exemple fictif](https://github.com/Anakinyo/payload-archiviste/blob/main/public/exemple-import-clients.csv)
est aussi téléchargeable depuis l'écran d'import.

Deux colonnes sont indispensables :

- un identifiant stable : par exemple le code INSEE d'une commune ;
- le nom affiché du client.

Le numéro de ligne du tableur n'est pas un identifiant stable : un tri ou une
insertion le change. Pour un autre annuaire, attribuer une référence durable
dans le fichier source et la conserver d'un export à l'autre.

Champs facultatifs : nom légal, catégorie, SIRET, e-mail, téléphone, nom et
fonction du contact, adresse, code postal, ville, pays, point de départ,
distance aller simple en kilomètres et durée aller simple en minutes.

Les catégories techniques sont `collectivite`, `entreprise`, `administration`,
`association` et `particulier`. Sans colonne catégorie, le formulaire propose
une catégorie par défaut. Le pays absent devient `France`.

Conserver identifiants, téléphones, SIRET et codes postaux comme **texte** dans
le tableur. L'import conserve `01230`, mais ne peut pas recréer un zéro déjà
perdu avant l'export. Les distances acceptent `42.5` ou `42,5`, sans `km`.
Dans un CSV séparé par virgules, une valeur à virgule doit être entre guillemets.
Les durées sont des nombres de minutes : convertir `1h30` en `90` avant l'import.

## 2. Installer un vrai parseur CSV

```powershell
pnpm add csv-parse
```

Pour reproduire exactement le projet, récupérer son fichier de verrouillage
et utiliser `pnpm install --frozen-lockfile`. Le projet utilise `csv-parse`
7.0.3. Nous importons `parse` depuis `csv-parse/sync`, uniquement côté serveur.

Un `split(',')` casserait les adresses contenant une virgule et les cellules
multilignes. Le parseur gère guillemets, fins de ligne et BOM UTF-8. La conversion
automatique en nombres reste désactivée pour préserver les zéros initiaux.
Les erreurs de structure ne sont pas ignorées : le fichier doit être corrigé.
Voir les [options officielles](https://csv.js.org/parse/options/).

```ts
const records = parse(csv, {
  delimiter,
  bom: true,
  skip_empty_lines: true,
  trim: true,
  cast: false,
  max_record_size: 32000,
  to: MAX_IMPORT_ROWS + 2,
})
```

Lire une ligne au-delà de la limite permet de refuser un fichier trop long,
au lieu de tronquer silencieusement l'import. Les en-têtes doivent être uniques,
non vides et limités à 40 colonnes. Les champs importés sont limités à 250
caractères; la référence stable, à 100 caractères.

## 3. Séparer interface, domaine et base

| Fichier | Responsabilité |
| --- | --- |
| `src/domain/clientImportFields.ts` | Types partagés et libellés de colonnes |
| `src/domain/clientImport.ts` | Lecture, normalisation, validation et détection des doublons |
| `src/imports/clientImport.ts` | Requêtes Payload, prévisualisation signée et transaction |
| `src/endpoints/clientImport.ts` | Endpoint HTTP authentifié |
| `src/components/ClientImport.tsx` | Formulaire React interactif |
| `src/app/(frontend)/clients/import/page.tsx` | Page serveur et contrôle de connexion |

Ces chemins sont relatifs à la racine de l'application. Le
[code complet](https://github.com/Anakinyo/payload-archiviste/tree/main/src/imports)
accompagne le cours. La collection `Clients` possède déjà `importKey` et
`travel` : aucun nouveau champ et aucune migration ne sont nécessaires pour
une base à jour au chapitre 11.

Le composant interactif utilise `'use client'` pour les sélections et les appels
HTTP. Le parseur, les accès à la base et la signature restent côté serveur.
Comme le bouton d'aperçu du chapitre 7, ses champs restent désactivés jusqu'à
l'hydratation React : une interaction ne doit pas être perdue pendant le
chargement du JavaScript. Le projet utilise `useSyncExternalStore` pour ce
contrôle, avec une valeur serveur `false` et une valeur navigateur `true`.
Le CSV et l'aperçu restent temporairement en mémoire; ni stockage navigateur,
ni fichier source archivé, ni journal d'import persistant ne sont ajoutés.
Actualiser la page oblige à reprendre la sélection du fichier.

## 4. Choisir la correspondance des colonnes

Depuis **Clients**, ouvrir **Importer des clients**, sélectionner le fichier et
son séparateur, puis **Lire le fichier**. Le serveur ne crée encore rien.

L'écran présente ensuite une sélection pour chaque champ cible. Il suggère
certaines correspondances usuelles (`nom`, `email`, `cp`, `code insee`…), mais
il faut vérifier les sélections et les exemples affichés. Les colonnes non
affectées ne sont pas importées. Une colonne ne peut pas alimenter deux champs.

La source, par exemple `communes`, qualifie l'identifiant :

```text
source : communes
référence : 54395
importKey : communes:54395
```

La source accepte 2 à 40 caractères : première lettre minuscule, puis lettres
minuscules, chiffres, tirets ou soulignés. Garder la même source lors des imports
suivants. Changer arbitrairement la source contournerait la reconnaissance par
référence, même si les autres contrôles peuvent encore détecter un doublon.

## 5. Prévisualiser sans écrire

Le domaine produit quatre états :

| État | Comportement |
| --- | --- |
| Nouveau | Création possible, sélection modifiable |
| Déjà importé | Même référence reconnue, fiche existante conservée |
| À vérifier | Doublon ou correspondance ambiguë, création exclue |
| Invalide | Champ incorrect, création exclue |

La détection compare référence d'import, SIRET, e-mail normalisé et nom avec
localité (code postal, ou ville si le code manque). Les noms sont normalisés
pour ignorer casse, accents et espaces superflus. Les clients archivés restent
pris en compte. Deux lignes partageant une identité dans le fichier sont toutes
deux exclues : nous ne choisissons pas arbitrairement la première.

Une adresse e-mail partagée peut provoquer un faux positif; le lien vers la
fiche existante permet une vérification humaine. Ce n'est pas une recherche
approximative : une faute dans le nom avec d'autres identifiants différents
peut passer inaperçue. Relire la prévisualisation reste indispensable.

Même si le fichier contient des coordonnées plus récentes, une référence déjà
importée reste **conservée sans mise à jour**. Modifier la fiche séparément dans
Payload après vérification. L'import des mises à jour formera une évolution
distincte, avec comparaison explicite des champs et politique de conflits.

Chaque ligne peut être dépliée avec **Coordonnées complètes** pour vérifier les
valeurs normalisées : contact, adresse, catégorie et trajet, notamment. Les
exemples des trois premières lignes dans le choix des colonnes ne suffisent pas
à valider un fichier entier.

## 6. Contrôler l'endpoint Payload

Nous ajoutons `clientImportEndpoints` à la propriété `endpoints` de la
configuration Payload. `POST /api/client-import` accepte trois actions :
`inspect`, `preview` et `confirm`.

```ts
import { clientImportEndpoints } from './endpoints/clientImport'

export default buildConfig({
  endpoints: clientImportEndpoints,
  // Conserver les collections, globals et autres options déjà configurés.
})
```

Dans un projet qui possède déjà des endpoints globaux, ajouter ces endpoints
au tableau existant au lieu de le remplacer. Ici, les endpoints des devis
sont déclarés dans leurs collections et restent inchangés.

Chaque action exige un utilisateur connecté. Les lectures et créations passent
`req` et `overrideAccess: false` à la Local API : elles ne contournent pas les
permissions des collections. Le serveur vérifie également l'origine du navigateur
lorsqu'elle est présente, le contenu JSON et les limites de taille.
Les réponses portent `Cache-Control: private, no-store`.

La taille HTTP est contrôlée pendant la lecture du flux, puis celle du CSV
est contrôlée avant parsing. Une validation dans React améliore l'expérience,
mais ne remplace jamais ces contrôles : une requête peut être fabriquée sans
utiliser le formulaire.

## 7. Confirmer la version réellement relue

La prévisualisation renvoie un jeton valable quinze minutes, signé avec
`PAYLOAD_SECRET`. Il lie l'utilisateur aux données normalisées, aux résultats
de la détection des doublons et à l'expiration. Le secret n'est jamais envoyé
au navigateur. Ce mécanisme évite une table de prévisualisations pour ce premier
import; il ne constitue pas un journal d'audit persistant.

L'utilisateur choisit les lignes nouvelles, puis coche la confirmation. Le
serveur ne fait pas confiance aux statuts renvoyés par le navigateur :

1. vérifier signature, expiration, utilisateur et confirmation ;
2. ouvrir une transaction SQLite ;
3. relire les clients et recalculer le plan ;
4. comparer son empreinte avec celle de l'aperçu signé ;
5. créer seulement les lignes sélectionnées, nouvelles et valides ;
6. valider la transaction, ou tout annuler en cas d'erreur.

Si une autre opération crée entre-temps un client correspondant, il faut
actualiser la prévisualisation. Une nouvelle tentative après succès ne recrée
pas les mêmes références : le plan a changé. Les contraintes uniques sur
`importKey` et `siret` apportent un contrôle supplémentaire dans la base.

Le lot est **atomique** : si la deuxième création échoue, la première est
annulée. Cela n'annule pas les imports réussis auparavant. Faire une sauvegarde
avant le premier import réel reste recommandé, serveur arrêté :

```powershell
pnpm backup:local
```

Cette version privilégie des lots modestes. Au-delà de 500 lignes, préparer
plusieurs fichiers avec la même source. Elle refuse aussi les bases de plus
de 50 000 clients : un import massif demanderait index dédiés et tâches de fond.

## 8. Tester et lancer

Les tests utilisent uniquement la base jetable `payload-test.db`. Aucun fichier
client réel n'est nécessaire.

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm exec tsc --noEmit
pnpm lint
pnpm build
pnpm start
```

`pnpm test` lance également les trois suites successivement. Ne pas lancer
plusieurs suites simultanément : intégration et navigateur réinitialisent la
même base de test. Les fichiers d'intégration tournent désormais successivement
avec `--no-file-parallelism`, comme les suites navigateur avec un seul worker,
pour éviter les écritures SQLite concurrentes depuis plusieurs processus.

Couverture ajoutée : guillemets et cellules multilignes, BOM, zéros initiaux,
limites, colonnes obligatoires, nombres français, validation, doublons,
authentification, origine HTTP, absence d'écriture à la prévisualisation,
jeton altéré, données modifiées, annulation d'un lot, sélection partielle et
réimport sans écrasement. Playwright vérifie aussi le formulaire et ses états
sur grands et petits écrans. Cela ne rend pas l'application accessible depuis
un téléphone : le serveur reste limité à `127.0.0.1` sur le portable.

Sources des tests : [unitaires](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/unit/clientImport.unit.spec.ts),
[intégration](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/int/clientImport.int.spec.ts),
[navigateur](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/e2e/clientImport.e2e.spec.ts).

## Bilan et suite

Nous avons utilisé un endpoint Payload, la Local API avec permissions, un
composant React interactif et une transaction pour traiter un besoin métier
concret. Le tableur n'est pas connecté en continu : importer un export est
une action ponctuelle et manuelle, pas une synchronisation.

Le [chapitre 13](./13-personnaliser-devis-et-mission.md) personnalise les devis
et fige les informations publiques de mission. XLSX, connexion directe à
Google Sheets, mise à jour par comparaison, historique d'import et annulation
d'un import validé restent des évolutions possibles, non implémentées ici.
