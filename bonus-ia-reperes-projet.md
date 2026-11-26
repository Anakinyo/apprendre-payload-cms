# Bonus IA — Donner à un assistant de code les repères du projet

Après une pause, un assistant doit retrouver les règles de travail et le code du parcours à modifier. Trois fichiers Markdown suffisent dans ce fil rouge. Ils se versionnent avec le projet et restent utiles aux personnes qui découvrent le dépôt.

## 1. Mettre les règles durables dans `AGENTS.md`

À la racine du dépôt de l’application, créer ou compléter `AGENTS.md`. Si Next.js y a déjà généré un bloc, le conserver et ajouter les règles du projet en dessous. Codex découvre ce fichier quand on lance une session depuis le dépôt. [Guide officiel de Codex](https://learn.chatgpt.com/docs/agent-configuration/agents-md)

Voici un point de départ à adapter aux commandes réellement disponibles dans son dépôt.

```markdown
# Travailler sur l’application

- L’application tourne en local sous Windows avec Payload, Next.js et SQLite.
- Utiliser pnpm. Voir le README pour trouver le code du parcours concerné.
- Tester avec des données fictives sur la base de test.
- Les suites d’intégration et navigateur réinitialisent la même base de test.
  Les exécuter l’une après l’autre.
- Tout envoi de devis ou de facture à un client demande une validation humaine.
- Si un parcours utilisateur change, vérifier le chapitre correspondant du cours.
```

Dans ce fil rouge, le dépôt de l’application utilise également un `CLAUDE.md` contenant `@AGENTS.md` pour partager ces consignes avec Claude Code. Chaque dépôt peut avoir son propre `AGENTS.md`. Celui du cours précise par exemple que les chapitres sont en français et que les exemples restent fictifs.

## 2. Mettre la carte du code dans le README

Ajouter quelques lignes qui indiquent où commencer une recherche. Pour l’application du fil rouge, `src/app` et `src/components` contiennent les écrans, `src/collections` décrit les données Payload, `src/domain` porte les règles métier, et `src/pdf` contient les générateurs. Le dossier `tests` regroupe les scénarios de vérification.

Pour reprendre les étiquettes de classement, par exemple, commencer par `src/pdf/ClassificationLabelsDocument.tsx` pour le rendu, puis consulter `src/domain/classificationLabels.ts` pour les dimensions et la plage de numéros.

## 3. Garder un point de reprise

Dans ce cours, [AVANCEMENT.md](./AVANCEMENT.md) indique ce qui est livré et ce qui reste à vérifier en usage réel. Après un changement significatif, actualiser la ligne concernée. [RECETTE.md](./RECETTE.md) donne les essais à faire sur le terrain, comme vérifier l’alignement des étiquettes sur une vraie planche.

## Exercice

Ouvrir une nouvelle session d’assistance dans le dépôt de l’application et lui demander où modifier le contenu d’une étiquette de classement, quels tests lancer et quelle base utiliser. Vérifier les chemins et les commandes indiqués. Si une information manque régulièrement, ajouter une ligne au fichier qui correspond à son rôle.

Des repères courts peuvent réduire le temps de recherche et le contexte à relire. Le gain en tokens dépend de la tâche. Au fil du projet, remplacer les consignes périmées et garder les secrets ainsi que les données réelles hors de ces fichiers.
