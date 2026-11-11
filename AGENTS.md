# Instructions pour les agents

Ce dépôt est un catalogue documentaire de skills pour agents IA, sous licence MIT, et non une application compilable.

## Structure et contributions

- Chaque skill réside dans `skills/<skill-name>/SKILL.md`.
- `SKILL.md` porte les instructions du skill et un frontmatter YAML obligatoire contenant `name` et `description`.
- Les ressources Markdown facultatives, comme `code.md`, accompagnent le skill dans son répertoire. Conserver cette organisation lors des contributions.

## Validation

Aucune commande locale de build, de test ou de lint ni infrastructure de validation automatisée n’est définie. Ne pas en inventer.

Le code d’exemple, notamment Java/JUnit dans `skills/spring-boot-project-organization/exemples-spring-boot.md`, est illustratif : ne pas le traiter comme un projet à compiler ou à tester dans ce dépôt.

Valider les changements documentaires par une inspection ciblée des fichiers concernés et, si nécessaire, du diff Git. Utiliser `git diff --check` pour vérifier les erreurs d’espacement du diff.
