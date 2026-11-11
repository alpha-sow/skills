# Skills pour agents IA

Une collection de guides pour aider les agents IA à organiser et réaliser des tâches de développement. Ce dépôt contient des instructions et des exemples pédagogiques, pas une application à exécuter.

## Skill disponible

| Skill | Objectif | Ressource complémentaire |
| --- | --- | --- |
| [spring-boot-project-organization](skills/spring-boot-project-organization/SKILL.md) | Organiser un projet Java Spring Boot : responsabilités Application/Domain/Infrastructure, DDD, Clean Architecture, architecture hexagonale, dépendances et tests ciblés. | [Exemple Java/Spring](skills/spring-boot-project-organization/exemples-spring-boot.md) : modèle métier, port, cas d’usage, contrôleur, configuration, requête HTTP et test JUnit. |

## Installation pour OpenCode

Installez le skill `spring-boot-project-organization` avec le CLI `skills` :

```sh
npx skills add https://github.com/alpha-sow/skills --skill spring-boot-project-organization
```

Pour mettre à jour le skill dans le projet (`-p` pour le scope projet) :

```sh
npx skills update spring-boot-project-organization -p
```

## Organisation d’un skill

```text
skills/
└── <skill-name>/
    ├── SKILL.md       # Instructions et métadonnées du skill
    └── code.md        # Ressource Markdown facultative
```

Chaque `SKILL.md` comporte un frontmatter YAML avec les champs obligatoires `name` et `description`. Les ressources Markdown complémentaires restent dans le même répertoire.

## Validation documentaire

Consultez [AGENTS.md](AGENTS.md) avant toute modification. Vérifiez le contenu, les liens et le diff, puis contrôlez les erreurs d’espacement avec :

```sh
git diff --check
```

Aucune commande de build, de test ou de lint n’est configurée. Le code Java fourni est illustratif et n’est pas destiné à être compilé ou testé dans ce dépôt.
