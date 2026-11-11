# Contribuer

Ce dépôt est un catalogue documentaire de skills pour agents IA, et non une application compilable.

## Ajouter ou modifier un skill

- Placer chaque skill dans `skills/<skill-name>/SKILL.md`.
- Commencer `SKILL.md` par un frontmatter YAML contenant au minimum `name` et `description` :

  ```yaml
  ---
  name: mon-skill
  description: Décrit le rôle du skill et les situations où l’utiliser.
  ---
  ```

- Rédiger les instructions après le frontmatter et conserver l’organisation existante lors des modifications.
- Placer les ressources Markdown facultatives, comme `code.md`, dans le même répertoire que le skill.

## Vérifier les changements

Relire les fichiers concernés : clarté des instructions, cohérence du frontmatter, chemins et exemples. Les exemples de code sont illustratifs ; ce dépôt ne définit aucune commande locale de build, de test ou de lint, ni de validation automatisée.

Inspecter le diff et vérifier les erreurs d’espacement :

```sh
git diff
git diff --check
```

Les fichiers non suivis ne figurent pas dans ce diff : relire directement leur contenu. Pour des changements déjà indexés, utiliser aussi `git diff --cached` et `git diff --cached --check`.

## Messages de commit et releases

Utiliser Conventional Commits, avec un type suivi de `:` et d’une description courte :

```text
feat: ajouter un skill de revue de code
fix: corriger les instructions d’un skill
docs: clarifier un exemple de contribution
```

Pour une modification incompatible, ajouter une note `BREAKING CHANGE:` dans le corps du commit :

```text
feat: réorganiser les ressources d’un skill

BREAKING CHANGE: les ressources du skill utilisent de nouveaux chemins.
```

Sur `main`, semantic-release détermine automatiquement les releases depuis les commits : `feat` déclenche une version mineure, `fix` une version corrective et `BREAKING CHANGE` une version majeure. Un commit `docs` seul ne déclenche pas de release.

Les releases sont publiées uniquement sur GitHub Releases. Le paquet est privé et n’est pas publié sur npm.
