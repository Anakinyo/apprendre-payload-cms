# Chapitre 13 - Personnaliser le devis et figer les informations de mission

## Objectif

L'entreprise souhaite adapter la présentation de ses devis sans modifier le
code à chaque changement de texte. Les devis liés à une mission doivent aussi
reprendre son cadre public : diagnostic, volume estimé, durée, conditions de
démarrage et livrables prévus.

Nous ajoutons un modèle configurable dans le global de l'entreprise, puis
copions sa configuration et les informations de mission dans chaque nouvel
aperçu. Les documents enregistrés restent immuables.

Ce chapitre n'ajoute ni facture, ni ordre de mission distinct, ni génération de
bordereau réglementaire. Il ne modifie pas la validation humaine ou l'envoi des
devis. Les textes saisis par l'entreprise doivent être relus avant validation;
ils ne deviennent pas des mentions juridiquement vérifiées par l'application.

## 1. Ajouter les paramètres du modèle

Dans `src/globals/CompanySettings.ts`, ajouter le groupe `quoteTemplate` :

```ts
{
  name: 'quoteTemplate',
  type: 'group',
  label: 'Modèle des nouveaux devis',
  fields: [
    { name: 'heading', type: 'text', label: 'Titre du document',
      maxLength: 80, defaultValue: 'Devis' },
    { name: 'introduction', type: 'textarea', label: 'Introduction', maxLength: 2000 },
    { name: 'conclusion', type: 'textarea', label: 'Conclusion', maxLength: 2000 },
    { name: 'accent', type: 'select', label: 'Couleur du modèle', defaultValue: 'charcoal',
      options: [
        { label: 'Gris anthracite', value: 'charcoal' },
        { label: 'Vert', value: 'green' },
        { label: 'Bleu', value: 'blue' },
      ] },
  ],
}
```

Une **Group field** structure plusieurs valeurs dans un objet TypeScript.
Les **Text / Textarea fields** limitent les textes; une **Select field** propose
une liste fermée. L'administration Payload fournit directement les contrôles.
Il n'est pas nécessaire de construire un éditeur de modèle personnalisé.

Nous utilisons du texte en clair, sans HTML, expression JavaScript ou syntaxe
`{{client}}`. Les informations variables du devis sont déjà injectées depuis
les champs structurés : entreprise, client, mission, lignes et montants. Les
trois couleurs sont traduites côté serveur en codes hexadécimaux autorisés.

Les valeurs par défaut donnent le titre `Devis`, des textes supplémentaires
vides et une couleur anthracite. Un titre vide retombe sur `Devis`. Aucun texte
commercial ou contractuel n'est ajouté automatiquement pour l'entreprise.

Après l'ajout des champs :

```powershell
pnpm generate:types
pnpm exec tsc --noEmit
```

Le type `CompanySetting` généré contient désormais `quoteTemplate`. Ne pas
modifier manuellement `src/payload-types.ts` : c'est une sortie du générateur.

## 2. Mettre à jour une base locale existante

Une base neuve est créée par Payload avec les nouveaux champs. Pour une base
existante à jour au chapitre 12, **arrêter le serveur** avant la mise à jour :

```powershell
pnpm backup:local
pnpm upgrade:chapter13
```

Le script `scripts/upgrade-chapter13.ts` est réservé à la base locale
`file:./payload-archiviste.db`, hors mode production. Il crée une copie SQLite
avec `VACUUM INTO` avant toute modification, puis appelle
`src/database/upgradeChapter13SQLite.ts`.

L'upgrade ajoute quatre colonnes texte nullables à `company_settings`, sans
reconstruire la table :

```text
quote_template_heading
quote_template_introduction
quote_template_conclusion
quote_template_accent
```

Les lignes et PDF existants ne sont ni copiés, ni remplacés. Le helper peut
être exécuté plusieurs fois : il ajoute uniquement les colonnes absentes et
conserve les personnalisations déjà enregistrées. Les valeurs nulles des
anciennes lignes sont prises en charge par les valeurs de repli du domaine.

Conserver `PAYLOAD_DISABLE_SCHEMA_PUSH=1` dans le `.env` de cette base existante.
Ne pas supprimer la base en cas d'erreur. Ce script pédagogique local n'est
pas une migration de production : un déploiement distant demanderait le
workflow de migrations adapté à sa base et à son environnement.

## 3. Normaliser et valider le modèle côté serveur

`src/domain/quoteDesign.ts` centralise les valeurs de repli et la validation.
Les champs provenant de Payload peuvent être absents ou `null`, notamment
juste après l'upgrade. Le modèle figé contient ensuite des chaînes et une
couleur autorisée, jamais des valeurs nulles.

```ts
type QuoteDesign = {
  heading: string
  introduction: string
  conclusion: string
  accent: 'charcoal' | 'green' | 'blue'
}
```

La validation est répétée à la frontière du rendu PDF et de la finalisation.
Ainsi, une version de modèle inconnue ou une configuration incomplète est
refusée clairement avant d'entrer dans le moteur de rendu. Les protections ne
reposent pas uniquement sur le formulaire d'administration.

## 4. Choisir les données publiques de la mission

`src/domain/missionDocument.ts` construit une liste de textes à partir d'une
liste explicite de champs. Il ne copie pas l'objet `Mission` entier.

Informations reprises lorsqu'elles sont renseignées :

- type, intitulé et lieu de la mission ;
- date du diagnostic et volume estimé en mètres linéaires ;
- lieux de conservation ;
- durée estimée et heures de référence par journée ;
- conditions de démarrage, règle de trajet et condition de facturation ;
- livrable prévu pour le type de mission, si sa case est cochée.

Pour une mission de classement, le plan de classement peut apparaître, mais
pas le rapport de récolement simplement parce qu'une autre case vaut `true`.
Pour une mission mixte, les deux livrables pertinents peuvent être repris.
Ces libellés décrivent le périmètre prévu : ils n'attestent ni réalisation,
ni validation administrative d'une élimination.

Les notes internes, le statut de travail, les identifiants techniques et la
photo du diagnostic ne sont pas imprimés. Les dates sont formatées en UTC et
les nombres en français. Les valeurs numériques doivent être finies et
positives ou nulles. Le document accepte au maximum 100 lieux de conservation
et 2 500 caractères par rubrique de mission; il refuse un dépassement au lieu
de couper silencieusement des informations.

Les durées estimées ne sont pas des journées réellement effectuées. Le suivi
quotidien des missions et des dépenses reste une étape distincte.

## 5. Versionner le snapshot

Les nouveaux aperçus produits par `buildQuoteSnapshot` portent maintenant
`templateVersion: 2` et une propriété `design`. Les détails de mission sont
copiés sous forme de chaînes dans `mission.details`.

```ts
{
  templateVersion: 2,
  design: quoteDesign(company.quoteTemplate),
  // Identités, adresses, lignes et montants déjà figés aux chapitres précédents.
  mission: mission ? {
    title: mission.title,
    location: mission.locationLabel ?? '',
    details: missionDocumentDetails(mission),
  } : null,
}
```

Cette version désigne le format des données et le comportement du renderer,
pas un compteur des modifications de l'entreprise. Deux aperçus de version 2
peuvent contenir des introductions différentes : chacun garde son propre texte.

Les snapshots de version 1 restent acceptés. Ils n'ont pas besoin d'une
propriété `design` et utilisent l'ancien titre `Devis`, sans les nouvelles
rubriques. Le type `QuoteSnapshot` exprime cette compatibilité; le renderer et
`validateFinalSnapshot` acceptent uniquement les versions prises en charge.

**Moment d'application :** changer le modèle affecte le prochain aperçu, même
si son devis brouillon existait déjà. Cela ne modifie aucun aperçu enregistré.
Pour changer un document déjà approuvé, produire puis relire une nouvelle
version. La finalisation utilise toujours le snapshot de la version approuvée,
jamais les paramètres d'entreprise actuels.

## 6. Faire évoluer le PDF

Dans `src/pdf/QuoteDocument.tsx`, le modèle de version 2 affiche :

1. le titre personnalisé, dans la couleur choisie ;
2. l'introduction après les identités des deux parties ;
3. le cadre public de la mission avant les prestations ;
4. la conclusion après les conditions de paiement ;
5. le bloc d'accord et de signature sur un document final.

L'en-tête et le pied de page continuent d'indiquer clairement le devis et sa
référence. Un titre commercial ne supprime pas l'identification du document.
Les textes restent dans des composants `Text` du moteur React PDF : ils ne
sont pas interprétés comme du HTML. Les paragraphes longs sont découpés en
fragments et peuvent se répartir sur plusieurs pages.

Les montants, la numérotation, les empreintes PDF et les confirmations d'envoi
ne changent pas. Les PDF déjà enregistrés sont servis à partir de leurs octets
conservés : ils ne sont pas régénérés quand l'entreprise change son modèle.

Ce chapitre n'intègre pas encore le logo ou la photo de couverture, ne propose
pas de modèle DOCX à téléverser et ne reproduit pas automatiquement un PDF de
référence. La mise en page reste gérée dans React, les textes dans Payload.

## 7. Tester avec des documents fictifs

```powershell
pnpm test
pnpm exec tsc --noEmit
pnpm lint
pnpm build
```

Les nouveaux contrôles couvrent limites de texte, couleurs, dates, nombres,
livrables pertinents, absence de notes internes, copie indépendante des données,
compatibilité version 1 et migration additive exécutée deux fois.

Le test d'intégration modifie le modèle et le volume de mission après avoir
créé un premier aperçu. Il vérifie qu'un nouvel aperçu reprend les changements,
alors que le snapshot et les octets PDF du premier restent identiques.
Le test navigateur configure les champs dans l'admin, enregistre, produit
un aperçu puis vérifie sa conservation après une seconde modification.

Pour contrôler la mise en page sans données réelles ni écriture dans la base :

```powershell
pnpm exec cross-env NODE_OPTIONS=--no-deprecation tsx scripts/render-chapter13-examples.ts
```

Ce script utilise uniquement les fixtures fictives du projet. Il écrit deux
PDF dans `test-results`, ignoré par Git : un exemple court avec mission et un
exemple long pour vérifier la pagination. Relire les premières et dernières
pages, ainsi qu'une page intermédiaire, en contrôlant marges, montants,
signature et pagination. Les tests automatiques ne remplacent pas cette passe
visuelle lorsqu'on change une mise en page PDF.

Code des tests : [unitaires](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/unit/quoteDesign.unit.spec.ts),
[snapshot](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/unit/quoteSnapshot.unit.spec.ts),
[intégration](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/int/quoteTemplate.int.spec.ts),
[upgrade](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/int/quoteTemplateUpgrade.int.spec.ts),
[navigateur](https://github.com/Anakinyo/payload-archiviste/blob/main/tests/e2e/quoteTemplate.e2e.spec.ts).

## 8. Essayer le parcours sur le portable

Après sauvegarde, upgrade et compilation :

```powershell
pnpm start
```

Ouvrir `http://127.0.0.1:3000`, se connecter, puis ouvrir les paramètres de
l'entreprise. Compléter le modèle et enregistrer. Dans une mission, renseigner
le diagnostic et la planification; dans un devis de ce même client, sélectionner
cette mission, enregistrer et générer un aperçu. Relire les textes et le cadre
de mission avant de l'approuver.

Un nouveau champ de configuration n'est pas une raison de renvoyer les anciens
documents. Aucune émission d'e-mail automatique n'est ajoutée ici.

## Suite du parcours

Le prochain chapitre peut ajouter le journal des journées réalisées et un
suivi opérationnel des missions. Les ordres de mission distincts, photos,
logos et modèles plus avancés pourront ensuite réutiliser les mêmes principes
de données structurées, snapshot immuable, aperçu et validation humaine.
