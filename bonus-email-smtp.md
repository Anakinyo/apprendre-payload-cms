# Bonus : configurer les e-mails et le mot de passe oublie

Ce bonus complete le chapitre 9 sans modifier le parcours des factures.
L'entreprise utilise une boite e-mail existante. Payload lui confie ses e-mails
via SMTP : aucun nouveau service payant n'est impose par le code.
Les limites et conditions du fournisseur de messagerie restent applicables.

## 1. Recuperer les parametres

Dans la documentation de votre fournisseur, relever le serveur SMTP, le port,
l'identifiant et l'adresse expediteur autorisee. Ne pas confondre SMTP (envoi)
avec IMAP (lecture). Ne pas deviner le serveur a partir du nom de domaine.
Le port 465 utilise TLS direct ; le port 587 utilise STARTTLS.
Un mot de passe d'application peut etre necessaire selon le fournisseur.

## 2. Configurer sous Windows

Dans le dossier de l'application :

```powershell
pnpm email:configure
pnpm email:verify
```

La premiere commande demande les parametres et masque la saisie du mot de passe.
Elle les conserve dans `.smtp.local.json`, exclu de Git. Attention : ce fichier
contient le secret en clair, pas sous forme chiffree. Proteger la session Windows,
ne jamais le publier ni le joindre a une demande d'aide. Il ne fait pas partie
des sauvegardes metier ; il faudra reconfigurer SMTP apres une restauration.

La seconde commande verifie la connexion et l'authentification, sans envoyer de
message. Elle ne prouve donc pas encore la reception d'un e-mail.

Pour une configuration manuelle (notamment macOS/Linux), renseigner dans `.env` :

```dotenv
MAIL_MODE=smtp
MAIL_FROM=contact@example.com
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=contact@example.com
SMTP_PASSWORD=mot-de-passe-a-remplacer
APP_URL=http://127.0.0.1:3000
```

Ces valeurs sont des exemples, pas un service utilisable. Les variables
d'environnement prennent priorite sur le fichier local. Redemarrer l'application
apres une modification de `.env`. Ne jamais committer de vrais identifiants.

## 3. Tester le parcours complet

Demarrer l'application, ouvrir `/admin/forgot` et saisir l'adresse du compte
utilisateur existant, pas obligatoirement celle de l'expediteur SMTP. Consulter
la boite et les courriers indesirables. Suivre le lien, choisir un nouveau mot
de passe puis se connecter. Le lien expire apres une heure et n'est utilisable
qu'une fois. Attendre au moins 15 secondes entre deux demandes.

Avec l'installation locale, ouvrir le lien sur l'ordinateur qui fait tourner
le serveur. `127.0.0.1` sur un telephone designe le telephone, pas cet ordinateur.
Une connexion Internet est necessaire pour envoyer et recevoir le mail.

## 4. Comprendre le code

`siteMailConfig.ts` valide les parametres et impose une connexion chiffree.
`siteEmailAdapter.ts` branche l'adaptateur officiel Nodemailer sur Payload.
Les e-mails d'authentification et `payload.sendEmail(...)` utilisent ce transport.
Sans configuration, l'envoi echoue explicitement : aucun lien de reinitialisation
n'est remplace par un message dans la console.

La collection Users personnalise l'objet, le contenu et la duree du lien.
L'adresse du lien provient de `APP_URL`, pas d'un en-tete HTTP fourni par un visiteur.

Avec Payload 3.90.1 et notre verrou SQLite `BEGIN IMMEDIATE`, la reservation
anti-repetition native sur une connexion separee bloque. `resetThrottle.ts`
realise donc la reservation dans la transaction existante. Le delai de 15 secondes
est conserve ; un echec d'envoi annule la reservation avec la transaction.
Ce contournement est specifique a cette installation et doit etre reevalue lors
d'une mise a jour de Payload ou d'un passage a PostgreSQL.

## 5. Garder les envois clients sous controle

Configurer SMTP n'active pas les envois de devis aux clients. Ceux-ci restent en
mode capture par defaut et exigent toujours une confirmation humaine.
Pour autoriser leur transport SMTP, le chapitre 9 conserve ses deux reglages :
`QUOTE_EMAIL_MODE=smtp` et `ALLOW_CLIENT_EMAIL_SEND=yes`.
Sans `QUOTE_SMTP_HOST`, le transport global est reutilise. Les anciens reglages
`QUOTE_SMTP_*` restent prioritaires quand un serveur specifique est renseigne.
Ne pas activer ces options simplement pour tester un mot de passe oublie.

## 6. Tests automatises

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
```

Les tests unitaires verifient TLS, les parametres et le lien local. Le test
`tests/int/passwordMail.int.spec.ts` utilise une boite SMTP de test sur la machine,
un utilisateur fictif et la base de test : reception du message, limitation des
demandes, changement du mot de passe, connexion et refus de reutiliser le lien.
Il ne contacte aucun client et ignore `.smtp.local.json` en environnement de test.
Cela ne remplace pas la verification de reception avec le vrai fournisseur.

References : [e-mail dans Payload](https://payloadcms.com/docs/email/overview)
et [e-mails d'authentification](https://payloadcms.com/docs/authentication/email).
