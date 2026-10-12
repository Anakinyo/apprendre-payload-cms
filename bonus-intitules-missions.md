# Bonus : proposer un intitulé de mission automatiquement

## Besoin

Une nouvelle mission reçoit un intitulé `Type de mission - Client - Année`,
par exemple `Classement - Mairie exemple - 2027`.
L'année vient de la date de début. En l'absence de date, on utilise l'année
de création, conservée lors des modifications ultérieures.

## Mise en place

1. Centraliser les libellés et la composition du texte dans
   `src/domain/missionTitle.ts`.
2. Ajouter un champ personnalisé Payload dans `src/components/MissionTitleField.tsx`.
   `useFormFields` observe le client, le type et la date ; `useField` met à jour
   l'intitulé. La lecture du client utilise l'API authentifiée habituelle.
3. Annuler la requête précédente quand le client change : une réponse tardive
   ne doit pas remplacer le nom du nouveau client.
4. Refaire le calcul côté serveur dans le hook `beforeValidate` de `Missions`.
   Le navigateur apporte le confort ; le serveur garantit le comportement
   même pour une création par API.
5. Déclarer le composant dans `admin.components.Field`, puis lancer
   `pnpm generate:importmap`.

Un intitulé personnalisé reste conservé. Un intitulé automatique suit les
changements de date, de type ou de client lors de l'enregistrement de la mission.
Les anciennes missions ne sont pas renommées en masse. Renommer un client
ne déclenche pas une réécriture automatique de toutes ses missions.
Les documents déjà figés restent inchangés.

## Vérifications

```powershell
pnpm test:unit
pnpm test:int
pnpm test:e2e
pnpm build
```

Les tests couvrent la composition, l'année de début, l'année de repli,
les mises à jour côté serveur, la conservation d'un intitulé personnalisé
et la suggestion dans le formulaire réel.

Pour la recette manuelle, créer une mission de test dans une installation
séparée, choisir un client et un type, puis modifier l'année. Vérifier aussi
qu'un intitulé saisi à la main n'est pas écrasé.

Ce bonus n'ajoute aucune colonne à la base : aucune migration n'est nécessaire.
