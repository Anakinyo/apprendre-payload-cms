# Chapitre 9 - Numéroter le devis final et préparer son e-mail

## Objectif

Transformer une version approuvée en interne en un document final conservé,
puis préparer son e-mail. Chaque action nécessite une confirmation humaine.
Le mode initial est une **simulation sans envoi réseau**.

L'approbation interne n'est pas la signature du client. Le PDF final comprend
un emplacement « Bon pour accord », mais ce chapitre ne crée pas un service
de signature électronique ni une preuve de réception du message.

## 1. Installer les dépendances

Depuis le projet du chapitre 8, serveur arrêté :

```powershell
pnpm add nodemailer@10.0.14
pnpm add -D @types/nodemailer@8.0.2 smtp-server@3.19.17 @types/smtp-server@3.5.13 mailparser@3.9.35 @types/mailparser@3.9.0
```

Nodemailer prépare les messages et parle au serveur SMTP. `mailparser` permet
aux tests de lire la pièce jointe réelle. `smtp-server` fournit uniquement un
serveur fictif local pour les tests, pas un service de messagerie à déployer.

Le [transport de capture](https://nodemailer.com/transports/stream) produit
un message MIME sans le distribuer. Un fichier `.eml` contient le message et
ses pièces jointes : l'ouvrir dans un logiciel de messagerie ne constitue pas
une obligation de l'envoyer.

## 2. Ajouter trois collections

| Collection | Rôle | Règle importante |
| --- | --- | --- |
| `quote-number-series` | Année et prochain numéro disponible | Une série par année, pas de remise à zéro par l'API |
| `issued-quotes` | Document final, données figées et PDF | Une finalisation par aperçu, aucun remplacement |
| `quote-deliveries` | Journal des tentatives d'e-mail | Une seule réservation SMTP par devis final |

Les collections figurent dans `payload.config.ts`. Les champs `ui` affichent
les composants React, sans stocker de données métier supplémentaires :
`QuoteIssueActions` pour la création et `QuoteDeliveryActions` pour l'e-mail.

Les PDF et les messages MIME sont cachés dans l'admin et interdits en lecture
par les champs de l'API. Des endpoints authentifiés donnent accès à leurs
fichiers. `admin.hidden` seul ne protège pas un champ : il faut aussi son
contrôle `access.read`.

Les documents et le journal sont accessibles uniquement aux utilisateurs
connectés. Ils ne peuvent pas être réécrits ou supprimés par l'API publique.
Le nettoyage des fixtures utilise la Local API privilégiée, côté serveur.

### Mettre en place le code

Pour suivre depuis le chapitre 8, reprendre ces fichiers de référence dans
les mêmes dossiers du projet. Lire les sections suivantes avant de les ajouter :

- [QuoteNumberSeries.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/collections/QuoteNumberSeries.ts), [IssuedQuotes.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/collections/IssuedQuotes.ts) et [QuoteDeliveries.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/collections/QuoteDeliveries.ts) : schéma et hooks;
- [quoteIssue.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/domain/quoteIssue.ts) : contrôles du document et format du numéro;
- [requestSQL.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/database/requestSQL.ts) et [serializedSQLite.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/database/serializedSQLite.ts) : transactions locales;
- [quoteMail.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/mail/quoteMail.ts) et [dispatchQuote.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/mail/dispatchQuote.ts) : validation, transport et journal;
- [quoteIssue.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/endpoints/quoteIssue.ts) et [issuedQuotes.ts](https://github.com/Anakinyo/payload-archiviste/blob/main/src/endpoints/issuedQuotes.ts) : endpoints authentifiés;
- [QuoteIssueActions.tsx](https://github.com/Anakinyo/payload-archiviste/blob/main/src/components/QuoteIssueActions.tsx) et [QuoteDeliveryActions.tsx](https://github.com/Anakinyo/payload-archiviste/blob/main/src/components/QuoteDeliveryActions.tsx) : contrôles de l'admin.

Dans `payload.config.ts`, importer les trois collections et remplacer
l'import de `sqliteAdapter` par celui de `serializedSQLite`. Conserver les
options `client` et `push` du chapitre 8, puis les passer à `serializedSQLite`.
Ajouter les collections à la liste existante, sans retirer les précédentes :

```ts
collections: [Users, Media, Clients, Missions, Quotes, QuotePreviews,
  QuoteDecisions, QuoteNumberSeries, IssuedQuotes, QuoteDeliveries],
```

Dans `QuotePreviews.ts`, importer `quoteIssueEndpoints` et les ajouter à la
liste `endpoints` avec `...quoteIssueEndpoints`, après les endpoints de relecture.
Dans la branche approuvée de `QuoteReviewActions`, afficher :

```tsx
<QuoteIssueActions key={id} previewID={id} />
```

Reprendre aussi la variante finale de
[QuoteDocument.tsx](https://github.com/Anakinyo/payload-archiviste/blob/main/src/pdf/QuoteDocument.tsx)
et les règles `.quote-review__journal` du
[style de l'admin](https://github.com/Anakinyo/payload-archiviste/blob/main/src/app/%28payload%29/custom.scss).
Les composants existants conservent leur comportement pour les versions refusées
et les aperçus. La mise à jour de la base et la génération de l'import map
arrivent à la section 6; ne pas démarrer l'admin avant cette mise à jour.

## 3. Choisir le premier numéro disponible

Dans « Configuration → Séries de devis », créer une année et saisir le prochain
numéro **réellement libre dans l'entreprise**, après vérification des documents
déjà émis. Le tutoriel ne crée aucune série automatiquement dans la vraie base.

Exemple fictif : pour l'année 2026 et la valeur 45, le numéro sera
`D-2026-0045`. Le suivant sera `D-2026-0046`. Les nombres dépassant quatre
chiffres ne sont pas tronqués. Les séries sont limitées à 999999.

L'année est celle de la date du devis **figée dans l'aperçu**, et non celle de
l'horloge au moment du clic. Une version datée de l'année précédente nécessite
donc sa propre série. Vérifier cette date avant la validation interne.

Une série ne se modifie pas après sa création par l'admin. Cette première
version privilégie la protection contre les renumérotations accidentelles;
un besoin de correction administrative demandera une procédure contrôlée.

## 4. Finaliser exactement la version approuvée

La chaîne de données reste explicite :

```text
Devis modifiable → aperçu figé → décision interne → devis final figé
```

Le hook `beforeValidate` de `IssuedQuotes` :

1. exige un auteur connecté et `confirmed: true`;
2. charge l'aperçu et sa décision;
3. exige une approbation et vérifie l'empreinte du PDF relu;
4. contrôle les identités, les adresses, le SIRET et la mention TVA si nécessaire;
5. réserve le numéro dans la même transaction que la création du document;
6. génère le PDF final depuis le `snapshot` de l'aperçu;
7. conserve le PDF, son empreinte et l'identité de l'auteur.

Une modification du client ou du devis courant après approbation ne change
pas le document final. Pour corriger son contenu, créer et relire une nouvelle
version. Les contrôles de complétude ne certifient pas la conformité juridique
du devis : adapter et faire vérifier le modèle pour l'activité réelle.

`renderQuotePDF(snapshot, number, 'final')` réutilise le modèle du chapitre 7,
ajoute le numéro et l'espace d'accord, et remplace les mentions d'aperçu.
Les anciens PDF restent strictement identiques. Le modèle est générique,
pas encore une reproduction exacte d'une charte graphique d'entreprise.

## 5. Comprendre la transaction SQLite

Une transaction regroupe plusieurs écritures : soit tout est enregistré,
soit tout est annulé. Ici, une création en échec ne consomme pas de numéro.

Le compteur est incrémenté par une requête paramétrée, sur la session de
transaction de Payload, et non par « lire la valeur puis ajouter 1 en JavaScript » :

```sql
UPDATE quote_number_series
SET next_number = next_number + 1,
    updated_at = strftime('%Y-%m-%dT%H:%M:%fZ', 'now')
WHERE year = ? AND next_number BETWEEN 1 AND 999999
RETURNING next_number - 1 AS allocated;
```

Les contraintes uniques sur l'année, le numéro et l'aperçu protègent aussi
contre les doublons. Un deuxième appel à l'endpoint de finalisation retourne
le document existant au lieu d'en créer un autre.

L'adaptateur SQLite n'activait pas les transactions dans notre configuration
initiale. `serializedSQLite` active `transactionOptions` avec le comportement
`immediate` et met en file les transactions d'écriture dans le processus local.
Cela évite d'ouvrir des transactions libSQL concurrentes sur ce même client.
Les verrous SQLite continuent de s'appliquer entre processus : cette file n'est
pas un verrou distribué. Un conflit externe peut faire échouer une opération,
qui devra être relancée manuellement après vérification.

Cette intégration et sa requête SQL sont spécifiques à SQLite. Le passage à
PostgreSQL devra les remplacer et rejouer les tests de concurrence.
Voir [l'adaptateur local](https://github.com/Anakinyo/payload-archiviste/blob/main/src/database/serializedSQLite.ts)
et [la réservation du numéro](https://github.com/Anakinyo/payload-archiviste/blob/main/src/database/requestSQL.ts).

## 6. Mettre à jour une base locale existante

Comme au chapitre 8, ne pas accepter une reconstruction hasardeuse des tables
qui contiennent déjà des données. Conserver dans `.env` :

```dotenv
PAYLOAD_DISABLE_SCHEMA_PUSH=1
```

Copier le helper et le script de mise à jour du dépôt, puis ajouter à `scripts`
dans `package.json` :

```json
"upgrade:chapter9": "cross-env NODE_OPTIONS=--no-deprecation tsx scripts/upgrade-chapter9.ts"
```

Serveur et tests arrêtés, pour la base locale du chapitre 8 :

```powershell
pnpm upgrade:chapter9
pnpm generate:types
pnpm generate:importmap
```

Le script refuse la production et une URL différente de
`file:./payload-archiviste.db`. Il crée d'abord une sauvegarde dans `../.backups`,
puis ajoute les trois tables, leurs index et les relations de verrouillage
sans remplacer les anciennes lignes. Il ne choisit aucun numéro de départ.
Le helper est réexécutable; chaque lancement du script crée une sauvegarde.
Ce script pédagogique local n'est pas un système de migrations de production.

Sources : [helper](https://github.com/Anakinyo/payload-archiviste/blob/main/src/database/upgradeChapter9SQLite.ts),
[script](https://github.com/Anakinyo/payload-archiviste/blob/main/scripts/upgrade-chapter9.ts).

## 7. Essayer le mode simulation

Dans `.env`, ajouter ou conserver :

```dotenv
QUOTE_EMAIL_MODE=capture
ALLOW_CLIENT_EMAIL_SEND=no
```

L'adresse de contact de l'entreprise sert d'expéditeur pour la capture.
Une adresse valide est nécessaire, même si aucun message n'est transmis.

```powershell
pnpm dev
```

1. Compléter les identités et les adresses dans l'entreprise et le client.
2. Créer la série annuelle avec le premier numéro libre.
3. Enregistrer un devis, générer son aperçu et l'approuver en interne.
4. Confirmer « Je confirme la création du document final », puis créer le devis.
5. Ouvrir le PDF final et vérifier son numéro, ses montants et son contenu.
6. Suivre « Préparer l'e-mail » et vérifier le mode simulation affiché.
7. Relire le destinataire, l'objet et le message; cocher les deux confirmations.
8. Cliquer « Simuler l'envoi », essayer l'annulation, puis confirmer la simulation.
9. Télécharger le `.eml` du journal et vérifier sa pièce jointe.

Modifier le destinataire, l'objet ou le message invalide la confirmation
correspondante. Après chaque tentative, les confirmations sont remises à zéro.
La création du PDF et l'approbation interne ne déclenchent aucun e-mail.

## 8. Préparer un vrai SMTP, plus tard

Les valeurs suivantes sont un exemple de configuration, pas des identifiants
utilisables. Les obtenir auprès du fournisseur de messagerie et garder `.env`
hors de Git. Ne jamais copier un mot de passe dans le cours ou une capture.

```dotenv
QUOTE_EMAIL_MODE=smtp
ALLOW_CLIENT_EMAIL_SEND=yes
QUOTE_MAIL_FROM=devis@example.com
QUOTE_SMTP_HOST=smtp.example.com
QUOTE_SMTP_PORT=587
QUOTE_SMTP_SECURE=false
QUOTE_SMTP_USER=identifiant-fourni
QUOTE_SMTP_PASSWORD=secret-fourni
```

Avec le port STARTTLS habituel, `secure=false` n'autorise pas ici un envoi en
clair : `requireTLS=true` est imposé. Un fournisseur utilisant TLS dès la
connexion, souvent sur 465, demande `secure=true`. Respecter sa documentation
et ne pas désactiver la vérification des certificats.
Voir [la configuration SMTP de Nodemailer](https://nodemailer.com/smtp).

Redémarrer le serveur et actualiser l'interface après un changement de mode.
La requête inclut le mode que l'utilisateur a confirmé : si le serveur est
passé de simulation à SMTP depuis l'affichage, l'action est refusée.
L'envoi réel demande toujours les confirmations et le dernier clic humain.

Le serveur réserve et enregistre une tentative **avant** de contacter SMTP.
Une contrainte unique bloque une deuxième tentative SMTP sur le même devis,
y compris si la première reste en cours ou a un résultat incertain.
Les simulations peuvent être répétées et ne bloquent pas la future tentative SMTP.

« Accepté par le serveur SMTP » ne signifie ni livré, ni lu par le client.
Une erreur peut survenir après une acceptation distante : pas de relance
automatique. Vérifier le journal, le Message-ID et le fournisseur avant toute
action. La gestion contrôlée d'un renvoi est volontairement hors de ce chapitre.

## 9. Installer, exécuter et comprendre les tests

L'installation initiale de Vitest et Playwright reste décrite aux chapitres
[4](./04-configurer-entreprise-global.md) et [6](./06-calculer-montants.md).
Sur un clone du projet :

```powershell
pnpm install --frozen-lockfile
pnpm exec playwright install chromium
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm exec tsc --noEmit
pnpm lint
pnpm build
```

`pnpm test` lance successivement les trois suites. Ne pas les lancer en parallèle
ni modifier le code pendant un test navigateur.

Les suites d'intégration et navigateur préparent désormais toutes deux une base
de test neuve via le script du chapitre 7, puis désactivent la synchronisation
concurrente du schéma avec `PAYLOAD_TEST_SCHEMA_READY=1`.
Seule `file:./payload-test.db` peut être recréée. Ne jamais y stocker des données
à conserver. `test.env` impose la capture et désactive les envois réels.

Conserver les autres commandes de `package.json` et mettre à jour `test:int` :

```json
"test:int": "cross-env NODE_OPTIONS=--no-deprecation DOTENV_CONFIG_PATH=./test.env tsx scripts/prepare-e2e.ts && cross-env NODE_OPTIONS=--no-deprecation DOTENV_CONFIG_PATH=./test.env PAYLOAD_TEST_SCHEMA_READY=1 vitest run --config ./vitest.config.mts tests/int"
```

Ajouter à `test.env` les mêmes deux paramètres de sécurité que pour la
simulation : `QUOTE_EMAIL_MODE=capture` et `ALLOW_CLIENT_EMAIL_SEND=no`.

Les tests SMTP changent temporairement ces paramètres vers un serveur fictif
sur `127.0.0.1` et un port aléatoire. L'exception sans TLS exige simultanément
`NODE_ENV=test`, une adresse locale et `QUOTE_SMTP_LOCAL_TEST=yes`. Ne pas
mettre ce dernier paramètre dans la configuration de l'application.

Les nouvelles vérifications couvrent les confirmations manquantes, les adresses
invalides et l'injection d'en-têtes, les pièces jointes MIME exactes, la numérotation
transactionnelle, les doublons et la concurrence, l'immuabilité, les accès anonymes,
les migrations sans perte et le blocage des reprises après SMTP accepté ou incertain.
Le navigateur vérifie aussi l'annulation, le parcours mobile et ordinateur,
le téléchargement du document et de sa simulation.

Sources complètes : [tests unitaires](https://github.com/Anakinyo/payload-archiviste/tree/main/tests/unit),
[tests d'intégration](https://github.com/Anakinyo/payload-archiviste/tree/main/tests/int),
[parcours navigateur](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/e2e/admin.e2e.spec.ts).

## Limites et prochaine étape

Nous disposons d'un parcours local protégé jusqu'au document final et à son
e-mail explicite. Il reste à personnaliser le modèle, valider les mentions pour
l'activité, configurer un vrai fournisseur et définir les rôles de production.
Le stockage PDF/MIME en base est simple pour l'apprentissage, mais devra être
réévalué pour des volumes importants et une politique de conservation.

Le chapitre 10 préparera PostgreSQL, les migrations, les sauvegardes et le
déploiement. Les factures, paiements et obligations de facturation électronique
appartiendront à une seconde partie, avec vérification des règles applicables.
