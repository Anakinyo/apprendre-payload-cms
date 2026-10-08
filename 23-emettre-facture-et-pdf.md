# 23. Émettre une facture et conserver son PDF

## Objectif et périmètre

Une relecture approuvée peut maintenant devenir une facture numérotée. L'utilisateur
confirme l'émission ; l'application recontrôle les données, attribue un numéro et
conserve les octets du PDF. Elle n'envoie aucun courriel et ne dépose rien sur un
portail externe.

Le parcours initial est limité aux fournisseurs EI, aux prestations de services
en France et au régime sans TVA déjà préparé dans les chapitres précédents. Les
pays du fournisseur et du client doivent être explicitement « France ». Les clients
publics et entreprises utilisent les mentions correspondant à leur catégorie.
Les associations, particuliers, acomptes, avoirs et opérations avec TVA ne sont
pas couverts par cette émission initiale.

Une facture PDF ne suffit pas à qualifier tout le circuit de transmission. Avant
une bascule réelle, vérifier les mentions et les obligations propres aux opérations
de l'entreprise avec les [informations officielles sur les factures](https://www.service-public.gouv.fr/entreprendre/vosdroits/F31808?profil=entrepreneur-individuel)
(consultées le 8 octobre 2026). L'application n'est pas présentée comme une solution
de facturation électronique certifiée ou connectée à Chorus Pro.

## 1. Une collection distincte du brouillon

`src/collections/IssuedInvoices.ts` définit `issued-invoices`. Elle conserve :

- les relations vers le brouillon, la relecture et son approbation ;
- le numéro, le compteur et le jour d'émission ;
- une copie des informations facturées ;
- le PDF encodé en base64 et son empreinte SHA-256 ;
- l'auteur et son adresse à la date d'émission.

Les accès en modification et suppression sont interdits. Les relations vers le
brouillon, la relecture et la décision sont uniques, comme le numéro et le compteur.
Ainsi, deux relectures du même brouillon ne permettent pas deux émissions.

La copie indépendante conserve le contenu du document même si une fiche client ou
des paramètres changent ensuite. Le PDF n'est pas régénéré au téléchargement.

## 2. Recontrôler dans la transaction

Le hook `beforeValidate` refuse l'opération sans utilisateur, confirmation explicite
ou transaction disponible. Il charge les documents avec `req`, `depth: 0` et
`overrideAccess: false`, puis rappelle `invoiceReadiness`.

Le diagnostic affiché précédemment dans une page n'est jamais accepté comme preuve
envoyée par le navigateur. Le serveur reconstruit le contenu actuel et vérifie
encore son identité avec la relecture approuvée.

Les champs calculés sont reconstruits sur le serveur. Un numéro ou un PDF fourni
dans la requête est ignoré. Les valeurs requises `draft` et `approval` des types
générés sont renseignées par le hook avant la validation Payload, pas choisies par
l'utilisateur.

## 3. Attribuer le numéro sans le perdre

L'adaptateur SQLite local utilise déjà des transactions immédiates et sérialise les
écritures dans le processus. Le hook utilise la session SQL de la requête pour :

1. vérifier qu'aucune facture n'existe pour ce brouillon ;
2. lire et valider le dernier compteur ;
3. mettre à jour le compteur sous condition de sa valeur précédente ;
4. produire le PDF ;
5. laisser Payload enregistrer la facture et valider la transaction.

Une erreur de rendu ou d'enregistrement entraîne un retour arrière de la transaction,
y compris du compteur. Les index uniques restent la dernière protection contre un
doublon. Ce mécanisme est spécifique à notre installation SQLite ; ne pas copier
le SQL tel quel vers PostgreSQL.

Une seconde demande pour un brouillon déjà facturé est refusée. Ce n'est pas une
API retournant automatiquement la première réponse : en cas de réponse réseau
incertaine, l'utilisateur actualise la page pour retrouver la facture éventuellement
créée. Il ne faut jamais fabriquer un nouveau brouillon pour contourner cette situation.

## 4. Verrouiller la reprise des numéros

Après la première émission dans l'application, le hook des paramètres refuse les
changements manuels de numérotation. Les autres paramètres restent modifiables.
Le traitement d'émission met lui-même à jour le compteur dans sa transaction.

Le préfixe reste fixe : aucun changement d'année ou redémarrage à zéro n'est
automatique. Toute évolution de série devra faire l'objet d'un traitement dédié.
Après bascule, ne pas continuer à émettre dans un autre outil sur la même série.

## 5. Générer et télécharger le PDF

`src/pdf/InvoiceDocument.tsx` utilise React PDF, déjà installé pour les devis.
Les parties, les lignes, les montants et les mentions proviennent du contenu figé.
La date d'émission et le numéro sont ajoutés au moment de la création.

`pdf-lib` inscrit ensuite la référence et la pagination sur chaque page. Le test
PDF utilise quarante lignes pour vérifier le passage sur plusieurs pages A4.
Les contrôles visuels restent nécessaires pour juger les coupures et la lisibilité.

Le champ base64 est caché de l'API générique. L'endpoint privé
`/api/issued-invoices/:id/pdf` authentifie l'utilisateur, lit ce champ en interne,
vérifie l'empreinte et renvoie les octets conservés. Ses en-têtes interdisent le
cache partagé et demandent un téléchargement. Le nom technique du fichier utilise
l'identifiant interne ; le numéro métier est inscrit dans le document.

## 6. Une confirmation humaine

Sur la relecture conservée, le formulaire d'émission apparaît uniquement si le
diagnostic ne détecte aucun blocage. L'utilisateur doit cocher la confirmation
avant de déclencher l'action. Le bouton est désactivé pendant la requête.

Une fois la facture créée, la page affiche son numéro, sa date et le téléchargement.
Elle ne propose plus d'émission pour ce brouillon. La collection « Factures émises »
permet également de retrouver les enregistrements dans l'administration.

## 7. Vérifier avant de déployer

Depuis l'application, arrêter le serveur de développement puis exécuter :

```powershell
pnpm install --frozen-lockfile
pnpm generate:types
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm lint
pnpm build
```

Les tests utilisent uniquement la base jetable prévue par `test.env`. Ne pas exécuter
les suites d'intégration et navigateur simultanément, car elles partagent cette base.

Les nouveaux tests couvrent les champs falsifiés, la confirmation, les changements
après approbation, l'échec du PDF, les demandes concurrentes, les doublons, le
verrouillage des numéros, l'immutabilité et le masquage du PDF dans l'API générique.
Le scénario navigateur contrôle aussi la confirmation, le téléchargement, le refus
anonyme, l'absence de seconde émission et l'affichage sur petit écran.

## 8. Mettre à jour la base locale

Cette fois une nouvelle table est nécessaire. Avec le serveur arrêté :

```powershell
pnpm backup:local
pnpm upgrade:chapter23
pnpm build
pnpm start
```

La commande de migration réalise une sauvegarde supplémentaire avant de créer la
table et ses index. Elle peut être relancée sans remettre le compteur à zéro. Elle
ne crée aucune facture et ne modifie pas les relectures historiques.

Ne jamais activer la synchronisation automatique du schéma sur la base réelle pour
remplacer cette procédure. Les scripts de préparation des tests ne sont pas des
outils de migration : ils effacent volontairement la base de test.

## Recette avant usage réel

Vérifier une installation d'essai séparée avec des données fictives, puis faire
valider les mentions, la reprise exacte de la numérotation, le PDF, les destinataires
et le circuit de dépôt. Ne pas émettre une facture « pour voir » dans la base réelle.

Le prochain lot concerne les paiements et soldes. La gestion des corrections et
avoirs doit être clarifiée avant de remplacer l'outil de facturation existant.
