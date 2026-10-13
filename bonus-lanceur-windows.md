# Bonus : ouvrir l'application par double-clic sous Windows

Ce bonus concerne l'utilisation quotidienne d'une application deja installee.
Il ne remplace pas le premier chapitre : Node.js, les dependances, le fichier
`.env` et la base locale doivent etre prepares au prealable.

## Utilisation quotidienne

1. Ouvrir le dossier du projet.
2. Double-cliquer sur **Lancer application.cmd**.
3. Attendre l'ouverture du navigateur, puis se connecter avec son compte habituel.

Le serveur reste en arriere-plan. Fermer le navigateur ne l'arrete pas.
Un nouveau double-clic reutilise le serveur demarre par ce lanceur, sans creer
une seconde instance. Apres un redemarrage de Windows, relancer le fichier.
L'application reste accessible seulement sur cet ordinateur, a
`http://127.0.0.1:3000`, pas depuis un telephone. Aucun abonnement n'est ajoute.

Pour un acces plus pratique : clic droit sur le fichier, **Afficher plus
d'options > Envoyer vers > Bureau (creer un raccourci)**. Creer un raccourci,
ne pas deplacer le fichier seul : il utilise le dossier `scripts` du projet.

## Premiere preparation et mises a jour

Une personne technique prepare l'installation selon les chapitres du cours,
puis lance `pnpm build` dans le dossier du projet. Le lanceur utilise cette
version compilee, sans recompiler a chaque ouverture.

Apres une modification du code : arreter le serveur de cette application,
effectuer les sauvegardes et migrations requises par le chapitre concerne,
relancer `pnpm build`, puis utiliser le lanceur. Ne pas lancer les tests navigateur
ou une compilation pendant que la version quotidienne fonctionne : ils partagent
le dossier `.next`. Pour un arret simple sans commande, redemarrer Windows avant
les operations de maintenance, puis ne pas ouvrir le lanceur avant leur fin.

## Comment le lanceur fonctionne

Le fichier `.cmd` appelle `scripts/launch-local.ps1` avec Windows PowerShell.
Le script verifie les prerequis, demarre Next.js sur l'adresse locale uniquement,
attend une reponse du serveur puis ouvre le navigateur par defaut.

Un verrou empeche les lancements simultanes. Le numero du processus et son heure
de demarrage permettent de reconnaitre le serveur deja lance. Si le port 3000
est occupe par un autre serveur, le lanceur s'arrete sans tuer ce processus.
Les journaux et cet etat temporaire restent dans `.local-launcher`, exclu de Git.
Ils ne contiennent pas une sauvegarde de la base de donnees.

Il s'agit d'un **lanceur**, pas d'un executable autonome : Node.js et les fichiers
du projet restent necessaires. L'option PowerShell `ExecutionPolicy Bypass`
ne s'applique qu'a cette execution, sans modifier la politique globale du PC.
N'executer que des scripts dont on connait la provenance ; une politique de
securite d'entreprise peut interdire leur execution.

## En cas de probleme

- Node.js introuvable : suivre le bonus d'installation du premier chapitre.
- Installation incomplete : verifier `.env`, les dependances et `pnpm build`.
- Port occupe : fermer l'ancien serveur lance dans un terminal, ou redemarrer
  Windows. Le lanceur n'arrete jamais arbitrairement une autre application.
- Serveur en erreur : lire `.local-launcher/server-error.log` et `server.log`.

Un echec laisse la fenetre ouverte pour lire le message. Le lanceur ne reinstalle
rien, n'effectue aucune migration et ne change ni les comptes ni les donnees.
